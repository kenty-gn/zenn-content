---
title: "Ghostty をTokyo Night ベースでネオン寄りにカスタマイズした設定全公開"
emoji: "👻"
type: "tech"
topics: ["ghostty", "terminal", "mac", "dotfiles", "開発環境"]
published: true
---

ターミナルをじっくりカスタマイズするのは、地味に楽しい時間です。

最近 Ghostty に乗り換えて、Tokyo Night テーマをベースにしつつ「もう少しネオン寄りにしたい」と思い、カラーパレットや透過・ブラーなどを調整しました。この記事ではその設定ファイルを全公開しつつ、各設定の意図を解説します。

## Ghostty とは

[Ghostty](https://ghostty.org/) は Mitchell Hashimoto（HashiCorp 創業者）が開発したターミナルエミュレータです。Zig 製で、GPU レンダリングによる高パフォーマンスと、シンプルな設定ファイル形式が特徴です。

macOS と Linux に対応しており、2024年末にパブリックリリースされました。iTerm2 や Warp を使っていた方でも、設定ファイル一枚で乗り換えられる手軽さがあります。

## 設定ファイルの場所

macOS の場合、設定ファイルはこちらに置きます。

```
~/.config/ghostty/config
```

ファイルが存在しない場合は新規作成するだけで OK です。TOML ライクなシンプルな形式で、`key = value` の形式で設定を書きます。

## 完成した設定ファイル

まず全体を載せます。以降のセクションでパートごとに解説します。

```
# Ghostty Config - Tokyo Night Custom

# Base theme (overrides below take priority)
theme = dark:TokyoNight Night,light:TokyoNight Day

# Color overrides - deeper background, neon accents
background = #13141c
foreground = #c8d3f5
cursor-color = #7aa2f7
cursor-text = #13141c
selection-background = #3d59a1
selection-foreground = #e0e8ff

# Custom palette - punchier colors
palette = 0=#1a1b2e
palette = 1=#ff637d
palette = 2=#a6e36e
palette = 3=#ffc66d
palette = 4=#7aa2f7
palette = 5=#c792ea
palette = 6=#59d8e6
palette = 7=#b4befe
palette = 8=#505677
palette = 9=#ff8fa0
palette = 10=#b5f383
palette = 11=#ffd68a
palette = 12=#89b4fa
palette = 13=#d4a8ff
palette = 14=#74e4f0
palette = 15=#dce4ff

# Font
font-family = "JetBrainsMono NFM"
font-size = 16
adjust-cell-height = 20%

# Window
window-padding-x = 16
window-padding-y = 12
window-padding-balance = true
window-decoration = true
window-save-state = always
window-colorspace = display-p3
macos-titlebar-style = tabs

# Transparency - dark + blur for depth
background-opacity = 0.85
background-blur = 80

# Cursor
cursor-style = bar
cursor-style-blink = false

# Scrollback
mouse-scroll-multiplier = 2
copy-on-select = clipboard
```

## 設定の解説

### テーマのベースと上書き

```
theme = dark:TokyoNight Night,light:TokyoNight Day
```

Ghostty の組み込みテーマを読み込みつつ、後続の設定で個別の値を上書きできます。ライト/ダークそれぞれ指定しておくと、macOS のシステム外観設定に追従して自動で切り替わります。

ベーステーマを指定してから個別の色を調整する、というのが Ghostty のカスタマイズの基本的な流れです。テーマのすべての値が一度読み込まれ、その後に書いた設定が上書きします。

### カラーオーバーライド

```
background = #13141c
foreground = #c8d3f5
cursor-color = #7aa2f7
cursor-text = #13141c
selection-background = #3d59a1
selection-foreground = #e0e8ff
```

Tokyo Night のデフォルト背景（`#1a1b2e`）より少し深い `#13141c` にしました。背景が暗いほどテキストのコントラストが上がり、長時間作業しても目が疲れにくい印象です。

カーソルは青系の `#7aa2f7` にしています。Tokyo Night の代表的なアクセントカラーで、コードと馴染みつつ存在感があります。

### カラーパレット（0〜15番）

```
palette = 0=#1a1b2e
...
palette = 8=#505677
...
```

ターミナルの 16 色パレットです。0〜7 番が通常色、8〜15 番がブライト（明るい）色です。

多くのカラーテーマは 0〜7 番だけ調整していますが、8〜15 番も合わせると統一感が出ます。たとえば `ls` コマンドのディレクトリ表示や、vim のシンタックスハイライトで 8〜15 番が使われる場面があります。

このカスタムパレットは Tokyo Night の雰囲気を維持しつつ、全体的に彩度を上げてネオン寄りの印象にしています。赤（1/9番）と緑（2/10番）の視認性を特に意識しました。

### フォント設定

```
font-family = "JetBrainsMono NFM"
font-size = 16
adjust-cell-height = 20%
```

`JetBrainsMono NFM` は JetBrains Mono の Nerd Font 版です。Nerd Font を使うことで、`ls` の代替である `eza` やプロンプトカスタマイズツール（Starship など）のアイコンが文字化けせず表示されます。

`adjust-cell-height = 20%` でセルの縦幅を広げています。デフォルトのままだと行間が詰まった印象になるので、少し広げると読みやすくなります。

インストールは [Nerd Fonts の公式サイト](https://www.nerdfonts.com/) か Homebrew から可能です。

```bash
brew install --cask font-jetbrains-mono-nerd-font
```

### ウィンドウ設定

```
window-padding-x = 16
window-padding-y = 12
window-padding-balance = true
window-decoration = true
window-save-state = always
window-colorspace = display-p3
macos-titlebar-style = tabs
```

いくつかポイントを絞って説明します。

**`window-padding-balance = true`**
内側の余白をウィンドウ全体で均等にします。スクロールバーの幅などを自動で考慮してくれるので、見た目がすっきりします。

**`window-save-state = always`**
ウィンドウの位置とサイズを終了時に保存します。次回起動時に同じ状態から始められるので、再配置の手間がなくなります。

**`window-colorspace = display-p3`**
Display P3 色域を有効にします。MacBook Pro や Pro Display XDR など P3 対応ディスプレイを使っている場合、カラーパレットの色をより鮮やかに表示できます。対応ディスプレイがなければ実質的な差はありません。

**`macos-titlebar-style = tabs`**
タイトルバーをタブスタイルにすることで、ウィンドウ上部がすっきりします。タブを使わない場合でも、ウィンドウの見た目が洗練されます。

### 透過とブラー

```
background-opacity = 0.85
background-blur = 80
```

個人的にこの設定がいちばん気に入っています。

`background-opacity` で背景を 85% の不透明度にして、`background-blur` でぼかしを強めにかけています。背景が完全に透明でなく、ぼけたガラス越しに見えるような奥行き感が出ます。

`background-blur` の値は 0〜100 で指定します。80 はかなり強いぼかしで、背景の内容がほとんど判別できない状態になります。透過を使いつつも背景のノイズに集中力を奪われたくない場合に有効です。

デフォルト（`background-opacity = 1.0`, ブラーなし）との違いは、実際に試してみるとすぐわかります。

### カーソル設定

```
cursor-style = bar
cursor-style-blink = false
```

カーソルをブロック型からバー型（縦棒）に変更しました。`cursor-style-blink = false` でブリンクを無効にしています。

点滅するカーソルは注意を引きやすい反面、長時間の作業で目障りに感じることがあります。静止したバー型は見つけやすく、かつ落ち着いた印象です。

### スクロール設定

```
mouse-scroll-multiplier = 2
copy-on-select = clipboard
```

`mouse-scroll-multiplier = 2` でスクロール速度を 2 倍にしています。デフォルトのままだと長いログを追うときに遅く感じるため、少し速くしています。

`copy-on-select = clipboard` はテキストを選択すると自動でクリップボードにコピーします。Linux のマウス中ボタンペーストに慣れた方には自然な挙動です。`cmd + c` を省略できるので地味に便利です。

## まとめ

設定のポイントをまとめます。

| 設定 | 効果 |
| --- | --- |
| `theme` でベース読み込み + 個別上書き | テーマの恩恵を受けつつ細かく調整できる |
| palette 0〜15 すべてを調整 | 統一感のあるカラーリングになる |
| `background-opacity` + `background-blur` | 奥行き感のある背景になる |
| `macos-titlebar-style = tabs` | タイトルバーがすっきりする |
| Nerd Font 使用 | アイコンが文字化けしない |
| `cursor-style = bar` + ブリンク無効 | モダンで目に優しい |

Ghostty は設定ファイルを保存した瞬間にホットリロードされます。変更 → 保存 → 即反映のサイクルが速いので、色の微調整も気軽にできます。

最初はデフォルトのままでも十分綺麗ですが、一度自分好みに調整し始めると止まらなくなります。

---

副業・個人開発の収入・経費管理には、筆者が開発した iOS アプリ「**ふくログ**」をご利用ください。Claude Code を使って2日で作りました。

https://apps.apple.com/app/id6743806956
