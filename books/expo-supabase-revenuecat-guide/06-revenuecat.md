---
title: "第6章: RevenueCatでサブスクリプション課金"
free: false
---

# 第6章: RevenueCatでサブスクリプション課金

<!-- TODO: 執筆予定 Week 5-6 -->
<!-- アウトライン:
  6.1 RevenueCatとは
  6.2 App Store Connect側の設定
  6.3 RevenueCatダッシュボードの設定
  6.4 Expoアプリへの組み込み
  6.5 ペイウォール（課金画面）の実装
  6.6 課金状態の確認
  6.7 Pro機能の出し分け
-->


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
