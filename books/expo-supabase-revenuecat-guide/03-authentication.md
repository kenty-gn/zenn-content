---
title: "第3章: 認証フロー — ログイン・サインアップの実装"
free: false
---

# 第3章: 認証フロー --- ログイン・サインアップの実装

この章では、Supabase Auth を使った認証フローを実装します。メール/パスワード認証に加えて、Google・Apple の OAuth ログインにも対応します。さらに、Zustand で認証状態を管理するストアを作り、Zod でフォームバリデーションを行うカスタムフックまで仕上げます。

## 3.1 Supabase Authの概要

Supabase Auth は、認証に必要な機能をフルマネージドで提供してくれるサービスです。ふくログでは以下の3つの認証方式を使います。

| 認証方式 | 用途 |
|---|---|
| メール/パスワード | 最も基本的な方式。メールアドレスの確認メールが自動送信される |
| Google OAuth | ワンタップでログイン。Android ユーザーに人気 |
| Apple Sign-In | iOS アプリでは App Store 審査でソーシャルログインを提供する場合 Apple Sign-In が必須 |

セッション管理は Supabase クライアントが自動で行います。第1章で設定した `ExpoSecureStoreAdapter` により、アクセストークンとリフレッシュトークンが iOS Keychain / Android Keystore に安全に保存されます。トークンの自動更新（`autoRefreshToken: true`）も有効にしているので、ユーザーが明示的にログアウトしない限りセッションは維持されます。

## 3.2 認証サービス層の実装（auth.ts）

認証に関する API 呼び出しを `services/supabase/auth.ts` にまとめます。画面コンポーネントから直接 Supabase クライアントを触らず、サービス層を経由することで、テストしやすく保守性の高い設計になります。

### メール認証

```typescript
import { supabase } from './client';
import * as WebBrowser from 'expo-web-browser';
import { DEEP_LINKS } from '@/src/constants/config';

WebBrowser.maybeCompleteAuthSession();

/** メール/パスワードでサインアップ */
export async function signUpWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signUp({ email, password });
  if (error) throw error;
  return data;
}

/** メール/パスワードでサインイン */
export async function signInWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) throw error;
  return data;
}
```

`signUp` はアカウント作成、`signInWithPassword` は既存アカウントへのログインです。Supabase はデフォルトでメール確認を要求しますが、開発中は **Authentication > Providers > Email** で「Confirm email」をオフにすると、確認メールなしでテストできます。

### OAuth（Google / Apple）

Expo のモバイルアプリで OAuth を実装する場合、`expo-web-browser` を使ってシステムブラウザで認証画面を開き、ディープリンクでアプリに戻す流れになります。

```typescript
/** Googleでサインイン */
export async function signInWithGoogle() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: DEEP_LINKS.authCallback,
      skipBrowserRedirect: true,
    },
  });
  if (error) throw error;

  if (data.url) {
    const result = await WebBrowser.openAuthSessionAsync(
      data.url,
      DEEP_LINKS.authCallback
    );
    if (result.type === 'success') {
      await handleAuthCallback(result.url);
    }
  }
}
```

ポイントは `skipBrowserRedirect: true` です。これを指定すると、Supabase は認証URLを返すだけで自動リダイレクトしません。返ってきたURLを `WebBrowser.openAuthSessionAsync` に渡すことで、Expo が管理するブラウザセッションで認証が行われます。

### OAuthコールバック処理

認証完了後、Supabase はアクセストークンをURLフラグメントに含めてリダイレクトします。このトークンを取り出してセッションにセットします。

```typescript
async function handleAuthCallback(url: string) {
  const params = new URL(url).hash.substring(1);
  const urlParams = new URLSearchParams(params);
  const accessToken = urlParams.get('access_token');
  const refreshToken = urlParams.get('refresh_token');

  if (accessToken && refreshToken) {
    await supabase.auth.setSession({
      access_token: accessToken,
      refresh_token: refreshToken,
    });
  }
}
```

