---
title: "第7章: Supabase × RevenueCat連携 — Webhookで課金状態を同期"
free: false
---

# 第7章: Supabase x RevenueCat連携 --- Webhookで課金状態を同期

この章では、RevenueCat の Webhook と Supabase Edge Functions を連携させ、課金状態をサーバーサイドで安全に管理する仕組みを実装します。この章がふくログのアーキテクチャの要であり、Expo + Supabase + RevenueCat を統合する最も重要なピースです。

## 7.1 なぜWebhookが必要か

前章では、RevenueCat SDK を使ってクライアント側で課金状態を判定しました。しかし、この方法だけでは以下の問題があります。

**セキュリティの問題**: クライアント側の `isPro` フラグは、技術的にはメモリを書き換えることで改ざん可能です。Pro 限定機能の制御をクライアントだけに頼ると、セキュリティ上のリスクになります。

**サーバーサイドでの権限チェック**: RLS ポリシーで「Pro ユーザーだけがアクセスできるデータ」を制御したい場合、データベースに課金状態が保存されている必要があります。

**オフライン時の状態管理**: RevenueCat SDK はネットワークが必要ですが、Supabase のデータベースに課金状態を保存しておけば、キャッシュされたプロフィール情報から判定できます。

Webhook を使った連携フローは以下のようになります。

```
ユーザーが購入
  → Apple/Google が RevenueCat に通知
    → RevenueCat が Webhook を送信
      → Supabase Edge Function が受信
        → profiles テーブルの subscription_tier を更新
```

この仕組みにより、課金状態が **サーバーサイドの正規データ** として管理されます。

## 7.2 RevenueCat Webhookの設定

### Webhook URLの登録

1. RevenueCat ダッシュボードで対象プロジェクトを選択
2. **Integrations** > **Webhooks** を選択
3. **Webhook URL** に Edge Function の URL を入力:
   ```
   https://<project-ref>.supabase.co/functions/v1/revenuecat-webhook
   ```
4. **Authorization header** に `Bearer <your-secret>` を設定

### Webhookで送信されるイベント

RevenueCat は様々なイベントを Webhook で送信します。ふくログで処理するイベントを整理しておきます。

| イベント | 意味 | 処理 |
|---|---|---|
| `INITIAL_PURCHASE` | 初回購入 | Pro に昇格 |
| `RENEWAL` | サブスクリプション更新 | Pro を維持 |
| `UNCANCELLATION` | キャンセル取り消し | Pro に復帰 |
| `NON_RENEWING_PURCHASE` | 非更新型の購入 | Pro に昇格 |
| `PRODUCT_CHANGE` | プラン変更（月額→年額等） | Pro を維持 |
| `EXPIRATION` | サブスクリプション期限切れ | Free に降格 |

`CANCELLATION`（キャンセル）イベントでは即座に Free にしない点に注目してください。Apple のサブスクリプションでは、キャンセルしても現在の課金期間が終了するまでは有効です。実際に権限を取り消すのは `EXPIRATION` が来たときです。

## 7.3 Edge Functionの実装

### 完全なコード

```typescript
// supabase/functions/revenuecat-webhook/index.ts
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2'

const WEBHOOK_SECRET = Deno.env.get('REVENUECAT_WEBHOOK_SECRET')
const SUPABASE_URL = Deno.env.get('SUPABASE_URL')!
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!

// Pro を付与するイベント
const PRO_EVENTS = new Set([
  'INITIAL_PURCHASE',
  'RENEWAL',
  'UNCANCELLATION',
  'NON_RENEWING_PURCHASE',
  'PRODUCT_CHANGE',
])

// Pro を取り消すイベント
const FREE_EVENTS = new Set([
  'EXPIRATION',
])

Deno.serve(async (req) => {
  if (req.method !== 'POST') {
    return new Response('Method Not Allowed', { status: 405 })
  }

  // Webhook 認証
  if (WEBHOOK_SECRET) {
    const authHeader = req.headers.get('Authorization')
    if (authHeader !== `Bearer ${WEBHOOK_SECRET}`) {
      return new Response('Unauthorized', { status: 401 })
    }
  }

  try {
    const { event } = await req.json()
    const userId = event?.app_user_id
    const eventType = event?.type

    if (!userId || !eventType) {
      return new Response(
        JSON.stringify({ error: 'Missing app_user_id or type' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } },
      )
    }

    let newTier: string | null = null

    if (PRO_EVENTS.has(eventType)) {
      newTier = 'pro'
    } else if (FREE_EVENTS.has(eventType)) {
      newTier = 'free'
    }

    if (newTier) {
      const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY)
      const { error } = await supabase
        .from('profiles')
        .update({ subscription_tier: newTier })
        .eq('id', userId)

      if (error) {
        console.error('Failed to update subscription_tier:', error.message)
        return new Response(
          JSON.stringify({ error: error.message }),
          { status: 500, headers: { 'Content-Type': 'application/json' } },
        )
      }
    }

    return new Response(
      JSON.stringify({ success: true, event_type: eventType, new_tier: newTier }),
      { status: 200, headers: { 'Content-Type': 'application/json' } },
    )
  } catch (err) {
    console.error('Webhook processing error:', err)
    return new Response(
      JSON.stringify({ error: 'Internal Server Error' }),
      { status: 500, headers: { 'Content-Type': 'application/json' } },
    )
  }
})
```

