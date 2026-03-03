---
title: "Claude Codeで2日・10時間のiOSアプリ開発 — うまくいったこと・限界・コツを全部書く"
emoji: "🤖"
type: "idea"
topics: ["claudecode", "ai", "expo", "個人開発", "バイブコーディング"]
published: true
---

# Claude Codeで2日・10時間のiOSアプリ開発 — うまくいったこと・限界・コツを全部書く

**48ファイル、約4,000行のTypeScript。認証、CRUD、ダッシュボード、サブスク課金、Edge Function。**

これを2日・約10時間で書きました。外注見積もりなら数十万円の規模です。API費用は数千円。

種明かしをすると、**Claude Code**（Anthropic社のAIコーディングツール）を使ったバイブコーディングです。この記事では、iOSアプリ「ふくログ」の開発過程をすべて公開します。何をAIに任せて、何を自分でやったのか。バイブコーディングのリアルをお伝えします。

## この記事を読むべき人

- Claude CodeやGitHub CopilotなどのAIコーディングツールに興味がある
- 個人開発でアプリを作りたいが、時間が足りないと感じている
- バイブコーディングの実際の開発フローを知りたい
- Expo + Supabase + RevenueCatの技術スタックに興味がある

## 作ったアプリ「ふくログ」

**ふくログ**は、副業エンジニア向けの収入・経費管理アプリです。

主な機能はこちら。

- **認証**: メール / Google / Apple ログイン
- **収入・経費のCRUD**: 15種類のプラットフォーム、12種類の経費カテゴリに対応
- **20万円ダッシュボード**: 確定申告が必要かどうかをリアルタイム判定
- **月次レポート**: 月別・年間の収支サマリー
- **CSV出力**: 確定申告用データのエクスポート
- **サブスク課金**: RevenueCatで月額/年額のPro版課金

技術スタックは **Expo SDK 54 + Supabase + RevenueCat + TypeScript + Zustand + Zod** です。

最終的な規模はこうなりました。

| 項目 | 数値 |
|------|------|
| TypeScript ファイル数 | 48 |
| 総コード行数 | 約4,000行 |
| 画面数 | 11画面 |
| DBテーブル数 | 5テーブル |
| マイグレーションファイル | 10本 |
| Edge Function | 1本（RevenueCat Webhook） |

## Day 1: 要件定義 → 基盤構築 → 主要機能実装（約6時間）

### Claude Codeへの最初のプロンプト

最初にやったのは、アプリの全体像をClaude Codeに伝えることです。

```
副業エンジニア向けの収入・経費管理アプリを作りたい。
- Expo SDK 54 + Supabase + TypeScript
- 認証はメール/Google/Apple
- 収入と経費をCRUDで管理
- 副業所得が20万円を超えたら確定申告が必要なので、
  年間の所得をリアルタイムで表示するダッシュボードが欲しい
- 状態管理はZustand、バリデーションはZod
```

ポイントは、**技術スタックと「なぜ作るのか」の両方を伝えている**ことです。「20万円を超えたら確定申告が必要」というドメイン知識を最初に共有したことで、Claude Codeはダッシュボードの設計だけでなく、税計算ロジックや確定申告判定まで一貫した設計を提案してくれました。

このプロンプトから、Claude Codeはプロジェクトの全体構成を提案してくれました。

```
src/
├── components/     # UIコンポーネント（13ファイル）
│   ├── home/       # ホーム画面用（3ファイル）
│   ├── input/      # 入力UI（2ファイル）
│   └── ui/         # 汎用部品（8ファイル）
├── constants/      # 定数・設定（5ファイル）
├── hooks/          # カスタムフック（6ファイル）
├── services/       # 外部サービス連携（5ファイル）
│   └── supabase/   # Supabase API層
├── stores/         # Zustand ストア（3ファイル）
├── types/          # 型定義（4ファイル）
└── utils/          # ユーティリティ（6ファイル）
```

### Supabaseのテーブル設計もAIに任せた

「副業の収入と経費を管理するDBスキーマを設計して。RLSポリシー付きで」と伝えたら、5テーブル分のスキーマが一発で生成されました。

```sql
-- profiles: ユーザー情報 + サブスク状態
-- income_records: 収入記録（プラットフォーム別）
-- expense_records: 経費記録（按分率対応）
-- platforms: ユーザー別プラットフォーム設定
-- expense_categories: ユーザー別経費カテゴリ設定
```

特に良かったのは、新規ユーザー登録時にデフォルトのプラットフォーム（クラウドワークス、ランサーズ、Zennなど11個）と経費カテゴリ（PC・ガジェット、クラウドサービスなど12個）を自動挿入するトリガーまで提案してくれたことです。

