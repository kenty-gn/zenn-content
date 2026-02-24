---
title: "第6章: RevenueCatでサブスクリプション課金"
free: false
---

# 第6章: RevenueCatでサブスクリプション課金

この章では、RevenueCat を使ってサブスクリプション課金を実装します。RevenueCat SDK の初期化から、商品情報の取得、購入処理、課金状態の管理、そして Pro 機能の出し分けまでを一通り実装します。

## 6.1 RevenueCatとは

RevenueCat は、モバイルアプリのサブスクリプション課金を簡単にする SDK + バックエンドサービスです。

### なぜ自前で IAP を実装しないのか

Apple の StoreKit や Google の Billing Library を直接使って課金を実装すると、以下の作業が必要になります。

1. **レシート検証**: 購入後にサーバー側でレシートの正当性を Apple/Google に問い合わせる
2. **サブスクリプションの状態管理**: 更新、キャンセル、期限切れ、払い戻し、グレースピリオドなど多数の状態遷移を処理
3. **クロスプラットフォーム対応**: iOS と Android で全く異なるAPIを使う
4. **サーバー通知の処理**: Apple/Google からのサーバー通知（更新、キャンセル等）を受け取る

RevenueCat はこれらすべてを抽象化し、シンプルな API で提供してくれます。料金は月間収益 $2,500 まで無料なので、個人開発の初期段階では費用がかかりません。

## 6.2 App Store Connect側の設定

RevenueCat を使う前に、App Store Connect でサブスクリプション商品を設定します。

### サブスクリプショングループの作成

1. App Store Connect で対象アプリを選択
2. **サブスクリプション** > **サブスクリプショングループを作成**
3. グループ名: `Pro Plan`

### 商品IDの設定

ふくログでは月額と年額の2つのプランを用意します。

| 商品ID | プラン | 価格 |
|---|---|---|
| `pro_monthly` | 月額 | 490円 |
| `pro_annual` | 年額 | 4,900円（月額換算408円、約17%お得） |

年額プランを月額の約10ヶ月分に設定するのは、App Store のサブスクリプションでよく使われるプライシング戦略です。

### サンドボックステスターの追加

App Store Connect の **ユーザーとアクセス** > **Sandbox** > **テスターアカウント** から、テスト用の Apple ID を追加します。このアカウントを使うと、課金が実際には発生せず、サブスクリプションの更新間隔も短縮されます（月額が5分に短縮）。

## 6.3 RevenueCatダッシュボードの設定

