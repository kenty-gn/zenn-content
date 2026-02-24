---
title: "第4章: メイン機能の実装 — 収入・経費の記録"
free: false
---

# 第4章: メイン機能の実装 — 収入・経費の記録

<!-- TODO: 執筆予定 Week 3-4 -->
<!-- アウトライン:
  4.1 タブナビゲーションの設計
  4.2 ホーム画面 — 20万円ダッシュボード
  4.3 収入入力画面
  4.4 経費入力画面
  4.5 記録一覧画面
  4.6 データの型定義
-->


## ソースコード

### `hooks/useRecords.ts`

```typescript
import { useCallback, useState } from 'react';
import { Alert } from 'react-native';
import { useRecordStore } from '@/src/stores/recordStore';
import { useAuthStore } from '@/src/stores/authStore';
import type { CreateIncomeData, CreateExpenseData } from '@/src/services/supabase';

interface UseRecordsReturn {
  addIncome: (data: Omit<CreateIncomeData, 'user_id'>) => Promise<boolean>;
  addExpense: (data: Omit<CreateExpenseData, 'user_id'>) => Promise<boolean>;
  deleteRecord: (id: string, type: 'income' | 'expense') => Promise<boolean>;
  isSubmitting: boolean;
  error: string | null;
  clearError: () => void;
}

export function useRecords(): UseRecordsReturn {
  const storeAddIncome = useRecordStore(s => s.addIncome);
  const storeAddExpense = useRecordStore(s => s.addExpense);
  const storeDeleteRecord = useRecordStore(s => s.deleteRecord);
  const user = useAuthStore(state => state.user);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const addIncome = useCallback(
    async (data: Omit<CreateIncomeData, 'user_id'>): Promise<boolean> => {
      setError(null);
      setIsSubmitting(true);
      try {
        if (!user) throw new Error('認証されていません');

        await storeAddIncome({ ...data, user_id: user.id });
        return true;
      } catch (err) {
        const message = err instanceof Error ? err.message : '収入の追加に失敗しました';
        setError(message);
        return false;
      } finally {
        setIsSubmitting(false);
      }
    },
    [storeAddIncome, user]
  );

  const addExpense = useCallback(
    async (data: Omit<CreateExpenseData, 'user_id'>): Promise<boolean> => {
      setError(null);
      setIsSubmitting(true);
      try {
        if (!user) throw new Error('認証されていません');

        await storeAddExpense({ ...data, user_id: user.id });
        return true;
      } catch (err) {
        const message = err instanceof Error ? err.message : '経費の追加に失敗しました';
        setError(message);
        return false;
      } finally {
        setIsSubmitting(false);
      }
    },
    [storeAddExpense, user]
  );

  const deleteRecord = useCallback(
    async (id: string, type: 'income' | 'expense'): Promise<boolean> => {
      return new Promise(resolve => {
        const label = type === 'income' ? '収入' : '経費';
        Alert.alert(
          `${label}の削除`,
          `この${label}記録を削除しますか？`,
          [
            { text: 'キャンセル', style: 'cancel', onPress: () => resolve(false) },
            {
              text: '削除',
              style: 'destructive',
              onPress: async () => {
                setError(null);
                setIsSubmitting(true);
                try {
                  await storeDeleteRecord(id, type);
                  resolve(true);
                } catch (err) {
                  const message =
                    err instanceof Error ? err.message : `${label}の削除に失敗しました`;
                  setError(message);
                  resolve(false);
                } finally {
                  setIsSubmitting(false);
                }
              },
            },
          ]
        );
      });
    },
    [storeDeleteRecord]
  );

  const clearError = useCallback(() => setError(null), []);

  return {
    addIncome,
    addExpense,
    deleteRecord,
    isSubmitting,
    error,
    clearError,
  };
}
```

### `hooks/useSummary.ts`

```typescript
import { useMemo } from 'react';
import { useRecordStore } from '@/src/stores/recordStore';

type MergedRecord = {
  id: string;
  type: 'income' | 'expense';
  amount: number;
  label: string;
  date: string;
  category: string;
};

export function useSummary() {
  const incomeRecords = useRecordStore(s => s.incomeRecords);
  const expenseRecords = useRecordStore(s => s.expenseRecords);
  const currentYear = useRecordStore(s => s.currentYear);
  const currentMonth = useRecordStore(s => s.currentMonth);
  const isLoading = useRecordStore(s => s.isLoading);
  const getMonthlySummaryFn = useRecordStore(s => s.getMonthlySummary);
  const getYearlySummaryFn = useRecordStore(s => s.getYearlySummary);

  const monthlySummary = useMemo(() => {
    return getMonthlySummaryFn();
  }, [incomeRecords, expenseRecords, currentYear, currentMonth, getMonthlySummaryFn]);

  const yearlySummary = useMemo(() => {
    return getYearlySummaryFn();
  }, [incomeRecords, expenseRecords, currentYear, getYearlySummaryFn]);

  const getRecentRecords = useMemo(() => {
    return (limit: number = 10): MergedRecord[] => {
      const incomeItems: MergedRecord[] = incomeRecords.map(r => ({
        id: r.id,
        type: 'income' as const,
        amount: r.amount,
        label: r.platform || r.category,
        date: r.recorded_at,
        category: r.category,
      }));

      const expenseItems: MergedRecord[] = expenseRecords.map(r => ({
        id: r.id,
        type: 'expense' as const,
        amount: r.effective_amount,
        label: r.description || r.category,
        date: r.recorded_at,
        category: r.category,
      }));

      return [...incomeItems, ...expenseItems]
        .sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime())
        .slice(0, limit);
    };
  }, [incomeRecords, expenseRecords]);

  return {
    monthlySummary,
    yearlySummary,
    getRecentRecords,
    currentYear,
    currentMonth,
    isLoading,
  };
}
```