URLフラグメント（`#` 以降）にトークンが含まれる理由は、フラグメントがサーバーに送信されないためです。URLのクエリパラメータ（`?` 以降）だと、中間サーバーのログに残るリスクがあります。

## 3.3 Zustandで認証ストアを作る

### なぜZustandを選んだか

React Native での状態管理には Redux、Context API、Zustand などの選択肢があります。ふくログでは **Zustand** を採用しました。

| ライブラリ | ボイラープレート | 学習コスト | Bundle サイズ |
|---|---|---|---|
| Redux Toolkit | 多い | 高い | 11KB |
| Context API | 少ない | 低い | 0KB（組み込み） |
| Zustand | **最小** | **低い** | **1.2KB** |

Zustand の最大のメリットは、Provider で囲む必要がなく、**コンポーネントツリーの外側からでもストアにアクセスできる** 点です。これは後の章で Edge Function からの Webhook でプロフィールを更新する場面で活きてきます。

### ストアの実装

```typescript
import { create } from 'zustand';

export const useAuthStore = create<AuthStore>((set, get) => ({
  user: null,
  session: null,
  profile: null,
  isLoading: true,
  isAuthenticated: false,

  initialize: async () => {
    try {
      set({ isLoading: true });
      const session = await getSession();

      if (session?.user) {
        const profile = await getProfile(session.user.id);
        set({
          user: session.user,
          session,
          profile,
          isAuthenticated: true,
          isLoading: false,
        });
      } else {
        set({ user: null, session: null, profile: null,
              isAuthenticated: false, isLoading: false });
      }

      // 認証状態の変更をリアルタイムに監視
      authSubscription?.unsubscribe();
      const { data } = supabase.auth.onAuthStateChange(async (event, session) => {
        if (event === 'SIGNED_IN' && session?.user) {
          // 重複イベントを防ぐ
          const currentState = get();
          if (currentState.isAuthenticated
              && currentState.user?.id === session.user.id) {
            return;
          }
          set({ user: session.user, session,
                isAuthenticated: true, isLoading: false });
          // プロフィールを非同期で取得
          const profile = await getProfile(session.user.id);
          if (profile) set({ profile });
        } else if (event === 'SIGNED_OUT') {
          set({ user: null, session: null, profile: null,
                isAuthenticated: false, isLoading: false });
        } else if (event === 'TOKEN_REFRESHED' && session) {
          set({ session });
        }
      });
      authSubscription = data.subscription;
    } catch {
      set({ user: null, session: null, profile: null,
            isAuthenticated: false, isLoading: false });
    }
  },
  // ... signInWithEmail, signUpWithEmail 等
}));
```

`initialize` 関数は、アプリ起動時にルートレイアウト（`_layout.tsx`）から呼び出します。まず既存セッションを復元し、次に `onAuthStateChange` リスナーを登録します。これにより、OAuth の認証完了時やトークンリフレッシュ時にもストアが自動的に更新されます。

### サインアップ時のプロフィール取得

```typescript
signUpWithEmail: async (email: string, password: string) => {
  set({ isLoading: true });
  const { user, session } = await apiSignUpWithEmail(email, password);
  if (user && session) {
    // トリガーによるプロフィール作成を待つ
    let profile = null;
    for (let i = 0; i < 5; i++) {
      await new Promise(resolve => setTimeout(resolve, 300));
      profile = await getProfile(user.id);
      if (profile) break;
    }
    set({ user, session, profile, isAuthenticated: true, isLoading: false });
  }
},
```

ここでリトライループを使っている理由は、第2章で設定した `handle_new_user` トリガーが非同期で実行されるためです。`signUp` が完了した直後にはプロフィールがまだ作成されていないことがあるため、最大5回（1.5秒間）リトライして待ちます。

### セレクター

コンポーネントから必要な値だけを購読するセレクターを定義します。

```typescript
export const useIsAuthenticated = () => useAuthStore(state => state.isAuthenticated);
export const useIsLoading = () => useAuthStore(state => state.isLoading);
export const useUser = () => useAuthStore(state => state.user);
export const useProfile = () => useAuthStore(state => state.profile);
```

