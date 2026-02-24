---
title: "第5章: Supabase Edge Functions — サーバーサイドロジック"
free: false
---

# 第5章: Supabase Edge Functions --- サーバーサイドロジック

ここまで、データの操作はすべてクライアント（Expo アプリ）から直接 Supabase に対して行ってきました。しかし、クライアント側だけでは対応しにくいケースがあります。この章では、Supabase Edge Functions を使ったサーバーサイドロジックの実装方法を解説します。

## 5.1 Edge Functionsとは

Supabase Edge Functions は、Deno ランタイム上で動作するサーバーレス関数です。AWS Lambda や Cloudflare Workers に近い存在ですが、Supabase プロジェクトに統合されているため、データベースへのアクセスが簡単にできるのが大きな利点です。

### いつEdge Functionsを使うべきか

| ユースケース | なぜクライアントだけでは不十分か |
|---|---|
| Webhook の受信 | 外部サービスからのHTTPリクエストを受け取る必要がある |
| 秘密鍵を使う処理 | `service_role` キーなどをクライアントに含められない |
| 複雑な集計処理 | クライアントで大量データを処理するとメモリ・通信量が膨大になる |
| 外部API連携 | APIキーを安全に管理しつつサーバー間通信を行いたい |

ふくログでは、第7章で実装する **RevenueCat Webhook** の受信に Edge Functions を使います。この章ではまず Edge Functions の基本的な作り方とデプロイ方法を押さえておきましょう。

## 5.2 ローカル開発環境のセットアップ

Edge Functions をローカルで開発・テストするには、Supabase CLI が必要です。

```bash
# Supabase CLI のインストール
brew install supabase/tap/supabase

# プロジェクトの初期化（初回のみ）
supabase init

# ローカル開発サーバーの起動
supabase functions serve
```

`supabase init` を実行すると、プロジェクトルートに `supabase/` ディレクトリが作成されます。Edge Functions は `supabase/functions/` ディレクトリ内に、関数名のフォルダを作って `index.ts` を配置します。

```
supabase/
└── functions/
    └── revenuecat-webhook/
        └── index.ts
```

## 5.3 Edge Functionの基本構造

Edge Functions は Deno の `Deno.serve` を使って HTTP リクエストを処理します。基本的な構造を見てみましょう。

```typescript
// supabase/functions/hello/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

Deno.serve(async (req) => {
  // メソッドチェック
  if (req.method !== 'POST') {
    return new Response('Method Not Allowed', { status: 405 })
  }

  // リクエストボディの取得
  const { name } = await req.json()

  // Supabase クライアントの作成（service_role キーを使用）
  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  )

  // データベース操作
  const { data, error } = await supabase
    .from('profiles')
    .select('*')
    .limit(10)

  // レスポンス
  return new Response(
    JSON.stringify({ message: `Hello ${name}`, data }),
    { status: 200, headers: { 'Content-Type': 'application/json' } }
  )
})
```

注目すべき点は、Edge Functions 内では **`service_role` キーを使って Supabase クライアントを作成する**ことです。`service_role` キーは RLS をバイパスできる管理者権限を持つため、クライアントには絶対に含めてはいけませんが、Edge Functions は安全なサーバー環境なので使用できます。

環境変数 `SUPABASE_URL` と `SUPABASE_SERVICE_ROLE_KEY` は、デプロイ先の Supabase プロジェクトでは自動的に設定されます。追加のシークレットは `supabase secrets set` コマンドで設定します。

## 5.4 アプリからEdge Functionsを呼び出す

アプリ側からは `supabase.functions.invoke()` で Edge Functions を呼び出せます。

```typescript
// アプリ側での呼び出し例
async function callEdgeFunction() {
  const { data, error } = await supabase.functions.invoke('hello', {
    body: { name: 'ふくログ' },
  });

  if (error) {
    console.error('Edge Function error:', error.message);
    return;
  }

  console.log('Response:', data);
}
```

`supabase.functions.invoke()` は、現在のユーザーセッションのアクセストークンを自動的に `Authorization` ヘッダーに付与します。Edge Function 側でユーザーを特定したい場合は、このトークンをデコードするか、Supabase クライアントで認証チェックを行います。

### エラーハンドリング

