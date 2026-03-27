---
title: "Claude Code Channels完全ガイド【2026年3月】DiscordとTelegramからAIにコードを書かせる"
emoji: "💬"
type: "tech"
topics: ["claudecode", "ai", "discord", "telegram"]
published: true
---

DiscordやTelegramから、ローカルのClaude Codeに直接指示を出せる。

それが2026年3月20日にAnthropicが発表した **Claude Code Channels** です。自前のボット実装もWebhookもなし。既存のDiscord/Telegramのチャット画面がそのままClaude Codeのターミナルになります。

Remote Controlに続く「常時稼働型開発」の進化形として、発表直後から使い込んでいます。セットアップ手順から実践的な活用シーン、Remote Controlとの使い分けまで解説します。

## この記事を読むべき人

- Claude Code Channelsの概要をざっくり把握したい方
- DiscordまたはTelegramのボット経由でClaude Codeを使いたい方
- CI失敗通知や監視アラートをClaude Codeに自動連携したい方
- Remote ControlとChannelsの違いを知りたい方
- push-based（プッシュ型）開発環境に興味がある方

## Claude Code Channels とは

### 一言で言うと

「ローカルで動くClaude Codeセッションを、Discord/Telegramのチャットから操作できる機能」です。

Remote Controlと同じく、**セッションはローカルマシン上で動いています**。ファイルシステム、git、MCPサーバー、環境変数——全てが手元のPCに存在したまま、メッセージングアプリがその「窓口」になります。

```
┌───────────────┐   メッセージ   ┌───────────────┐   stdio通信   ┌───────────────┐
│   Discord /   │ ◄──────────► │  Channelsの    │ ◄──────────► │  Claude Code  │
│   Telegram    │              │  MCPサーバー   │              │  セッション    │
└───────────────┘              └───────────────┘              └───────┬───────┘
                                                                      │
                                                               ローカル環境
                                                               ├── ファイルシステム
                                                               ├── 環境変数(.env)
                                                               ├── MCPサーバー
                                                               └── Git / npm 等
```

アーキテクチャはMCPベースのプラグイン構造になっています。Bunランタイム上でMCPサーバーがバックグラウンドで起動し、Discord/TelegramのbotがそのMCPサーバーとstdioで通信する仕組みです。

### Remote Controlとの違い

2026年2月発表のRemote Controlと同じ「ローカルを外から操作する」機能ですが、設計の思想が異なります。

| 機能 | Remote Control | Channels |
|------|---------------|----------|
| 接続方法 | Claude専用アプリ / ブラウザ | Discord / Telegram |
| 操作起点 | 人間が能動的に指示 | push通知に反応させることも可能 |
| CI/アラート連携 | 不可 | DiscordのCI失敗通知をそのままClaude Codeに流せる |
| プラグイン拡張 | 不可 | MCPプラグインで対応チャンネルを追加できる |
| UI | フルIDE風の操作感 | チャット形式 |
| セットアップ | QRコードをスキャンするだけ | ボットトークンの設定が必要 |

Remote Controlは「人間が能動的にモバイルからアクセスする」ユースケースに強く、Channelsは「外部システムのイベントをClaude Codeに届ける」ユースケースに強いです。

## 前提条件

| 項目 | 詳細 |
|------|------|
| Claude Code | v2.1.80 以上 |
| Claudeプラン | Pro ($20/月) または Max ($100〜200/月) |
| OS | macOS / Linux / WSL |
| Discord | Discordボットのトークン（Developer Portalで取得） |
| Telegram | BotFatherで作成したボットのAPIトークン |

### バージョン確認とアップデート

```bash
claude --version
```

v2.1.80未満の場合はアップデートします。

```bash
npm update -g @anthropic-ai/claude-code
```

## セットアップ手順（Discord編）

### ステップ1: Discord Developer PortalでBotを作成