```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO profiles (id) VALUES (NEW.id);
  INSERT INTO platforms (user_id, name, category, sort_order) VALUES
    (NEW.id, 'クラウドワークス', 'contract', 1),
    (NEW.id, 'ランサーズ', 'contract', 2),
    -- ... 全11プラットフォーム
  INSERT INTO expense_categories (user_id, name, icon, default_ratio, sort_order) VALUES
    (NEW.id, 'PC・ガジェット', 'laptop', 50, 1),
    (NEW.id, 'クラウドサービス', 'cloud', 100, 2),
    -- ... 全12カテゴリ（按分率のデフォルト値付き）
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

「ユーザーが初回ログイン後にゼロ状態で放置されない」というUX配慮がトリガーレベルで組み込まれていて、これは自分では後回しにしていたであろう部分です。

RLSポリシーも `auth.uid() = user_id` で適切に設定されていました。ここは自分でも確認しましたが、問題なしでした。

### 認証フローの実装（約3時間）

認証まわりはDay 1で最も時間がかかった部分です。メール認証はすぐできましたが、Google/AppleのOAuth連携は `expo-web-browser` と `expo-auth-session` の組み合わせで少しハマりました。

Claude Codeが生成した認証サービス層（`services/supabase/auth.ts`、約140行）は、以下の構成です。

```typescript
// OAuth認証の流れ
export async function signInWithGoogle() {
  const { data, error } = await supabase.auth.signInWithOAuth({
    provider: 'google',
    options: {
      redirectTo: DEEP_LINKS.authCallback,
      skipBrowserRedirect: true,
    },
  });
  if (error) throw error;

  if (data.url) {
    const result = await WebBrowser.openAuthSessionAsync(
      data.url,
      DEEP_LINKS.authCallback
    );
    if (result.type === 'success') {
      await handleAuthCallback(result.url);
    }
  }
}
```

`skipBrowserRedirect: true` を指定して、Expoの `WebBrowser.openAuthSessionAsync` でOAuthフローを制御する点がポイントです。コールバックURLのパース処理（URLハッシュからアクセストークンを抽出する `handleAuthCallback`）まで、Claude Codeが正しいパターンで生成してくれました。

認証の状態管理（`stores/authStore.ts`、約215行）はZustandで実装し、`onAuthStateChange` リスナーでセッション変更をリアルタイムに反映する設計にしました。ここは重複イベントの回避処理など、Claude Codeが生成したコードに自分で微調整を加えた部分です（詳しくは後述）。

### ダッシュボード + CRUDの実装

収入と経費のCRUD処理はシンプルでした。Supabase APIとZustandストアの接続（`stores/recordStore.ts`、約220行）を1ファイルにまとめ、月別・年間の集計処理もこの中に書きました。

```typescript
getYearlySummary: (year?: number): YearlySummaryResult => {
  const yearIncomes = state.incomeRecords.filter(r => r.tax_year === y);
  const yearExpenses = state.expenseRecords.filter(r => r.tax_year === y);

  const totalIncome = yearIncomes.reduce((sum, r) => sum + r.amount, 0);
  const totalExpense = yearExpenses.reduce((sum, r) => sum + r.effective_amount, 0);
  const netIncome = totalIncome - totalExpense;

  return {
    totalIncome, totalExpense, netIncome,
    progressPercent: Math.min((netIncome / 200000) * 100, 100),
    // ...
  };
},
```

経費の `effective_amount`（按分後の金額）を使って集計している点がミソです。ユーザーが入力した按分率（例: PC費用の50%を経費計上）が自動で反映されるので、20万円のプログレスバーには正しい「所得」が表示されます。

20万円ルールの判定ロジック（`utils/tax.ts`、59行）も、Claude Codeが正確に実装しました。

```typescript
export function judgeTaxFiling(params: TaxCalculationParams): TaxFilingResult {
  const netIncome = params.totalIncome - params.totalExpense;

  // 医療費控除・ふるさと納税で確定申告する場合
  if (params.hasMedicalDeduction || params.hasFurusatoTax) {
    return {
      required: true,
      reason: '医療費控除またはふるさと納税の申告をする場合、副業所得も含めて申告が必要です',
    };
  }

  // 2箇所以上から給与がある場合
  if (params.salarySourceCount >= 2) {
    return { required: true, reason: '2箇所以上から給与を受けている場合、確定申告が必要です' };
  }

  // 所得が20万円を超える場合
  if (netIncome > 200000) {
    return {
      required: true,
      reason: `副業所得が${netIncome.toLocaleString()}円（20万円超）のため、確定申告が必要です`,
    };
  }

  // 20万円以下
  return {
    required: false,
    reason: '所得税の確定申告は不要です',
    note: '住民税の申告は市区町村への届出が別途必要です（所得が1円以上ある場合）',
  };
}
```

医療費控除やふるさと納税で確定申告をする場合は20万円以下でも申告が必要、2箇所以上から給与がある場合のケースも網羅されています。税制のドメイン知識をプロンプトで伝えただけで、ここまで正確なロジックが出てきたのは驚きでした。

## Day 2: 課金実装 → 仕上げ → ビルド（約4時間）

### RevenueCat連携もAIが書いた

2日目の最大のタスクはサブスク課金の実装です。RevenueCat SDK連携（`services/revenueCat.ts`、約140行）は、初期化・権限チェック・購入・復元・リスナーの5機能をきれいに分離して生成されました。

```typescript
export async function purchaseMonthly(): Promise<boolean> {
  const sdk = await getPurchases();
  if (!sdk) return false;

  const offerings = await sdk.getOfferings();
  const monthlyPackage = offerings.current?.monthly;
  if (!monthlyPackage) return false;

  const { customerInfo } = await sdk.purchasePackage(monthlyPackage);
  return customerInfo.entitlements.active[REVENUE_CAT.entitlementId] !== undefined;
}
```

注目すべきは、Web環境でのクラッシュ回避設計です。

```typescript
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

