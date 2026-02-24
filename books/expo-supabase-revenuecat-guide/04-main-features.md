---
title: "第4章: メイン機能の実装 — 収入・経費の記録"
free: false
---

# 第4章: メイン機能の実装 --- 収入・経費の記録

この章では、ふくログの中核機能を実装します。Zustand でデータを管理する recordStore、収入・経費を操作するカスタムフック、20万円ラインの進捗を表示するサマリー計算、そしてプラットフォーム・経費カテゴリの設計までを一気に作り上げます。

## 4.1 データの型定義

まずは TypeScript の型定義から始めます。型を先に定義しておくことで、以降の実装で IDE の補完が効き、型安全に開発を進められます。

### 収入の型

```typescript
// types/income.ts
export type IncomeCategory = 'contract' | 'content' | 'app' | 'other';

export const INCOME_CATEGORY_LABELS: Record<IncomeCategory, string> = {
  contract: '受託開発',
  content: 'コンテンツ',
  app: 'アプリ',
  other: 'その他',
};

export interface IncomeEntry {
  id: string;
  user_id: string;
  amount: number;
  category: IncomeCategory;
  platform: string;
  description: string | null;
  recorded_at: string;        // YYYY-MM-DD
  tax_year: number;
  has_withholding_tax: boolean;
  withholding_tax_amount: number;
  created_at: string;
  updated_at: string;
}
```

収入カテゴリを4つに絞っているのは、副業エンジニアの収入源が大きく「受託開発」「コンテンツ（記事・動画など）」「アプリ（自社プロダクト）」「その他」に分類できるためです。プラットフォーム（クラウドワークス、Zenn 等）は別カラムで管理し、カテゴリは集計用のグルーピングとして使います。

### 経費の型

```typescript
// types/expense.ts
export type ExpenseCategory =
  | 'pc_gadget' | 'cloud_service' | 'dev_tool' | 'book_education'
  | 'domain_server' | 'communication' | 'electricity' | 'app_registration'
  | 'payment_fee' | 'coworking' | 'transportation' | 'other';

export interface ExpenseEntry {
  id: string;
  user_id: string;
  amount: number;               // 支払額（按分前の全額）
  category: ExpenseCategory;
  business_ratio: number;        // 按分率（0-100）
  effective_amount: number;      // 経費計上額（amount * business_ratio / 100）
  description: string | null;
  recorded_at: string;
  tax_year: number;
  created_at: string;
  updated_at: string;
}
```

経費カテゴリは確定申告の勘定科目に近い分類にしています。エンジニア特有の「クラウドサービス」「開発ツール」「Apple/Google登録料」などを用意しておくことで、入力時にカテゴリを選びやすくなります。

## 4.2 プラットフォームと経費カテゴリの設計

### プラットフォーム一覧

収入を記録する際、「どのプラットフォームからの収入か」を選択できるようにします。

```typescript
// constants/platforms.ts
export type PlatformCategory = 'contract' | 'content' | 'app' | 'other';

export const DEFAULT_PLATFORMS: PlatformDef[] = [
  // 受託開発
  { name: 'クラウドワークス', category: 'contract', sort_order: 1 },
  { name: 'ランサーズ',       category: 'contract', sort_order: 2 },
  { name: '直接契約',         category: 'contract', sort_order: 3 },
  { name: 'ココナラ',         category: 'contract', sort_order: 4 },
  // コンテンツ
  { name: 'Zenn',             category: 'content',  sort_order: 5 },
  { name: 'note',             category: 'content',  sort_order: 6 },
  { name: 'Qiita',            category: 'content',  sort_order: 7 },
  { name: 'YouTube',          category: 'content',  sort_order: 8 },
  { name: 'Udemy',            category: 'content',  sort_order: 9 },
  // アプリ
  { name: 'App Store',        category: 'app',      sort_order: 10 },
  { name: 'Google Play',      category: 'app',      sort_order: 11 },
  // その他
  { name: 'GitHub Sponsors',  category: 'other',    sort_order: 12 },
  { name: '技術書典',          category: 'other',    sort_order: 13 },
  { name: '講演・登壇',        category: 'other',    sort_order: 14 },
  { name: 'その他',            category: 'other',    sort_order: 99 },
];
```

プラットフォームはユーザーごとにカスタマイズできるよう、Supabase の `platforms` テーブルにも保存します。新規ユーザーにはデフォルト値をセットし、`useUserPlatforms` フックで取得します。

### 経費カテゴリとデフォルト按分率