Zustand ではセレクターを使うことで、不要な再レンダリングを防げます。たとえば `useIsAuthenticated()` を使うと、`isAuthenticated` が変わったときだけコンポーネントが再レンダリングされ、`profile` の変更では再レンダリングされません。

## 3.4 useAuthフック

コンポーネントから認証操作を呼び出す際のインターフェースとして、`useAuth` カスタムフックを作ります。このフックの役割は **バリデーション**、**エラーハンドリング**、**送信状態の管理** です。

```typescript
import { z } from 'zod';

const emailSchema = z.string().email('有効なメールアドレスを入力してください');
const passwordSchema = z
  .string()
  .min(8, 'パスワードは8文字以上で入力してください')
  .regex(/[A-Za-z]/, 'パスワードには英字を含めてください')
  .regex(/[0-9]/, 'パスワードには数字を含めてください');
```

Zod スキーマを使うことで、バリデーションルールとエラーメッセージを宣言的に定義できます。サインアップ時にはメールとパスワードの両方を検証し、ログイン時にはメール形式のみ検証します（パスワードのバリデーションはサーバーに任せる）。

### エラーメッセージの日本語化

Supabase Auth が返す英語のエラーメッセージを、日本語に変換するヘルパーを用意します。

```typescript
function translateAuthError(message: string): string {
  const errorMap: Record<string, string> = {
    'Invalid login credentials': 'メールアドレスまたはパスワードが正しくありません',
    'Email not confirmed': 'メールアドレスの確認が完了していません',
    'User already registered': 'このメールアドレスは既に登録されています',
    'Email rate limit exceeded': 'しばらく時間をおいてから再度お試しください',
    'For security purposes, you can only request this once every 60 seconds':
      'セキュリティのため、60秒後に再度お試しください',
  };
  return errorMap[message] || message;
}
```

日本語のアプリで英語のエラーメッセージが表示されると、ユーザー体験が大きく損なわれます。よくあるエラーパターンを網羅的にマッピングしておくことで、ユーザーが迷わず次のアクションを取れるようになります。

## 3.5 ルート保護（Protected Routes）

認証状態に応じて画面を切り替えるロジックは、Expo Router のルートレイアウト（`app/_layout.tsx`）に集約します。

```typescript
// app/_layout.tsx のイメージ
export default function RootLayout() {
  const { isAuthenticated, isLoading } = useAuthStatus();

  useEffect(() => {
    useAuthStore.getState().initialize();
  }, []);

  if (isLoading) return <LoadingScreen />;

  return (
    <Stack screenOptions={{ headerShown: false }}>
      {isAuthenticated ? (
        <Stack.Screen name="(tabs)" />
      ) : (
        <Stack.Screen name="(auth)" />
      )}
    </Stack>
  );
}
```

`isAuthenticated` が `false` の場合は認証画面グループ（`(auth)/`）を、`true` の場合はタブ画面グループ（`(tabs)/`）を表示します。この分岐をルートレイアウトに置くことで、認証チェックが確実に行われ、未認証ユーザーがメイン画面にアクセスすることを防げます。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- メール/パスワードでサインアップ・ログインができる
- Google / Apple でソーシャルログインができる
- Zustand ストアで認証状態がリアクティブに管理されている
- Zod でフォーム入力がバリデーションされ、エラーは日本語で表示される
- 認証状態に応じて画面が自動的に切り替わる

次の章では、認証されたユーザーが実際にデータを入力する、メイン機能（収入・経費の記録）の実装に進みます。

## ソースコード

### `stores/authStore.ts`

