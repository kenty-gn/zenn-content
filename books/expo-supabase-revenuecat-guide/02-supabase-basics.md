---
title: "第2章: Supabaseの基礎 — DB設計とRLS"
free: true
---

# 第2章: Supabaseの基礎 --- DB設計とRLS

この章では、ふくログのバックエンドとなる Supabase のプロジェクトを作成し、テーブル設計から行レベルセキュリティ（RLS）の設定、そしてアプリからの接続までを一気に進めます。「なぜこのテーブル構成にするのか」「RLS がないと何が起こるのか」を理解しながら、コピペで使える SQL を掲載していきます。

## 2.1 Supabaseプロジェクトの作成

[Supabase のダッシュボード](https://supabase.com/dashboard) にアクセスし、**New Project** をクリックします。

1. **Organization** を選択（初回はデフォルトの Personal org でOK）
2. **Project name** に `side-income-tracker` と入力
3. **Database Password** を設定（後で使うのでメモしておく）
4. **Region** は `Northeast Asia (Tokyo)` を選択（日本からのレイテンシが最小）
5. **Create new project** をクリック

プロジェクトが作成されたら、**Settings > API** から以下の2つの値を取得し、第1章で作成した `.env` ファイルに貼り付けます。

- **Project URL** → `EXPO_PUBLIC_SUPABASE_URL`
- **anon / public key** → `EXPO_PUBLIC_SUPABASE_ANON_KEY`

:::message
**service_role key** は管理者権限を持つキーです。クライアントアプリには絶対に含めないでください。Edge Functions（第5章・第7章）でのみ使います。
:::

## 2.2 テーブル設計

ふくログでは、以下の4つのテーブルを使います。

| テーブル名 | 役割 |
|---|---|
| `profiles` | ユーザープロフィール（認証時に自動作成） |
| `income_records` | 収入記録 |
| `expense_records` | 経費記録 |
| `platforms` | 収入プラットフォーム（ユーザーごとにカスタマイズ可能） |

### profiles テーブル

Supabase Auth でユーザーが作成されると `auth.users` テーブルにレコードが挿入されますが、アプリ固有のデータ（表示名、月間目標、サブスクリプション状態など）は別テーブルで管理します。

```sql
-- profiles テーブル
create table public.profiles (
  id uuid references auth.users(id) on delete cascade primary key,
  display_name text,
  employment_type text default 'employee' check (employment_type in ('employee', 'freelance', 'unknown')),
  salary_source_count integer default 1,
  monthly_goal integer default 200000,
  has_medical_deduction boolean default false,
  has_furusato_tax boolean default false,
  onboarding_completed boolean default false,
  subscription_tier text default 'free' check (subscription_tier in ('free', 'pro')),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);
```

`id` カラムが `auth.users(id)` を参照している点がポイントです。これにより、Supabase Auth のユーザーと 1:1 で紐づきます。`on delete cascade` を付けているので、ユーザーが削除されるとプロフィールも自動的に削除されます。

新規ユーザー登録時にプロフィールを自動作成するトリガーも設定しておきます。

```sql
-- 新規ユーザー登録時にプロフィールを自動作成するトリガー
create or replace function public.handle_new_user()
returns trigger as $$
begin
  insert into public.profiles (id, display_name)
  values (new.id, new.raw_user_meta_data->>'full_name');
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

### income_records テーブル

```sql
-- 収入記録テーブル
create table public.income_records (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade not null,
  amount integer not null check (amount > 0),
  category text default 'other',
  platform text not null,
  description text,
  recorded_at date not null,
  tax_year integer not null,
  has_withholding_tax boolean default false,
  withholding_tax_amount integer default 0,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- インデックス（月別・年別の検索を高速化）
create index idx_income_user_date on income_records(user_id, recorded_at);
create index idx_income_user_tax_year on income_records(user_id, tax_year);
```

設計上の判断をいくつか補足します。

**`amount` を integer にしている理由**: 金額を円単位の整数で管理することで、浮動小数点の丸め誤差を回避します。確定申告では1円単位の正確性が求められるため、`numeric` や `decimal` ではなく最もシンプルな `integer` を選びました。

**`tax_year` カラムを持つ理由**: 12月の売上が翌年1月に入金されるケースがあり、`recorded_at` の年と税年度が一致しないことがあります。`tax_year` を独立カラムとして持つことで、確定申告時に正確な年度集計ができます。

**`has_withholding_tax` と `withholding_tax_amount`**: クラウドソーシングや原稿料では源泉徴収されることがあります。確定申告で還付を受けるために、源泉徴収額を記録できるようにしています。

### expense_records テーブル

```sql
-- 経費記録テーブル
create table public.expense_records (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users(id) on delete cascade not null,
  amount integer not null check (amount > 0),
  category text not null,
  business_ratio integer default 100 check (business_ratio between 0 and 100),
  effective_amount integer not null check (effective_amount >= 0),
  description text,
  recorded_at date not null,
  tax_year integer not null,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- インデックス
create index idx_expense_user_date on expense_records(user_id, recorded_at);
create index idx_expense_user_tax_year on expense_records(user_id, tax_year);
```

経費テーブルの最も重要な設計は **按分（あんぶん）** の扱いです。

- `amount`: 実際に支払った金額（全額）
- `business_ratio`: 事業使用割合（0〜100%）
- `effective_amount`: 経費として計上する金額（`amount * business_ratio / 100`）

たとえば10万円のPCを購入し、事業使用が50%の場合、`amount = 100000`、`business_ratio = 50`、`effective_amount = 50000` となります。按分前後の金額を両方保存しておくことで、後から按分率を変更した場合の追跡が容易になります。

## 2.3 RLS（Row Level Security）とは

RLS は PostgreSQL の機能で、**テーブルの各行に対してアクセス制御ルールを設定**できる仕組みです。

RLS を設定しないとどうなるか考えてみましょう。Supabase の `anon` キーはクライアントに含まれるため、誰でもAPIを叩くことができます。RLS が無効だと、以下のようなクエリで **全ユーザーの収入データが取得できてしまいます**。

```typescript
// RLSなし → 全員のデータが見えてしまう！
const { data } = await supabase.from('income_records').select('*');
// → 他のユーザーの収入データも含まれる
```

RLS を有効にすると、ポリシーで許可された行だけがクエリ結果に含まれます。「自分のデータだけ見える」を **データベースレベルで強制** できるため、クライアント側でフィルタリングを忘れても安全です。

## 2.4 RLSポリシーの実装

各テーブルに対して、SELECT / INSERT / UPDATE / DELETE のポリシーを設定します。

```sql
-- ========================================
-- profiles テーブルの RLS
-- ========================================
alter table public.profiles enable row level security;

-- 自分のプロフィールだけ読める
create policy "Users can read own profile"
  on profiles for select
  using (auth.uid() = id);

-- 自分のプロフィールだけ更新できる
create policy "Users can update own profile"
  on profiles for update
  using (auth.uid() = id);

-- ========================================
-- income_records テーブルの RLS
-- ========================================
alter table public.income_records enable row level security;

-- 自分の収入記録だけ読める
create policy "Users can read own income"
  on income_records for select
  using (auth.uid() = user_id);

-- 自分の収入記録だけ作成できる
create policy "Users can insert own income"
  on income_records for insert
  with check (auth.uid() = user_id);

-- 自分の収入記録だけ更新できる
create policy "Users can update own income"
  on income_records for update
  using (auth.uid() = user_id);

-- 自分の収入記録だけ削除できる
create policy "Users can delete own income"
  on income_records for delete
  using (auth.uid() = user_id);

-- ========================================
-- expense_records テーブルの RLS
-- ========================================
alter table public.expense_records enable row level security;

create policy "Users can read own expenses"
  on expense_records for select
  using (auth.uid() = user_id);

create policy "Users can insert own expenses"
  on expense_records for insert
  with check (auth.uid() = user_id);

create policy "Users can update own expenses"
  on expense_records for update
  using (auth.uid() = user_id);

create policy "Users can delete own expenses"
  on expense_records for delete
  using (auth.uid() = user_id);
```

`auth.uid()` は Supabase が提供する関数で、**現在認証されているユーザーの ID** を返します。これを各行の `user_id` （profiles の場合は `id`）と比較することで、「自分のデータだけ操作できる」というルールをデータベースレベルで保証します。

:::message
ポリシーの `using` と `with check` の違い:
- **`using`**: SELECT / UPDATE / DELETE で「どの行が見えるか」を制御
- **`with check`**: INSERT / UPDATE で「挿入・更新後のデータが条件を満たすか」を検証

INSERT には `with check` を使います。「自分の `user_id` でしかレコードを作れない」ことを保証するためです。
:::

## 2.5 CRUD操作の実装

テーブルとRLSの設定ができたら、アプリからデータを操作するサービス層を実装します。第1章で初期化した Supabase クライアントを使って、収入と経費の CRUD 関数を作ります。

### 収入サービス（income.ts）

```typescript
import { supabase } from './client';

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

/** 収入を記録する */
export async function createIncome(data: CreateIncomeData) {
  const { data: record, error } = await supabase
    .from('income_records')
    .insert(data)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/** 月別の収入を取得する */
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
```

`.insert(data).select().single()` のチェーンに注目してください。`.select()` を付けることで、挿入されたレコードがレスポンスに含まれます。`.single()` は結果が1行であることを保証し、型推論を効かせやすくします。

月別取得では `gte`（以上）と `lt`（未満）を使って日付範囲をフィルタリングしています。`between` ではなく `gte` + `lt` を使う理由は、月末の境界を正確に扱うためです。「3月1日以上、4月1日未満」とすることで、3月31日のデータも確実に含まれます。

### 経費サービス（expense.ts）

```typescript
import { supabase } from './client';

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

/** 経費を記録する */
export async function createExpense(data: CreateExpenseData) {
  const { data: record, error } = await supabase
    .from('expense_records')
    .insert(data)
    .select()
    .single();
  if (error) throw error;
  return record;
}

/** 経費記録を削除する */
export async function deleteExpense(id: string) {
  const { error } = await supabase
    .from('expense_records')
    .delete()
    .eq('id', id);
  if (error) throw error;
}
```

削除では `.eq('id', id)` だけを指定していますが、RLS が有効なので他のユーザーのレコードを削除しようとしても自動的にブロックされます。アプリ側で `user_id` のチェックを忘れても安全、というのが RLS の大きなメリットです。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- Supabase プロジェクトが作成され、Project URL と anon key を取得した
- `profiles`、`income_records`、`expense_records` テーブルが作成されている
- 各テーブルに RLS が有効化され、「自分のデータだけ操作できる」ポリシーが設定されている
- 新規ユーザー登録時にプロフィールが自動作成されるトリガーが設定されている
- アプリからデータを操作する CRUD 関数が実装されている

Supabase ダッシュボードの **Table Editor** で、テーブルが正しく作成されていることを確認してみてください。

次の章では、この Supabase Auth を使った認証フロー（ログイン・サインアップ・OAuth）を実装していきます。

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
