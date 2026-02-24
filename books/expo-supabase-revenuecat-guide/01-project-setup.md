---
title: "第1章: プロジェクトセットアップ"
free: true
---

# 第1章: プロジェクトセットアップ

この章では、Expo + Supabase + RevenueCat で構成する「ふくログ」アプリの土台を作ります。プロジェクトの作成からディレクトリ設計、必要なパッケージの導入、TypeScript・環境変数の設定まで、一気に進めていきます。

## 1.1 Expoプロジェクトの作成

まずは Expo のプロジェクトを作成します。

```bash
npx create-expo-app@latest side-income-tracker
cd side-income-tracker
```

テンプレートは **デフォルト（blank）** を選びます。`tabs` テンプレートはナビゲーションの雛形が付いてきますが、Expo Router v6 では `app/` ディレクトリのファイルベースルーティングを使うため、テンプレートのボイラープレートはかえって邪魔になります。blank から始めて必要なものだけ足していくのがおすすめです。

### ディレクトリ構成

ふくログでは以下のディレクトリ構成を採用しています。

```
side-income-tracker/
├── app/                  # Expo Router のルーティング
│   ├── (auth)/           # 認証系画面
│   ├── (tabs)/           # タブナビゲーション
│   └── _layout.tsx       # ルートレイアウト
├── src/
│   ├── services/         # API層（Supabase, RevenueCat）
│   ├── stores/           # Zustand ストア
│   ├── hooks/            # カスタムフック
│   ├── components/       # UIコンポーネント
│   ├── constants/        # 定数・設定
│   ├── types/            # 型定義
│   └── utils/            # ユーティリティ
├── app.json
├── tsconfig.json
└── package.json
```

ポイントは **`app/` と `src/` の分離** です。`app/` はルーティング定義に専念させ、ビジネスロジック・UI・状態管理は `src/` 以下に置きます。こうすることで、画面コンポーネントが肥大化するのを防ぎ、テストやリファクタリングがしやすくなります。

## 1.2 必要なパッケージのインストール

ふくログで使用する主要パッケージを一括インストールします。

```bash
# Supabase
npx expo install @supabase/supabase-js expo-secure-store

# 状態管理・バリデーション
npx expo install zustand zod

# UI・アニメーション
npx expo install expo-linear-gradient expo-haptics react-native-reanimated
```

各パッケージの役割を整理しておきます。

| パッケージ | 役割 |
|---|---|
| `@supabase/supabase-js` | Supabase の JavaScript クライアント。DB操作・認証・リアルタイム通信に使用 |
| `expo-secure-store` | iOS Keychain / Android Keystore にセッショントークンを安全に保存 |
| `zustand` | 軽量な状態管理ライブラリ。Redux より圧倒的にボイラープレートが少ない |
| `zod` | ランタイムの型バリデーション。フォーム入力やAPIレスポンスの検証に使用 |
| `expo-linear-gradient` | グラデーション背景の描画。ダッシュボードのUIで使用 |
| `expo-haptics` | 触覚フィードバック。記録追加時の操作感向上に使用 |
| `react-native-reanimated` | 高パフォーマンスなアニメーション。プログレスバー等で使用 |

:::message
`npx expo install` を使うと、現在の Expo SDK バージョンに互換性のあるバージョンが自動選択されます。`npm install` で入れるとバージョン不整合でビルドが通らないことがあるので、Expo のパッケージは必ず `npx expo install` で導入してください。
:::

## 1.3 TypeScript設定

