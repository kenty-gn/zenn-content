---
title: "第2章: Supabaseの基礎 — DB設計とRLS"
free: true
---

# 第2章: Supabaseの基礎 — DB設計とRLS

<!-- TODO: 執筆予定 Week 2-3 -->
<!-- アウトライン:
  2.1 Supabaseプロジェクトの作成
  2.2 テーブル設計（income_records, expense_records等）
  2.3 RLS（Row Level Security）とは
  2.4 RLSポリシーの実装
  2.5 Supabaseクライアントの初期化（ExpoSecureStoreAdapter）
-->


## ソースコード

### `services/supabase/income.ts`

```typescript
import { supabase } from './client';

/**
 * 収入記録の作成データ
 */
export interface CreateIncomeData {
  user_id: string;
  amount: number;
  category?: string;
  platform: string;
  description?: string;
  recorded_at: string; // YYYY-MM-DD
  tax_year?: number;
  has_withholding_tax?: boolean;
  withholding_tax_amount?: number;
}

/**
 * 収入記録の更新データ
 */
export interface UpdateIncomeData {
  amount?: number;
  platform?: string;
  description?: string;
  recorded_at?: string;
  tax_year?: number;
  has_withholding_tax?: boolean;
  withholding_tax_amount?: number;
}

/**
 * 収入を記録する
 */
export async function createIncome(data: CreateIncomeData) {
  const { data: record, error } = await supabase
    .from('income_records')
    .insert(data)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/**
 * 月別の収入を取得する
 */
export async function getIncomeByMonth(userId: string, year: number, month: number) {
  const startDate = `${year}-${String(month).padStart(2, '0')}-01`;
  const endDate = month === 12
    ? `${year + 1}-01-01`
    : `${year}-${String(month + 1).padStart(2, '0')}-01`;

  const { data, error } = await supabase
    .from('income_records')
    .select('*')
    .eq('user_id', userId)
    .gte('recorded_at', startDate)
    .lt('recorded_at', endDate)
    .order('recorded_at', { ascending: false });
  if (error) throw error;
  return data;
}

/**
 * 年間の収入を取得する
 */
export async function getIncomeByYear(userId: string, year: number) {
  const { data, error } = await supabase
    .from('income_records')
    .select('*')
    .eq('user_id', userId)
    .eq('tax_year', year)
    .order('recorded_at', { ascending: false });
  if (error) throw error;
  return data;
}

/**
 * 収入記録を更新する
 */
export async function updateIncome(id: string, data: UpdateIncomeData) {
  const { data: record, error } = await supabase
    .from('income_records')
    .update(data)
    .eq('id', id)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/**
 * 収入記録を削除する
 */
export async function deleteIncome(id: string) {
  const { error } = await supabase
    .from('income_records')
    .delete()
    .eq('id', id);
  if (error) throw error;
}
```

### `services/supabase/expense.ts`

```typescript
import { supabase } from './client';

/**
 * 経費記録の作成データ
 */
export interface CreateExpenseData {
  user_id: string;
  amount: number;
  category: string;
  business_ratio?: number;
  effective_amount: number;
  description?: string;
  recorded_at: string; // YYYY-MM-DD
  tax_year?: number;
}

/**
 * 経費記録の更新データ
 */
export interface UpdateExpenseData {
  amount?: number;
  category?: string;
  business_ratio?: number;
  effective_amount?: number;
  description?: string;
  recorded_at?: string;
  tax_year?: number;
}

/**
 * 経費を記録する
 */
export async function createExpense(data: CreateExpenseData) {
  const { data: record, error } = await supabase
    .from('expense_records')
    .insert(data)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/**
 * 月別の経費を取得する
 */
export async function getExpenseByMonth(userId: string, year: number, month: number) {
  const startDate = `${year}-${String(month).padStart(2, '0')}-01`;
  const endDate = month === 12
    ? `${year + 1}-01-01`
    : `${year}-${String(month + 1).padStart(2, '0')}-01`;

  const { data, error } = await supabase
    .from('expense_records')
    .select('*')
    .eq('user_id', userId)
    .gte('recorded_at', startDate)
    .lt('recorded_at', endDate)
    .order('recorded_at', { ascending: false });
  if (error) throw error;
  return data;
}

/**
 * 年間の経費を取得する
 */
export async function getExpenseByYear(userId: string, year: number) {
  const { data, error } = await supabase
    .from('expense_records')
    .select('*')
    .eq('user_id', userId)
    .eq('tax_year', year)
    .order('recorded_at', { ascending: false });
  if (error) throw error;
  return data;
}

/**
 * 経費記録を更新する
 */
export async function updateExpense(id: string, data: UpdateExpenseData) {
  const { data: record, error } = await supabase
    .from('expense_records')
    .update(data)
    .eq('id', id)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/**
 * 経費記録を削除する
 */
export async function deleteExpense(id: string) {
  const { error } = await supabase
    .from('expense_records')
    .delete()
    .eq('id', id);
  if (error) throw error;
}
```
