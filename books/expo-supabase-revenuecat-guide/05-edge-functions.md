---
title: "第5章: Supabase Edge Functions — サーバーサイドロジック"
free: false
---

# 第5章: Supabase Edge Functions — サーバーサイドロジック

<!-- TODO: 執筆予定 Week 5-6 -->
<!-- アウトライン:
  5.1 Edge Functionsとは
  5.2 年次サマリー計算関数
  5.3 Edge Functionsのデプロイ
  5.4 アプリからEdge Functionsを呼び出す
-->


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