`tsconfig.json` でパスエイリアスと厳格モードを設定します。

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["**/*.ts", "**/*.tsx", ".expo/types/**/*.ts", "expo-env.d.ts"]
}
```

`"strict": true` を有効にすることで、`noImplicitAny` や `strictNullChecks` などが一括でオンになります。型安全性を高めておくことで、Supabase のレスポンス型との整合性チェックが効くようになり、ランタイムエラーを大幅に減らせます。

パスエイリアス `@/*` を設定すると、深いディレクトリからのインポートが簡潔になります。

```typescript
// Before: 相対パスが深くなりがち
import { supabase } from '../../../services/supabase/client';

// After: パスエイリアスですっきり
import { supabase } from '@/src/services/supabase/client';
```

## 1.4 app.json の設定

`app.json` はアプリ全体の設定ファイルです。ふくログの設定から、特に重要なポイントを解説します。

```json
{
  "expo": {
    "name": "ふくログ",
    "slug": "side-income-tracker",
    "scheme": "sideincometracker",
    "userInterfaceStyle": "dark",
    "newArchEnabled": true,
    "ios": {
      "supportsTablet": true,
      "bundleIdentifier": "com.fukulog.app"
    },
    "android": {
      "package": "com.fukulog.app"
    },
    "plugins": [
      "expo-router",
      "expo-secure-store",
      "expo-web-browser"
    ],
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

**scheme** はディープリンクのURLスキームです。OAuth認証のコールバックで `sideincometracker://auth/callback` のようなURLを受け取るために必要です。第3章の認証実装で使います。

**bundleIdentifier / package** は App Store / Google Play に登録するアプリの一意識別子です。リリース後は変更できないので、最初にしっかり決めておきましょう。

**plugins** には、ネイティブモジュールの設定を自動生成する Expo Config Plugin を記述します。`expo-secure-store` を指定しておくと、iOS の Keychain アクセスに必要なエンタイトルメントが自動で追加されます。

**typedRoutes** を有効にすると、`router.push('/records/new')` のようなナビゲーションに対して TypeScript の型チェックが効くようになります。存在しないルートへの遷移をコンパイル時に検出できるので、ぜひ有効にしておきましょう。

## 1.5 環境変数の設定

Supabase の接続情報や RevenueCat の API キーは環境変数で管理します。プロジェクトルートに `.env` ファイルを作成します。

```bash
# .env
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key-here
EXPO_PUBLIC_REVENUECAT_APPLE_API_KEY=your-apple-api-key-here
EXPO_PUBLIC_REVENUECAT_GOOGLE_API_KEY=your-google-api-key-here
```

Expo では `EXPO_PUBLIC_` プレフィックスを付けた環境変数だけがクライアントサイドのバンドルに含まれます。プレフィックスなしの環境変数はビルド時に `undefined` になるので注意してください。

:::message alert
`EXPO_PUBLIC_` 付きの変数はバンドルに含まれるため、**エンドユーザーから見える** ことを前提にしてください。Supabase の `anon` キーは RLS（行レベルセキュリティ）で保護されているため公開しても問題ありませんが、`service_role` キーは絶対にここに書かないでください。
:::

`.gitignore` に `.env` を追加して、シークレットがリポジトリに入らないようにします。

```gitignore
# local env files
.env
.env*.local
```

チームメンバーや読者がすぐに環境構築できるよう、`.env.example` をコミットしておくとよいでしょう。

アプリ内では `constants/config.ts` で環境変数を集約し、型安全に参照できるようにしています。具体的な実装は末尾のソースコードを参照してください。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- Expo プロジェクトが作成され、`app/` と `src/` で責務が分離されたディレクトリ構成ができている
- Supabase クライアント、Zustand、Zod などの必要パッケージが導入されている
- TypeScript の厳格モードとパスエイリアスが設定されている
- `app.json` でディープリンクスキームやプラグインが設定されている
- 環境変数が `.env` で管理され、`config.ts` から型安全に参照できる

ターミナルで `npx expo start` を実行し、シミュレータまたは実機でアプリが起動することを確認してみてください。

次の章では、この土台の上に Supabase のデータベース設計とテーブル定義を進めていきます。

## ソースコード

### `constants/config.ts`

```typescript
/**
 * アプリ設定
 */
export const APP_CONFIG = {
  name: 'ふくログ',
  version: '1.0.0',
  scheme: 'sideincometracker',

  // 月間目標（デフォルト）
  defaultMonthlyGoal: 200000, // 20万円
} as const;

/**
 * 環境変数
 */
export const ENV = {
  supabaseUrl: process.env.EXPO_PUBLIC_SUPABASE_URL || '',
  supabaseAnonKey: process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY || '',
} as const;

/**
 * RevenueCat設定
 */
export const REVENUE_CAT = {
  appleApiKey: process.env.EXPO_PUBLIC_REVENUECAT_APPLE_API_KEY || '',
  googleApiKey: process.env.EXPO_PUBLIC_REVENUECAT_GOOGLE_API_KEY || '',
  entitlementId: 'pro',
  monthlyProductId: 'pro_monthly',
  annualProductId: 'pro_annual',
  monthlyPrice: '¥490',
  annualPrice: '¥4,900',
} as const;

/**
 * ディープリンク設定
 */
export const DEEP_LINKS = {
  authCallback: `${APP_CONFIG.scheme}://auth/callback`,
  resetPassword: `${APP_CONFIG.scheme}://auth/reset-password`,
} as const;
```

### `services/supabase/client.ts`

```typescript
import { createClient } from '@supabase/supabase-js';
import * as SecureStore from 'expo-secure-store';
import { Platform } from 'react-native';
import { ENV } from '@/src/constants/config';

/**
 * Secure Storage アダプター（iOS/Android用）
 */
const ExpoSecureStoreAdapter = {
  getItem: async (key: string): Promise<string | null> => {
    if (Platform.OS === 'web') {
      if (typeof window !== 'undefined' && window.localStorage) {
        return localStorage.getItem(key);
      }
      return null;
    }
    return await SecureStore.getItemAsync(key);
  },
  setItem: async (key: string, value: string): Promise<void> => {
    if (Platform.OS === 'web') {
      if (typeof window !== 'undefined' && window.localStorage) {
        localStorage.setItem(key, value);
      }
      return;
    }
    await SecureStore.setItemAsync(key, value);
  },
  removeItem: async (key: string): Promise<void> => {
    if (Platform.OS === 'web') {
      if (typeof window !== 'undefined' && window.localStorage) {
        localStorage.removeItem(key);
      }
      return;
    }
    await SecureStore.deleteItemAsync(key);
  },
};

/**
 * Supabaseクライアント
 */
export const supabase = createClient(
  ENV.supabaseUrl,
  ENV.supabaseAnonKey,
  {
    auth: {
      storage: ExpoSecureStoreAdapter,
      autoRefreshToken: true,
      persistSession: true,
      detectSessionInUrl: false,
    },
  }
);
```