```typescript
import { create } from 'zustand';
import { AuthStore, UpdateProfileData } from '@/src/types/auth';
import {
  supabase,
  signInWithEmail as apiSignInWithEmail,
  signUpWithEmail as apiSignUpWithEmail,
  signInWithGoogle as apiSignInWithGoogle,
  signInWithApple as apiSignInWithApple,
  signOut as apiSignOut,
  resetPassword as apiResetPassword,
  getProfile,
  getSession,
} from '@/src/services/supabase';

let authSubscription: { unsubscribe: () => void } | null = null;

export const useAuthStore = create<AuthStore>((set, get) => ({
  user: null,
  session: null,
  profile: null,
  isLoading: true,
  isAuthenticated: false,

  initialize: async () => {
    try {
      set({ isLoading: true });
      const session = await getSession();

      if (session?.user) {
        const profile = await getProfile(session.user.id);
        set({
          user: session.user,
          session,
          profile,
          isAuthenticated: true,
          isLoading: false,
        });
      } else {
        set({
          user: null,
          session: null,
          profile: null,
          isAuthenticated: false,
          isLoading: false,
        });
      }

      // 既存リスナーのクリーンアップ
      authSubscription?.unsubscribe();
      const { data } = supabase.auth.onAuthStateChange(async (event, session) => {
        if (event === 'SIGNED_IN' && session?.user) {
          const currentState = get();
          if (currentState.isAuthenticated && currentState.user?.id === session.user.id) {
            return;
          }

          set({
            user: session.user,
            session,
            isAuthenticated: true,
            isLoading: false,
          });

          try {
            const profile = await getProfile(session.user.id);
            if (profile) {
              set({ profile });
            }
          } catch {
            // プロフィール取得失敗は無視
          }
        } else if (event === 'SIGNED_OUT') {
          set({
            user: null,
            session: null,
            profile: null,
            isAuthenticated: false,
            isLoading: false,
          });
        } else if (event === 'TOKEN_REFRESHED' && session) {
          set({ session });
        }
      });
      authSubscription = data.subscription;
    } catch {
      set({
        user: null,
        session: null,
        profile: null,
        isAuthenticated: false,
        isLoading: false,
      });
    }
  },

  signInWithEmail: async (email: string, password: string) => {
    try {
      set({ isLoading: true });
      const { user, session } = await apiSignInWithEmail(email, password);
      if (user) {
        const profile = await getProfile(user.id);
        set({ user, session, profile, isAuthenticated: true, isLoading: false });
      }
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  signUpWithEmail: async (email: string, password: string) => {
    try {
      set({ isLoading: true });
      const { user, session } = await apiSignUpWithEmail(email, password);
      if (user && session) {
        let profile = null;
        for (let i = 0; i < 5; i++) {
          await new Promise(resolve => setTimeout(resolve, 300));
          profile = await getProfile(user.id);
          if (profile) break;
        }
        set({ user, session, profile, isAuthenticated: true, isLoading: false });
      } else {
        set({ isLoading: false });
      }
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  signInWithGoogle: async () => {
    try {
      set({ isLoading: true });
      await apiSignInWithGoogle();
    } catch (error) {
      set({ isLoading: false });
      throw error;
    } finally {
      setTimeout(() => {
        const state = get();
        if (state.isLoading && !state.isAuthenticated) {
          set({ isLoading: false });
        }
      }, 1000);
    }
  },

  signInWithApple: async () => {
    try {
      set({ isLoading: true });
      await apiSignInWithApple();
    } catch (error) {
      set({ isLoading: false });
      throw error;
    } finally {
      setTimeout(() => {
        const state = get();
        if (state.isLoading && !state.isAuthenticated) {
          set({ isLoading: false });
        }
      }, 1000);
    }
  },

  signOut: async () => {
    try {
      set({ isLoading: true });
      await apiSignOut();
      set({
        user: null,
        session: null,
        profile: null,
        isAuthenticated: false,
        isLoading: false,
      });
    } catch (error) {
      set({ isLoading: false });
      throw error;
    }
  },

  resetPassword: async (email: string) => {
    await apiResetPassword(email);
  },

  refreshProfile: async () => {
    const { user } = get();
    if (user) {
      const profile = await getProfile(user.id);
      set({ profile });
    }
  },

  updateProfile: async (data: UpdateProfileData) => {
    const { user } = get();
    if (!user) throw new Error('認証されていません');

    const { error } = await supabase
      .from('profiles')
      .update(data)
      .eq('id', user.id);

    if (error) throw error;

    const profile = await getProfile(user.id);
    set({ profile });
  },

}));

export const useIsAuthenticated = () => useAuthStore(state => state.isAuthenticated);
export const useIsLoading = () => useAuthStore(state => state.isLoading);
export const useUser = () => useAuthStore(state => state.user);
export const useProfile = () => useAuthStore(state => state.profile);
```

