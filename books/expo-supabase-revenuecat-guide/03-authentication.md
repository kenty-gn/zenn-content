---
title: "第3章: 認証フロー — ログイン・サインアップの実装"
free: false
---

# 第3章: 認証フロー — ログイン・サインアップの実装

<!-- TODO: 執筆予定 Week 3 -->
<!-- アウトライン:
  3.1 Supabase Authの概要
  3.2 認証サービス層の実装（auth.ts）
  3.3 Zustandで認証ストアを作る（authStore.ts）
  3.4 useAuthフック
  3.5 認証画面のUI実装
  3.6 ルート保護（Protected Routes）
-->


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