### コードの解説

**認証チェック**: `Authorization` ヘッダーの Bearer トークンを検証します。RevenueCat ダッシュボードで設定したシークレットと一致しない場合は `401 Unauthorized` を返します。これにより、不正なリクエストによる課金状態の改ざんを防ぎます。

**`app_user_id` の取得**: RevenueCat は Webhook ペイロードの `event.app_user_id` にユーザーIDを含めます。第6章で `sdk.configure({ appUserID: userId })` に Supabase の `user.id` を渡したので、ここで取得できるIDがそのまま Supabase のユーザーIDになります。

**`service_role` キーの使用**: Edge Function 内では `SUPABASE_SERVICE_ROLE_KEY` を使って Supabase クライアントを作成します。これは RLS をバイパスして profiles テーブルを直接更新するためです。Webhook はサーバー間通信なので、ユーザーのセッションは存在しません。

**イベントの分類**: `Set` を使ってイベントタイプを Pro 付与と Free 降格に分類しています。処理対象外のイベント（`CANCELLATION` 等）は `newTier` が `null` のまま、DBを更新せずに `200 OK` を返します。RevenueCat は `200` 以外のレスポンスを受け取るとリトライするため、処理対象外でも必ず `200` を返すことが重要です。

## 7.4 デプロイとテスト

### デプロイ

```bash
# シークレットの設定
supabase secrets set REVENUECAT_WEBHOOK_SECRET=your-webhook-secret

# Edge Function のデプロイ
supabase functions deploy revenuecat-webhook
```

`SUPABASE_URL` と `SUPABASE_SERVICE_ROLE_KEY` は Supabase が自動的に設定するため、手動で設定する必要はありません。

### テスト方法

1. **サンドボックスでテスト購入**: 実機で Sandbox テスターアカウントを使って購入
2. **RevenueCat ダッシュボードで配信ログを確認**: **Integrations > Webhooks** の下部に Webhook の配信履歴が表示される。ステータスコード `200` が返っていることを確認
3. **Supabase ダッシュボードで確認**: **Table Editor > profiles** テーブルで、対象ユーザーの `subscription_tier` が `pro` に更新されていることを確認

:::message
Webhook のデバッグが難しい場合は、Supabase ダッシュボードの **Edge Functions > Logs** でリアルタイムのログを確認できます。`console.error` で出力したメッセージがここに表示されます。
:::

## 7.5 アプリ側の同期

Webhook で DB が更新された後、アプリ側でも状態を同期する必要があります。`subscriptionStore` の `purchase` アクションでは、購入成功後に `refreshProfile()` を呼び出してプロフィールを再取得しています。

```typescript
purchase: async (plan = 'monthly') => {
  const success = plan === 'annual'
    ? await purchaseAnnual()
    : await purchaseMonthly();
  if (success) {
    set({ isPro: true });
    // DB の subscription_tier は Webhook → Edge Function が更新する
    // ローカルのプロフィールを再取得して同期
    useAuthStore.getState().refreshProfile();
  }
  return success;
},
```

ここでの処理順序は以下のとおりです。

1. RevenueCat SDK で購入処理 → 成功
2. ローカルの `isPro` を `true` にセット（即座にUIに反映）
3. RevenueCat が Webhook を送信 → Edge Function が DB を更新（非同期、数秒かかる場合あり）
4. `refreshProfile()` で Supabase からプロフィールを再取得 → `subscription_tier: 'pro'` を確認

ステップ2で即座にローカル状態を更新するのは、Webhook の処理が完了するまでの数秒間もユーザーに Pro 体験を提供するためです。実際の権限の正規データはステップ3で DB に書き込まれます。

## 7.6 本番環境への移行チェックリスト

サンドボックスでのテストが完了したら、本番環境への移行準備を行います。

- [ ] App Store Connect で税務・契約の設定が完了している
- [ ] サブスクリプション商品が「審査待ち」または「承認済み」になっている
- [ ] RevenueCat ダッシュボードで本番用の API キーを確認
- [ ] `.env` の API キーを本番用に更新
- [ ] Webhook URL がデプロイ済みの Edge Function を指している
- [ ] Edge Function のシークレットが本番用に設定されている
- [ ] サンドボックスとは別のテスターで最終確認

:::message alert
App Store Connect で「有料アプリ契約」が完了していないと、サブスクリプション商品を公開できません。銀行口座情報と税務情報の登録が必要です。審査に数日かかる場合があるので、早めに手続きしておきましょう。
:::

## この章の成果物

ここまでで、以下の状態になっているはずです。

- RevenueCat の Webhook が Supabase Edge Function に送信される設定が完了している
- Edge Function がイベントタイプに応じて `subscription_tier` を安全に更新する
- 課金 → Webhook → DB更新 → アプリ同期の一連のフローが動作する
- サーバーサイドで課金状態を検証できるため、改ざんリスクが排除されている

これで Expo + Supabase + RevenueCat の3つのサービスが完全に連携しました。次の章では、このアプリを実際に App Store に公開する手順を解説します。