1. https://discord.com/developers/applications にアクセス
2. 「New Application」でアプリを作成
3. 左メニュー「Bot」から「Add Bot」
4. 「Token」セクションの「Reset Token」でトークンを取得（コピーしておく）
5. 「Privileged Gateway Intents」で「Message Content Intent」をONにする
6. 「OAuth2 > URL Generator」でスコープ `bot` を選択し、権限に「Send Messages」「Read Messages/View Channels」を付与してサーバーに招待

### ステップ2: プラグインのインストールと設定

```bash
# Discordプラグインをインストール
claude channels install discord

# ボットトークンを設定
claude channels configure discord --token YOUR_BOT_TOKEN
```

### ステップ3: Channelsを有効にして起動

```bash
claude --channels
```

または既存セッションに追加する場合は、Claude Codeのプロンプトから:

```
/channels
```

起動するとターミナルに「Channels active: discord」と表示されます。

### ステップ4: ペアリング

BotをDMするかサーバーのチャンネルでメンションすると、ワンタイムコードが返ってきます。

```
ペアリングコードを入力してください: XXXX-XXXX
```

ターミナル上にコードを入力すると接続が完了します。以降はそのDMまたはチャンネルからClaude Codeに指示を送れます。

## セットアップ手順（Telegram編）

### ステップ1: BotFatherでボットを作成（約30秒）

1. TelegramでBotFather（@BotFather）にDM
2. `/newbot` と送信
3. ボット名とユーザーネームを入力（ユーザーネームは `bot` で終わる必要あり）
4. 表示されたAPIトークンをコピー

### ステップ2: プラグインのインストールと設定

```bash
# Telegramプラグインをインストール
claude channels install telegram

# ボットトークンを設定
claude channels configure telegram --token YOUR_BOT_TOKEN
```

### ステップ3: 起動とペアリング

```bash
claude --channels
```

作成したボットにTelegramでDMを送るとペアリングコードが返ってきます。ターミナルでコードを入力して接続完了です。

Telegramはメッセージ履歴APIが存在しないため、**セッションに接続してからのメッセージのみが届きます**。接続前のメッセージは受信できない点に注意してください。

## 使い方の実例

### 基本: チャットから普通に指示を送る

DiscordのDMやチャンネルからメッセージを送るだけです。

```
src/screens/HomeScreen.tsx を読んで、パフォーマンス上の問題があれば教えて
```

Claude Codeがローカルのファイルを読み込んで回答を返します。返答はそのままDiscord/Telegramのチャットに投稿されます。

### 応用1: CI失敗通知を直接Claudeに流す

GitHub ActionsやCircle CIの失敗通知がDiscordチャンネルに流れている場合、そのチャンネルにClaude Code Channelsを設定しておくと効果的です。

```
[GitHub Actions] Build failed: npm run test
Error: TypeError: Cannot read properties of undefined (reading 'map')
  at src/hooks/useIncomeList.ts:42
```

このような通知をClaude Codeが受け取ったとき、内容を読んで対応コマンドをいきなり実行するのではなく、まず原因の仮説を提示してくれます。承認後にファイルを修正・コミットします。

### 応用2: 外出先から開発を進める

外出前にPCでセッションを起動しておき、移動中にスマートフォンのDiscordから指示を出します。

```
前回のコミットからの変更をdiffで見せて。
レビューコメントがあれば日本語で出して
```

```
npm run lint を実行してエラーがあれば修正して
```

ターミナルコマンドの実行には承認が必要なので、スマートフォンから承認/拒否できます。

### 応用3: 監視アラートへの自動対応

Datadogや監視SaaSのアラートをDiscordチャンネルに転送している環境であれば、アラート文をそのままClaude Codeに届けられます。

```
[Alert] Response time P99 exceeded 3s for /api/income/list
```

Claude Codeがアラートを受け取り、対象エンドポイントのコードを読んでボトルネックの仮説を出す——というフローが可能になります。

## push-based開発とは何か

Remote Controlは「人間がモバイルから能動的に操作する」pull-based（プル型）でした。