### `constants/platforms.ts`

```typescript
/**
 * プラットフォームカテゴリ
 */
export type PlatformCategory = 'contract' | 'content' | 'app' | 'other';

/**
 * プラットフォームカテゴリ表示名
 */
export const PLATFORM_CATEGORY_LABELS: Record<PlatformCategory, string> = {
  contract: '受託開発',
  content: 'コンテンツ',
  app: 'アプリ',
  other: 'その他',
};

/**
 * デフォルトプラットフォーム定義
 */
export interface PlatformDef {
  name: string;
  category: PlatformCategory;
  sort_order: number;
}

/**
 * デフォルトプラットフォーム一覧
 * 新規ユーザー登録時に設定される
 */
export const DEFAULT_PLATFORMS: PlatformDef[] = [
  // 受託開発
  { name: 'クラウドワークス', category: 'contract', sort_order: 1 },
  { name: 'ランサーズ', category: 'contract', sort_order: 2 },
  { name: '直接契約', category: 'contract', sort_order: 3 },
  { name: 'ココナラ', category: 'contract', sort_order: 4 },
  // コンテンツ
  { name: 'Zenn', category: 'content', sort_order: 5 },
  { name: 'note', category: 'content', sort_order: 6 },
  { name: 'Qiita', category: 'content', sort_order: 7 },
  { name: 'YouTube', category: 'content', sort_order: 8 },
  { name: 'Udemy', category: 'content', sort_order: 9 },
  // アプリ
  { name: 'App Store', category: 'app', sort_order: 10 },
  { name: 'Google Play', category: 'app', sort_order: 11 },
  // その他
  { name: 'GitHub Sponsors', category: 'other', sort_order: 12 },
  { name: '技術書典', category: 'other', sort_order: 13 },
  { name: '講演・登壇', category: 'other', sort_order: 14 },
  { name: 'その他', category: 'other', sort_order: 99 },
];
```

### `constants/categories.ts`

```typescript
import type { ExpenseCategory } from '../types/expense';

/**
 * デフォルト経費カテゴリ定義
 */
export interface ExpenseCategoryDef {
  key: ExpenseCategory;
  name: string;
  icon: string;
  default_ratio: number;  // デフォルト按分率（%）
  sort_order: number;
}

/**
 * デフォルト経費カテゴリ一覧
 * 新規ユーザー登録時に設定される
 */
export const DEFAULT_EXPENSE_CATEGORIES: ExpenseCategoryDef[] = [
  { key: 'pc_gadget', name: 'PC・ガジェット', icon: 'laptop', default_ratio: 50, sort_order: 1 },
  { key: 'cloud_service', name: 'クラウドサービス', icon: 'cloud', default_ratio: 100, sort_order: 2 },
  { key: 'dev_tool', name: '開発ツール', icon: 'tool', default_ratio: 100, sort_order: 3 },
  { key: 'book_education', name: '技術書・教材', icon: 'book', default_ratio: 100, sort_order: 4 },
  { key: 'domain_server', name: 'ドメイン・サーバー', icon: 'globe', default_ratio: 100, sort_order: 5 },
  { key: 'communication', name: '通信費', icon: 'wifi', default_ratio: 30, sort_order: 6 },
  { key: 'electricity', name: '電気代', icon: 'zap', default_ratio: 20, sort_order: 7 },
  { key: 'app_registration', name: 'Apple/Google登録料', icon: 'smartphone', default_ratio: 100, sort_order: 8 },
  { key: 'payment_fee', name: '決済手数料', icon: 'credit-card', default_ratio: 100, sort_order: 9 },
  { key: 'coworking', name: 'コワーキングスペース', icon: 'building', default_ratio: 100, sort_order: 10 },
  { key: 'transportation', name: '交通費', icon: 'train', default_ratio: 100, sort_order: 11 },
  { key: 'other', name: 'その他', icon: 'more-horizontal', default_ratio: 100, sort_order: 99 },
];
```