`Platform.OS` のチェックとdynamic importを組み合わせることで、Expo Goでのデバッグ時にネイティブSDKがロードされずクラッシュしない設計になっています。この工夫をClaude Codeが自動で入れてくれたのは、かなり助かりました。

### Edge Function + Webhookの実装

RevenueCatのWebhookを受け取ってSupabaseの `profiles.subscription_tier` を更新するEdge Functionも、Claude Codeに一言伝えるだけで生成されました。

```typescript
// supabase/functions/revenuecat-webhook/index.ts
const PRO_EVENTS = new Set([
  'INITIAL_PURCHASE', 'RENEWAL', 'UNCANCELLATION',
  'NON_RENEWING_PURCHASE', 'PRODUCT_CHANGE',
]);
const FREE_EVENTS = new Set(['EXPIRATION']);
```

Webhook認証（Bearer Token）、イベント種別による分岐、Service Role KeyでのDB更新まで、約90行で完結しています。`Set` でイベント種別を管理するパターンも的確で、可読性とパフォーマンスのバランスが良い実装でした。

### UI/UXの調整

ダークテーマの配色、コンポーネントのスタイリングなどは `constants/theme.ts`（163行）に一元管理しています。

```typescript
export const colors = {
  dark: {
    bgPrimary: '#09090B',
    bgSecondary: '#18181B',
    accent: '#00D68F',
    accentLight: '#00E5A0',
    success: '#00D68F',
    warning: '#FFB800',
    error: '#FF4757',
    // ...
  },
} as const;
```

カラーパレット、スペーシング、タイポグラフィ、角丸、シャドウまで定数化されていて、全コンポーネントがこの定義を参照する設計です。アクセントカラー `#00D68F`（緑系）を基調とした、収入管理アプリらしいUIが仕上がりました。

このテーマファイル設計もClaude Codeの提案です。最初から `as const` でリテラル型にしてくれるので、TypeScriptの補完がばっちり効きます。

### EAS Buildでビルド

最終的に `eas build --platform ios` でビルドし、App Store Connectに提出しました。ビルドまわりはClaude Codeの守備範囲外で、`app.json` の設定やプロビジョニングプロファイルの管理は自分でやりました。

## Claude Codeを効率よく使うコツ

2日間で学んだ、実践的なコツを4つまとめます。

### 1. 「なぜ作るのか」をプロンプトに含める

技術的な要件だけでなく、ドメイン知識（「20万円超で確定申告が必要」など）を最初に伝えることで、AIが文脈を理解した設計を提案してくれます。「何を」ではなく「なぜ」を伝えるのが重要です。

### 2. 1機能ずつ完成させる

「認証 → CRUD → ダッシュボード → 課金」のように、1機能ずつ完成させてから次に進むのが重要です。一度に複数の機能を頼むと、依存関係が錯綜してバグが増えます。

Claude Codeはプロジェクト全体のコンテキストを理解した上でコードを生成するので、既存ファイルとの整合性を保ってくれます。途中から機能を追加するときも破綻しにくいのは、この仕組みのおかげです。

### 3. エラーはスタックトレースごと貼り付ける

ビルドエラーやランタイムエラーは、スタックトレースをそのままClaude Codeに貼り付けるだけで修正案を出してくれます。特にTypeScriptの型エラーは、ほぼ一発で解決しました。「このエラーが出ました」と自分の言葉で説明するより、そのまま貼るほうが精度が高いです。