Edge Functions の呼び出しでは、ネットワークエラーや関数内エラーの両方に対応する必要があります。

```typescript
async function invokeWithRetry(
  functionName: string,
  body: Record<string, unknown>,
  maxRetries = 2
) {
  for (let i = 0; i <= maxRetries; i++) {
    try {
      const { data, error } = await supabase.functions.invoke(functionName, { body });
      if (error) throw error;
      return data;
    } catch (err) {
      if (i === maxRetries) throw err;
      // 指数バックオフで待機
      await new Promise(r => setTimeout(r, 1000 * Math.pow(2, i)));
    }
  }
}
```

Edge Functions はコールドスタートで初回呼び出しに1-2秒かかることがあります。リトライと指数バックオフを組み合わせることで、一時的な遅延やネットワークエラーに対応できます。

## 5.5 Edge Functionsのデプロイ

開発が完了したら、以下のコマンドでデプロイします。

```bash
# 特定の関数をデプロイ
supabase functions deploy revenuecat-webhook

# 環境変数（シークレット）を設定
supabase secrets set REVENUECAT_WEBHOOK_SECRET=your-secret-here
```

デプロイされた関数は、以下のURLでアクセスできます。

```
https://<project-ref>.supabase.co/functions/v1/<function-name>
```

### CORSの設定

ブラウザから直接 Edge Functions を呼び出す場合は CORS ヘッダーの設定が必要ですが、Expo アプリ（React Native）から呼び出す場合はブラウザではないため CORS の問題は発生しません。`supabase.functions.invoke()` を使えば認証ヘッダーも自動で付与されるため、特別な設定は不要です。

ただし、Webhook として外部サービスから呼び出される場合は、認証をヘッダーベース（Bearer トークン）で行う必要があります。これは第7章で詳しく実装します。

## 5.6 サービス層のバレルエクスポート

Edge Functions の実装とは別に、アプリ側のサービス層を整理しておきましょう。`services/supabase/index.ts` でバレルエクスポートを設定し、各モジュールからのインポートを簡潔にします。

```typescript
// services/supabase/index.ts
export { supabase } from './client';
export {
  signUpWithEmail, signInWithEmail, signInWithGoogle,
  signInWithApple, signOut, resetPassword, getSession, getProfile,
} from './auth';
export {
  createIncome, getIncomeByMonth, getIncomeByYear,
  updateIncome, deleteIncome,
} from './income';
export type { CreateIncomeData, UpdateIncomeData } from './income';
export {
  createExpense, getExpenseByMonth, getExpenseByYear,
  updateExpense, deleteExpense,
} from './expense';
export type { CreateExpenseData, UpdateExpenseData } from './expense';
```

これにより、各ファイルからのインポートが統一されます。

```typescript
// Before: 個別にインポート
import { supabase } from '@/src/services/supabase/client';
import { createIncome } from '@/src/services/supabase/income';

// After: バレルエクスポートから一括インポート
import { supabase, createIncome } from '@/src/services/supabase';
```

## この章の成果物

ここまでで、以下の状態になっているはずです。

- Supabase CLI がインストールされ、ローカル開発環境が構築されている
- Edge Functions の基本構造（リクエスト処理、DB操作、レスポンス）を理解した
- アプリから `supabase.functions.invoke()` で Edge Functions を呼び出す方法を理解した
- Edge Functions のデプロイと環境変数の設定方法を把握した
- サービス層のバレルエクスポートが整理されている

次の章では、いよいよ RevenueCat を使ったサブスクリプション課金を実装します。

## ソースコード

### `services/supabase/index.ts`

```typescript
export { supabase } from './client';
export {
  signUpWithEmail,
  signInWithEmail,
  signInWithGoogle,
  signInWithApple,
  signOut,
  resetPassword,
  getSession,
  getProfile,
} from './auth';
export {
  createIncome,
  getIncomeByMonth,
  getIncomeByYear,
  updateIncome,
  deleteIncome,
} from './income';
export type { CreateIncomeData, UpdateIncomeData } from './income';
export {
  createExpense,
  getExpenseByMonth,
  getExpenseByYear,
  updateExpense,
  deleteExpense,
} from './expense';
export type { CreateExpenseData, UpdateExpenseData } from './expense';
```