```typescript
// constants/categories.ts
export const DEFAULT_EXPENSE_CATEGORIES: ExpenseCategoryDef[] = [
  { key: 'pc_gadget',        name: 'PC・ガジェット',       icon: 'laptop',    default_ratio: 50,  sort_order: 1 },
  { key: 'cloud_service',    name: 'クラウドサービス',     icon: 'cloud',     default_ratio: 100, sort_order: 2 },
  { key: 'dev_tool',         name: '開発ツール',           icon: 'tool',      default_ratio: 100, sort_order: 3 },
  { key: 'book_education',   name: '技術書・教材',         icon: 'book',      default_ratio: 100, sort_order: 4 },
  { key: 'domain_server',    name: 'ドメイン・サーバー',   icon: 'globe',     default_ratio: 100, sort_order: 5 },
  { key: 'communication',    name: '通信費',               icon: 'wifi',      default_ratio: 30,  sort_order: 6 },
  { key: 'electricity',      name: '電気代',               icon: 'zap',       default_ratio: 20,  sort_order: 7 },
  { key: 'app_registration', name: 'Apple/Google登録料',   icon: 'smartphone',default_ratio: 100, sort_order: 8 },
  { key: 'payment_fee',      name: '決済手数料',           icon: 'credit-card',default_ratio: 100, sort_order: 9 },
  { key: 'coworking',        name: 'コワーキングスペース', icon: 'building',  default_ratio: 100, sort_order: 10 },
  { key: 'transportation',   name: '交通費',               icon: 'train',     default_ratio: 100, sort_order: 11 },
  { key: 'other',            name: 'その他',               icon: 'more-horizontal', default_ratio: 100, sort_order: 99 },
];
```

`default_ratio` がデフォルトの按分率です。PC やスマホは私用と事業用の区別が難しいため50%に設定。通信費は30%、電気代は20%をデフォルトにしています。これらはあくまで初期値で、ユーザーが入力時に自由に変更できます。

:::message
按分率は税務調査で争点になりやすいポイントです。デフォルト値は一般的な目安として設定していますが、実際の按分率はユーザー自身の使用実態に基づいて判断する必要があります。アプリ内でも「按分率は実態に応じて設定してください」という注意書きを表示しています。
:::

## 4.3 recordStore — データ管理の中枢

収入・経費のデータ取得、追加、削除、集計を一元管理するストアを作ります。

```typescript
// stores/recordStore.ts
export const useRecordStore = create<RecordStore>((set, get) => ({
  incomeRecords: [],
  expenseRecords: [],
  isLoading: false,
  error: null,
  currentYear: getCurrentYear(),
  currentMonth: getCurrentMonth(),

  fetchMonthRecords: async (year?: number, month?: number) => {
    const y = year ?? getCurrentYear();
    const m = month ?? getCurrentMonth();
    set({ isLoading: true, currentYear: y, currentMonth: m });

    const user = useAuthStore.getState().user;
    if (!user) throw new Error('Not authenticated');

    const [incomeData, expenseData] = await Promise.all([
      getIncomeByMonth(user.id, y, m),
      getExpenseByMonth(user.id, y, m),
    ]);

    set({
      incomeRecords: (incomeData as IncomeEntry[]) || [],
      expenseRecords: (expenseData as ExpenseEntry[]) || [],
      isLoading: false,
    });
  },
  // ...
}));
```

`fetchMonthRecords` では `Promise.all` を使って収入と経費を並列に取得しています。直列に取得すると2回分のネットワークラウンドトリップが発生しますが、並列にすることで体感速度が約半分になります。

### 月次・年次サマリーの計算

ホーム画面で「今月の収入合計」「年間の20万円ライン進捗」を表示するために、ストア内にサマリー計算関数を持たせます。

```typescript
getMonthlySummary: (): MonthlySummaryResult => {
  const state = get();
  const prefix = `${state.currentYear}-${String(state.currentMonth).padStart(2, '0')}`;
  const monthIncomes = state.incomeRecords.filter(r => r.recorded_at.startsWith(prefix));
  const monthExpenses = state.expenseRecords.filter(r => r.recorded_at.startsWith(prefix));

  const totalIncome = monthIncomes.reduce((sum, r) => sum + r.amount, 0);
  const totalExpense = monthExpenses.reduce((sum, r) => sum + r.effective_amount, 0);

  return {
    totalIncome,
    totalExpense,
    netIncome: totalIncome - totalExpense,
    recordCount: monthIncomes.length + monthExpenses.length,
  };
},

getYearlySummary: (): YearlySummaryResult => {
  const state = get();
  const yearIncomes = state.incomeRecords.filter(r => r.tax_year === state.currentYear);
  const yearExpenses = state.expenseRecords.filter(r => r.tax_year === state.currentYear);

  const totalIncome = yearIncomes.reduce((sum, r) => sum + r.amount, 0);
  const totalExpense = yearExpenses.reduce((sum, r) => sum + r.effective_amount, 0);
  const netIncome = totalIncome - totalExpense;

  return {
    totalIncome,
    totalExpense,
    netIncome,
    progressPercent: Math.min((netIncome / 200000) * 100, 100),
  };
},
```