### 4. CLAUDE.mdでプロジェクトルールを共有する

プロジェクトルートに `CLAUDE.md` を置いて、命名規則、ディレクトリ構成、使用ライブラリ、コーディング規約などを書いておくと、生成コードの一貫性が格段に上がります。人間のチーム開発でREADMEを整備するのと同じ感覚です。

## バイブコーディングの限界と注意点

正直に書きます。AIに全部任せれば完璧、というわけではありません。

### AIが書いたコードは必ずレビューする

Claude Codeが生成した認証ストアの `onAuthStateChange` リスナーには、初期実装で**重複イベントの問題**がありました。ログイン時に `SIGNED_IN` イベントが2回発火し、プロフィールの二重取得が起きていたのです。

```typescript
// 自分で追加した重複回避の処理
const currentState = get();
if (currentState.isAuthenticated && currentState.user?.id === session.user.id) {
  return; // 既に同じユーザーでログイン済みならスキップ
}
```

このような細かいバグは、実機で動かして初めて気づきます。AIが生成したコードを盲信せず、動作確認は丁寧にやりましょう。

### セキュリティ関連は人間が確認する

RLSポリシー、Webhook認証、Service Role Keyの管理など、セキュリティに関わる部分は自分の目で確認しました。AIが正しいパターンを出してくれることが多いですが、「正しいパターンを出すことが多い」と「常に安全である」は違います。セキュリティの最終判断は人間がするべきです。

### テストは自分で書く/確認する

今回のMVPではテストコードを書いていません（正直に言います）。ただし、リリース後にはユニットテスト（特に税計算ロジック）を追加しています。AIが生成したビジネスロジックが正しいかどうかは、テストで担保するのが理想です。

## 開発コスト・時間の内訳

### 開発時間の内訳（合計 約10時間）

| タスク | 時間 | AI活用度 |
|--------|------|---------|
| 要件定義 + プロンプト設計 | 1時間 | 低（人間主導） |
| 認証フロー（メール/Google/Apple） | 3時間 | 高（実装はAI、デバッグは人間） |
| DB設計 + CRUD + ダッシュボード | 2時間 | 高（設計・実装ともにAI） |
| RevenueCat + 課金フロー | 2時間 | 高（SDK連携はAI、動作確認は人間） |
| UI調整 + バグ修正 | 1.5時間 | 中（提案はAI、判断は人間） |
| ビルド + App Store提出 | 0.5時間 | 低（人間主導） |

### Claude Code API費用

2日間のClaude Code利用で、API費用は**数千円程度**でした。外注すれば数十万円はかかる規模のアプリを、この費用で作れるのは個人開発者にとって革命的です。

## まとめ -- AIは「手を動かす」を変えた

2日間で48ファイル・約4,000行のTypeScriptコードが書かれ、認証・CRUD・ダッシュボード・サブスク課金・Edge Functionまで揃ったアプリがApp Storeに並びました。

### Claude Codeがやってくれたこと

- **プロジェクト構成の設計**: ディレクトリ構成、ファイル分割の粒度
- **DB設計**: テーブル定義、RLS、トリガー、マイグレーション
- **ビジネスロジック**: 税計算、集計処理、CSV出力
- **外部サービス連携**: Supabase認証、RevenueCat SDK、Webhook
- **UI設計**: テーマ定数化、コンポーネント設計、ダークモード対応

### 自分がやったこと

- **要件定義と優先順位の判断**: 何を作って何を作らないか
- **ドメイン知識の提供**: 20万円ルール、按分の概念、確定申告の条件
- **セキュリティの確認**: RLS、認証フロー、Secret管理
- **実機での動作確認とバグ修正**: OAuth callback、重複イベント回避
- **App Store提出**: ビルド設定、審査対応

この2日間で確信したのは、**AIは「手を動かす」部分を大幅にショートカットしてくれるが、「何を作るか」「安全に作れているか」を判断するのは人間の仕事**だということです。

逆に言えば、この役割分担を理解していれば、個人開発のハードルは確実に下がります。「週末2日で作れるアプリ」の幅が、格段に広がりました。

---

:::message
**宣伝**: この記事で紹介した「ふくログ」の開発手法を詳しく解説するZenn本を執筆中です。

**[Expo + Supabase + RevenueCat で稼げるアプリを作る実践ガイド](https://zenn.dev/kenty/books/expo-supabase-revenuecat-guide)**
認証・DB設計・RLS・サブスク課金・App Store公開までを一冊で解説します。

**ふくログ** -- 副業エンジニアのための収入・経費管理アプリ
App Storeで近日公開予定です。公開時にお知らせするので、興味がある方はフォローお願いします。
:::