Channelsが持つ最大の違いは **push-based（プッシュ型）** であることです。人間からの指示を待つのではなく、外部システムのイベント（CI失敗、監視アラート、誰かのメッセージ）が起点となってClaude Codeが動き出すことができます。

```
[pull-based / Remote Control]
人間 ──── スマホで操作 ───► Claude Code

[push-based / Channels]
CI失敗 ──── Discord通知 ───► Claude Code ──► 原因調査・修正案
監視Alert ─── Telegram通知 ──► Claude Code ──► ログ確認・対応提案
```

副業エンジニアとして本業を抱えている場合、「本業中に副業プロジェクトのCIが壊れた」というのはよくある状況です。ChannelsがDiscordのCI通知チャンネルを監視していれば、帰宅後に「こういうエラーが出ていたよ、こう修正した」という報告がすでにできている状態を作れます（もちろん実際の修正はコミット前に確認します）。

## 制限事項

### 現在の主な制限

| 制限 | 詳細 |
|------|------|
| ターミナル起動必須 | セッションを開いたままにする必要がある |
| 対応プラットフォーム | Discord / Telegram のみ（Slackは現在コミュニティ要望中） |
| Telegramの履歴 | セッション開始前のメッセージは受信不可 |
| 同時接続 | 複数のChannelを同時に有効化できるが、セッションは1つ |
| Research Preview | 2026年3月時点でMSXアカウント向けリサーチプレビュー |

### Discordセットアップの手間

Telegramと比べると、DiscordはDeveloper Portal操作・ボット権限設定・サーバー招待と工程が多いです。一度設定してしまえば以降は `claude --channels` の一発起動ですが、初回の敷居はTelegramの方が低いです。

### スリープ対策

外出中に使う場合はPCのスリープを防ぐ必要があります。macOSであれば:

```bash
# Channelsセッション中はスリープを防ぐ
caffeinate -i &

# 終了後
kill %1
```

## Remote ControlとChannelsの使い分け

どちらを使うべきかは、ユースケース次第です。

| シナリオ | 推奨 |
|---------|------|
| 外出先からモバイルで操作したい | Remote Control |
| CIの失敗通知に自動対応させたい | Channels |
| 監視アラートをClaudeに繋ぎたい | Channels |
| チームのDiscordチャンネルから操作したい | Channels |
| セットアップを最小限にしたい | Remote Control |
| Slack/WhatsAppから使いたい | どちらも現時点では非対応 |

筆者の運用は、**日常的な外出先からの指示はRemote Control、CI通知が流れるDiscordチャンネルへの接続にChannels**という使い分けです。両方同時に有効化することもできます。

## まとめ

Claude Code Channelsは、開発環境を「待ち受け型」に変えるアップデートです。

これまでClaude Codeは「人間が画面の前で入力する」ことが前提でした。ChannelsはDiscord/Telegramというプッシュ通知が飛び交うプラットフォームを接続口にすることで、**外部のイベントがClaude Codeを動かす**という仕組みを作れます。

2026年3月時点での主な制限:

- Pro/Maxプランのみ（Research Preview）
- 対応はDiscord/Telegramのみ（Slack等は今後）
- ターミナルセッションの起動が前提
- Telegramはメッセージ履歴が残らない

副業エンジニアとして個人開発を続ける上で、「本業中でもClaude Codeが動いている」状態は純粋に開発速度を上げます。Remote Controlと組み合わせて、常時稼働に近い環境を作る価値は十分にあります。

Remote Controlの詳細については[こちらの記事](/articles/claude-code-remote-control-guide)で解説しています。

---

:::message
**宣伝**: 筆者が開発した副業エンジニア向け収入・経費管理アプリ「**ふくログ**」

- 20万円ラインがプログレスバーで一目瞭然
- エンジニア向けの経費カテゴリ
- 無料で全機能使えます

App Storeで公開中です。
:::