### `hooks/useAuth.ts`

```typescript
import { useCallback, useState } from 'react';
import { useAuthStore } from '@/src/stores/authStore';
import { z } from 'zod';

const emailSchema = z.string().email('有効なメールアドレスを入力してください');

const passwordSchema = z
  .string()
  .min(8, 'パスワードは8文字以上で入力してください')
  .regex(/[A-Za-z]/, 'パスワードには英字を含めてください')
  .regex(/[0-9]/, 'パスワードには数字を含めてください');

export function useAuth() {
  const store = useAuthStore();
  const [error, setError] = useState<string | null>(null);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const signInWithEmail = useCallback(
    async (email: string, password: string) => {
      setError(null);
      setIsSubmitting(true);
      try {
        emailSchema.parse(email);
        await store.signInWithEmail(email, password);
      } catch (err) {
        if (err instanceof z.ZodError) {
          setError(err.issues[0].message);
        } else if (err instanceof Error) {
          setError(translateAuthError(err.message));
        } else {
          setError('ログインに失敗しました');
        }
        throw err;
      } finally {
        setIsSubmitting(false);
      }
    },
    [store]
  );

  const signUpWithEmail = useCallback(
    async (email: string, password: string) => {
      setError(null);
      setIsSubmitting(true);
      try {
        emailSchema.parse(email);
        passwordSchema.parse(password);
        await store.signUpWithEmail(email, password);
      } catch (err) {
        if (err instanceof z.ZodError) {
          setError(err.issues[0].message);
        } else if (err instanceof Error) {
          setError(translateAuthError(err.message));
        } else {
          setError('登録に失敗しました');
        }
        throw err;
      } finally {
        setIsSubmitting(false);
      }
    },
    [store]
  );

  const signInWithGoogle = useCallback(async () => {
    setError(null);
    setIsSubmitting(true);
    try {
      await store.signInWithGoogle();
    } catch (err) {
      if (err instanceof Error) {
        setError(translateAuthError(err.message));
      } else {
        setError('Googleログインに失敗しました');
      }
      throw err;
    } finally {
      setIsSubmitting(false);
    }
  }, [store]);

  const signInWithApple = useCallback(async () => {
    setError(null);
    setIsSubmitting(true);
    try {
      await store.signInWithApple();
    } catch (err) {
      if (err instanceof Error) {
        setError(translateAuthError(err.message));
      } else {
        setError('Appleログインに失敗しました');
      }
      throw err;
    } finally {
      setIsSubmitting(false);
    }
  }, [store]);

  const signOut = useCallback(async () => {
    setError(null);
    try {
      await store.signOut();
    } catch (err) {
      if (err instanceof Error) setError(err.message);
      throw err;
    }
  }, [store]);

  const resetPassword = useCallback(
    async (email: string) => {
      setError(null);
      setIsSubmitting(true);
      try {
        emailSchema.parse(email);
        await store.resetPassword(email);
      } catch (err) {
        if (err instanceof z.ZodError) {
          setError(err.issues[0].message);
        } else if (err instanceof Error) {
          setError(translateAuthError(err.message));
        } else {
          setError('パスワードリセットに失敗しました');
        }
        throw err;
      } finally {
        setIsSubmitting(false);
      }
    },
    [store]
  );

  const clearError = useCallback(() => setError(null), []);

  return {
    user: store.user,
    profile: store.profile,
    isAuthenticated: store.isAuthenticated,
    isLoading: store.isLoading,
    isSubmitting,
    error,
    signInWithEmail,
    signUpWithEmail,
    signInWithGoogle,
    signInWithApple,
    signOut,
    resetPassword,
    refreshProfile: store.refreshProfile,
    clearError,
  };
}

function translateAuthError(message: string): string {
  const errorMap: Record<string, string> = {
    'Invalid login credentials': 'メールアドレスまたはパスワードが正しくありません',
    'Email not confirmed': 'メールアドレスの確認が完了していません',
    'User already registered': 'このメールアドレスは既に登録されています',
    'Password should be at least 6 characters': 'パスワードは6文字以上で入力してください',
    'Unable to validate email address: invalid format': '有効なメールアドレスを入力してください',
    'Email rate limit exceeded': 'しばらく時間をおいてから再度お試しください',
    'For security purposes, you can only request this once every 60 seconds':
      'セキュリティのため、60秒後に再度お試しください',
  };
  return errorMap[message] || message;
}

export function useAuthStatus() {
  const isAuthenticated = useAuthStore(state => state.isAuthenticated);
  const isLoading = useAuthStore(state => state.isLoading);
  return { isAuthenticated, isLoading };
}
```