1. [RevenueCat](https://www.revenuecat.com/) でアカウントを作成
2. **Create New Project** でプロジェクトを作成
3. **Apps** > **Add App** で iOS アプリを追加
4. App Store Connect の **App-Specific Shared Secret** を RevenueCat に登録
5. **Entitlements** で `pro` を作成（Pro ユーザーが持つ権限）
6. **Products** に `pro_monthly` と `pro_annual` を登録し、`pro` エンタイトルメントに紐づける
7. **Offerings** > **Current** に月額と年額のパッケージを追加

**Entitlement（権限）** という概念が重要です。個別の商品ID（`pro_monthly`, `pro_annual`）ではなく、権限（`pro`）でアクセスを制御することで、後から商品構成を変更しても（例: 週額プランを追加しても）アプリ側のコードを変更する必要がありません。

## 6.4 Expoアプリへの組み込み

### パッケージのインストール

```bash
npx expo install react-native-purchases
```

:::message
`react-native-purchases` はネイティブモジュールを含むため、Expo Go では動作しません。**Development Build**（`eas build --profile development`）を使って実機でテストする必要があります。
:::

### RevenueCatサービス層の実装

RevenueCat SDK の操作を `services/revenueCat.ts` にまとめます。

```typescript
import { Platform } from 'react-native';
import { REVENUE_CAT } from '@/src/constants/config';

const isNative = Platform.OS === 'ios' || Platform.OS === 'android';

let Purchases: typeof import('react-native-purchases').default | null = null;

async function getPurchases() {
  if (!isNative) return null;
  if (!Purchases) {
    const mod = await import('react-native-purchases');
    Purchases = mod.default;
  }
  return Purchases;
}
```

`getPurchases()` で動的インポートを使っているのは、Web 環境で `react-native-purchases` をインポートするとエラーになるためです。`isNative` チェックを入れることで、Web でのプレビューやテストが可能になります。

### SDK初期化

```typescript
export async function initializeRevenueCat(userId: string): Promise<void> {
  const sdk = await getPurchases();
  if (!sdk) return;

  const apiKey = Platform.OS === 'ios'
    ? REVENUE_CAT.appleApiKey
    : REVENUE_CAT.googleApiKey;

  if (!apiKey) return;

  sdk.configure({ apiKey, appUserID: userId });
}
```

`appUserID` に Supabase の `user.id` を渡しています。これにより、RevenueCat のユーザーと Supabase のユーザーが同じ ID で紐づきます。この紐づけが、第7章の Webhook でユーザーを特定するために不可欠です。

## 6.5 購入処理の実装

### 月額・年額の購入

```typescript
export async function purchaseMonthly(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const offerings = await sdk.getOfferings();
    const monthlyPackage = offerings.current?.monthly;
    if (!monthlyPackage) return false;

    const { customerInfo } = await sdk.purchasePackage(monthlyPackage);
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}
```

`sdk.getOfferings()` で RevenueCat に設定した商品パッケージを取得し、`sdk.purchasePackage()` で購入フローを開始します。購入が成功すると `customerInfo` が返り、`entitlements.active` に購入した権限が含まれます。

購入処理の戻り値を `boolean` にしている理由は、購入がキャンセルされた場合や失敗した場合をシンプルに扱うためです。RevenueCat SDK はユーザーキャンセル時にもエラーを throw するため、`catch` で `false` を返して処理を統一しています。

### 購入復元

App Store の審査では「復元する」ボタンの設置が必須です。機種変更後に以前の購入を復元するために使います。

```typescript
export async function restorePurchases(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const customerInfo = await sdk.restorePurchases();
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}
```

## 6.6 課金状態の管理（subscriptionStore）

Zustand ストアで課金状態を一元管理します。

```typescript
export const useSubscriptionStore = create<SubscriptionStore>((set, get) => ({
  isPro: false,
  isLoading: false,
  isNative,

  initialize: async (userId: string) => {
    set({ isLoading: true });

    // まず Supabase の subscription_tier を確認（Webhook で更新済み）
    const profile = useAuthStore.getState().profile;
    if (profile?.subscription_tier === 'pro') {
      set({ isPro: true, isLoading: false });
    }

    if (!isNative) {
      set({ isLoading: false });
      return;
    }

    // RevenueCat SDK で最新の課金状態を取得
    await initializeRevenueCat(userId);
    const isPro = await checkProEntitlement();
    set({ isPro, isLoading: false });

    // リアルタイム変更監視
    listenerCleanup?.();
    listenerCleanup = addCustomerInfoListener((newIsPro) => {
      set({ isPro: newIsPro });
    });
  },
  // ...
}));
```

課金状態の判定には **2つのソース** を使っています。

1. **Supabase の `subscription_tier`**: Webhook（第7章で実装）によってサーバーサイドで更新される。信頼性が高い
2. **RevenueCat SDK の `checkProEntitlement()`**: クライアント側でリアルタイムに確認。ネイティブ環境でのみ利用可能

Supabase のデータを先にチェックすることで、RevenueCat SDK が初期化される前でも Pro 状態を反映できます。

### リアルタイム変更監視

```typescript
export function addCustomerInfoListener(
  onUpdate: (isPro: boolean) => void,
): () => void {
  if (!isNative) return () => {};

  const handler = (info: { entitlements: { active: Record<string, unknown> } }) => {
    const isPro = info.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
    onUpdate(isPro);
  };

  getPurchases().then(mod => {
    if (!mod) return;
    mod.addCustomerInfoUpdateListener(handler);
  });

  return () => { /* クリーンアップ */ };
}
```

`addCustomerInfoUpdateListener` は、サブスクリプションの状態が変わったとき（更新、キャンセル、期限切れ等）にコールバックを呼び出します。これにより、ユーザーがApp Store の設定からサブスクリプションをキャンセルした場合でも、アプリ内の表示がリアルタイムに更新されます。

## 6.7 Pro機能の出し分け

### useSubscriptionフック

```typescript
export function useSubscription() {
  const isPro = useSubscriptionStore(s => s.isPro);
  const isLoading = useSubscriptionStore(s => s.isLoading);
  const isNative = useSubscriptionStore(s => s.isNative);
  const router = useRouter();

  const requirePro = useCallback((): boolean => {
    if (isPro) return true;
    router.push('/paywall');
    return false;
  }, [isPro, router]);

  return { isPro, isLoading, isNative, requirePro };
}
```

`requirePro()` は Pro 限定機能にアクセスしようとしたときに呼び出すガード関数です。Pro ユーザーならそのまま処理を続行し、無料ユーザーならペイウォール画面に遷移します。

### 画面での使い方

```typescript
function ExportButton() {
  const { isPro, requirePro } = useSubscription();

  const handleExport = () => {
    if (!requirePro()) return; // 無料ユーザー → ペイウォールへ
    // Pro ユーザー → CSV出力処理
    exportToCSV();
  };

  return (
    <TouchableOpacity onPress={handleExport}>
      <Text>CSVエクスポート</Text>
      {!isPro && <Badge text="Pro" />}
    </TouchableOpacity>
  );
}
```

無料ユーザーには「Pro」バッジを表示して、タップするとペイウォールに遷移する流れです。機能自体を完全に隠すのではなく「見せるけどロックする」デザインにすることで、Pro プランのアップグレード動機を作ります。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- App Store Connect でサブスクリプション商品（月額・年額）が設定されている
- RevenueCat ダッシュボードで Entitlement と Offering が設定されている
- アプリ内で RevenueCat SDK が初期化され、購入処理が動く
- サンドボックスでテスト購入ができる
- `isPro` フラグで Pro/無料の機能出し分けが実装されている

しかし、ここまでの実装には重要な欠点があります。課金状態の判定がクライアント側のみで行われているため、改ざんリスクがあります。次の章では、RevenueCat の Webhook と Supabase Edge Functions を連携させ、**サーバーサイドで課金状態を安全に同期する仕組み** を実装します。

## ソースコード

### `services/revenueCat.ts`

```typescript
import { Platform } from 'react-native';
import { REVENUE_CAT } from '@/src/constants/config';

const isNative = Platform.OS === 'ios' || Platform.OS === 'android';

let Purchases: typeof import('react-native-purchases').default | null = null;

async function getPurchases() {
  if (!isNative) return null;
  if (!Purchases) {
    const mod = await import('react-native-purchases');
    Purchases = mod.default;
  }
  return Purchases;
}

/**
 * RevenueCat SDK初期化
 */
export async function initializeRevenueCat(userId: string): Promise<void> {
  const sdk = await getPurchases();
  if (!sdk) return;

  const apiKey = Platform.OS === 'ios'
    ? REVENUE_CAT.appleApiKey
    : REVENUE_CAT.googleApiKey;

  if (!apiKey) return;

  sdk.configure({ apiKey, appUserID: userId });
}

/**
 * Pro権限チェック
 */
export async function checkProEntitlement(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const customerInfo = await sdk.getCustomerInfo();
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}

/**
 * 商品情報取得
 */
export async function getOfferings() {
  const sdk = await getPurchases();
  if (!sdk) return null;

  try {
    const offerings = await sdk.getOfferings();
    return offerings.current;
  } catch {
    return null;
  }
}

/**
 * 月額購入
 */
export async function purchaseMonthly(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const offerings = await sdk.getOfferings();
    const monthlyPackage = offerings.current?.monthly;
    if (!monthlyPackage) return false;

    const { customerInfo } = await sdk.purchasePackage(monthlyPackage);
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}

/**
 * 年額購入
 */
export async function purchaseAnnual(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const offerings = await sdk.getOfferings();
    const annualPackage = offerings.current?.annual;
    if (!annualPackage) return false;

    const { customerInfo } = await sdk.purchasePackage(annualPackage);
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}

/**
 * 購入復元
 */
export async function restorePurchases(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  try {
    const customerInfo = await sdk.restorePurchases();
    return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
  } catch {
    return false;
  }
}

/**
 * リアルタイム変更監視
 */
export function addCustomerInfoListener(
  onUpdate: (isPro: boolean) => void,
): () => void {
  if (!isNative) return () => {};

  let sdk: typeof import('react-native-purchases').default | null = null;
  const handler = (info: { entitlements: { active: Record<string, unknown> } }) => {
    const isPro = info.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
    onUpdate(isPro);
  };

  getPurchases().then(mod => {
    if (!mod) return;
    sdk = mod;
    sdk.addCustomerInfoUpdateListener(handler);
  });

  return () => {
    sdk?.removeCustomerInfoUpdateListener(handler);
  };
}

export { isNative };
```

### `hooks/useSubscription.ts`

```typescript
import { useCallback } from 'react';
import { useRouter } from 'expo-router';
import { useSubscriptionStore } from '@/src/stores/subscriptionStore';

export function useSubscription() {
  const isPro = useSubscriptionStore(s => s.isPro);
  const isLoading = useSubscriptionStore(s => s.isLoading);
  const isNative = useSubscriptionStore(s => s.isNative);
  const router = useRouter();

  const requirePro = useCallback((): boolean => {
    if (isPro) return true;
    router.push('/paywall');
    return false;
  }, [isPro, router]);

  return { isPro, isLoading, isNative, requirePro };
}
```

### `stores/subscriptionStore.ts`

```typescript
import { create } from 'zustand';
import {
  initializeRevenueCat,
  checkProEntitlement,
  purchaseMonthly,
  purchaseAnnual,
  restorePurchases,
  addCustomerInfoListener,
  isNative,
} from '@/src/services/revenueCat';
import { useAuthStore } from '@/src/stores/authStore';

interface SubscriptionState {
  isPro: boolean;
  isLoading: boolean;
  isNative: boolean;
}

interface SubscriptionActions {
  initialize: (userId: string) => Promise<void>;
  purchase: (plan?: 'monthly' | 'annual') => Promise<boolean>;
  restore: () => Promise<boolean>;
  reset: () => void;
}

type SubscriptionStore = SubscriptionState & SubscriptionActions;

let listenerCleanup: (() => void) | null = null;

export const useSubscriptionStore = create<SubscriptionStore>((set, get) => ({
  isPro: false,
  isLoading: false,
  isNative,

  initialize: async (userId: string) => {
    set({ isLoading: true });

    // Supabase の subscription_tier を確認（webhook で更新済み）
    const profile = useAuthStore.getState().profile;
    if (profile?.subscription_tier === 'pro') {
      set({ isPro: true, isLoading: false });
    }

    if (!isNative) {
      set({ isLoading: false });
      return;
    }

    try {
      await initializeRevenueCat(userId);
      const isPro = await checkProEntitlement();
      set({ isPro, isLoading: false });

      // リアルタイム変更監視（ローカル状態のみ更新、DB は webhook が処理）
      listenerCleanup?.();
      listenerCleanup = addCustomerInfoListener((newIsPro) => {
        set({ isPro: newIsPro });
      });
    } catch {
      set({ isLoading: false });
    }
  },

  purchase: async (plan = 'monthly') => {
    set({ isLoading: true });
    try {
      const success = plan === 'annual'
        ? await purchaseAnnual()
        : await purchaseMonthly();
      if (success) {
        set({ isPro: true });
        // DB の subscription_tier は RevenueCat webhook → Edge Function が更新する
        // ローカルプロフィールを再取得して同期
        useAuthStore.getState().refreshProfile();
      }
      return success;
    } finally {
      set({ isLoading: false });
    }
  },

  restore: async () => {
    set({ isLoading: true });
    try {
      const success = await restorePurchases();
      if (success) {
        set({ isPro: true });
        useAuthStore.getState().refreshProfile();
      }
      return success;
    } finally {
      set({ isLoading: false });
    }
  },

  reset: () => {
    listenerCleanup?.();
    listenerCleanup = null;
    set({ isPro: false, isLoading: false });
  },
}));
```