`progressPercent` は年間所得（収入 - 経費）の20万円に対する進捗率です。副業所得が年間20万円を超えると確定申告が必要になるため、この「20万円ライン」がふくログのダッシュボードの中核的な指標になっています。

## 4.4 useRecordsフック — 画面からの操作インターフェース

画面コンポーネントから収入・経費の追加・削除を行うためのフックです。

```typescript
export function useRecords(): UseRecordsReturn {
  const storeAddIncome = useRecordStore(s => s.addIncome);
  const user = useAuthStore(state => state.user);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const addIncome = useCallback(
    async (data: Omit<CreateIncomeData, 'user_id'>): Promise<boolean> => {
      setIsSubmitting(true);
      try {
        if (!user) throw new Error('認証されていません');
        await storeAddIncome({ ...data, user_id: user.id });
        return true;
      } catch (err) {
        setError(err instanceof Error ? err.message : '収入の追加に失敗しました');
        return false;
      } finally {
        setIsSubmitting(false);
      }
    },
    [storeAddIncome, user]
  );

  // ...
}
```

フックの引数は `Omit<CreateIncomeData, 'user_id'>` としています。`user_id` はフック内部で認証ストアから取得するため、画面側は金額やプラットフォームなどの入力データだけを渡せばOKです。これにより、画面コンポーネントが認証ロジックを意識しなくて済みます。

### 削除の確認ダイアログ

```typescript
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
              await storeDeleteRecord(id, type);
              resolve(true);
            },
          },
        ]
      );
    });
  },
  [storeDeleteRecord]
);
```

`Alert.alert` を `Promise` でラップしているのがポイントです。ダイアログの結果を `await` で待てるようにすることで、呼び出し側で `const deleted = await deleteRecord(id, 'income')` のように直感的に使えます。

## 4.5 useSummaryフック — ダッシュボード表示用

ホーム画面で月次・年次のサマリーと直近の記録を表示するためのフックです。

```typescript
export function useSummary() {
  const incomeRecords = useRecordStore(s => s.incomeRecords);
  const expenseRecords = useRecordStore(s => s.expenseRecords);
  const getMonthlySummaryFn = useRecordStore(s => s.getMonthlySummary);
  const getYearlySummaryFn = useRecordStore(s => s.getYearlySummary);

  const monthlySummary = useMemo(() => {
    return getMonthlySummaryFn();
  }, [incomeRecords, expenseRecords, getMonthlySummaryFn]);

  const getRecentRecords = useMemo(() => {
    return (limit: number = 10): MergedRecord[] => {
      const incomeItems = incomeRecords.map(r => ({
        id: r.id, type: 'income' as const,
        amount: r.amount, label: r.platform || r.category,
        date: r.recorded_at, category: r.category,
      }));
      const expenseItems = expenseRecords.map(r => ({
        id: r.id, type: 'expense' as const,
        amount: r.effective_amount, label: r.description || r.category,
        date: r.recorded_at, category: r.category,
      }));
      return [...incomeItems, ...expenseItems]
        .sort((a, b) => new Date(b.date).getTime() - new Date(a.date).getTime())
        .slice(0, limit);
    };
  }, [incomeRecords, expenseRecords]);

  return { monthlySummary, yearlySummary, getRecentRecords };
}
```

`getRecentRecords` は収入と経費を統合した `MergedRecord` 型の配列を返します。ホーム画面の「最近の記録」セクションで、収入は緑、経費は赤で色分け表示するために `type` フィールドを持たせています。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- 収入・経費の型定義が完成し、TypeScript の恩恵を受けられる状態
- プラットフォーム15種類、経費カテゴリ12種類がデフォルトで用意されている
- recordStore がデータの取得・追加・削除・サマリー計算を一元管理している
- useRecords フックで画面から簡単にデータ操作ができる
- useSummary フックでダッシュボードに月次・年次の集計を表示できる

次の章では、クライアント側だけでは実現しにくい処理を Supabase Edge Functions で実装していきます。

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