### `services/supabase/auth.ts`

```typescript
import { supabase } from './client';
import { Profile } from '@/src/types/auth';
import * as WebBrowser from 'expo-web-browser';
import { DEEP_LINKS } from '@/src/constants/config';

WebBrowser.maybeCompleteAuthSession();

/**
 * メール/パスワードでサインアップ
 */
export async function signUpWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signUp({
    email,
    password,
  });
  if (error) throw error;
  return data;
}

/**
 * メール/パスワードでサインイン
 */
export async function signInWithEmail(email: string, password: string) {
  const { data, error } = await supabase.auth.signInWithPassword({
    email,
    password,
  });
  if (error) throw error;
  return data;
}

/**
 * Googleでサインイン
 */
export async function signInWithGoogle() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: DEEP_LINKS.authCallback,
      skipBrowserRedirect: true,
    },
  });
  if (error) throw error;

  if (data.url) {
    const result = await WebBrowser.openAuthSessionAsync(
      data.url,
      DEEP_LINKS.authCallback
    );
    if (result.type === 'success') {
      await handleAuthCallback(result.url);
    }
  }
}

/**
 * Appleでサインイン
 */
export async function signInWithApple() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'apple',
    options: {
      redirectTo: DEEP_LINKS.authCallback,
      skipBrowserRedirect: true,
    },
  });
  if (error) throw error;

  if (data.url) {
    const result = await WebBrowser.openAuthSessionAsync(
      data.url,
      DEEP_LINKS.authCallback
    );
    if (result.type === 'success') {
      await handleAuthCallback(result.url);
    }
  }
}

/**
 * OAuthコールバック処理
 */
async function handleAuthCallback(url: string) {
  const params = new URL(url).hash.substring(1);
  const urlParams = new URLSearchParams(params);
  const accessToken = urlParams.get('access_token');
  const refreshToken = urlParams.get('refresh_token');

  if (accessToken && refreshToken) {
    await supabase.auth.setSession({
      access_token: accessToken,
      refresh_token: refreshToken,
    });
  }
}

/**
 * パスワードリセット
 */
export async function resetPassword(email: string) {
  const { error } = await supabase.auth.resetPasswordForEmail(email, {
    redirectTo: DEEP_LINKS.resetPassword,
  });
  if (error) throw error;
}

/**
 * サインアウト
 */
export async function signOut() {
  const { error } = await supabase.auth.signOut();
  if (error) throw error;
}

/**
 * 現在のセッションを取得
 */
export async function getSession() {
  const { data, error } = await supabase.auth.getSession();
  if (error) throw error;
  return data.session;
}

/**
 * プロフィールを取得
 */
export async function getProfile(userId: string): Promise<Profile | null> {
  try {
    const { data, error } = await supabase
      .from('profiles')
      .select('*')
      .eq('id', userId)
      .single();

    if (error) return null;
    return data as Profile;
  } catch {
    return null;
  }
}
```
