---
title: "第8章: App Storeに公開する — EAS Build & Submit"
free: false
---

# 第8章: App Storeに公開する --- EAS Build & Submit

この章では、完成したふくログアプリを EAS（Expo Application Services）を使ってビルドし、App Store と Google Play に提出するまでの手順を解説します。証明書管理、ビルドプロファイル、審査のポイントまで、公開に必要な知識を一通りカバーします。

## 8.1 EAS Buildの準備

### EAS CLI のインストール

```bash
npm install -g eas-cli
eas login
```

### eas.json の設定

`eas.json` はビルドプロファイルの設定ファイルです。ふくログでは3つのプロファイルを使い分けます。

```json
{
  "cli": {
    "version": ">= 15.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "autoIncrement": true,
      "android": {
        "buildType": "app-bundle"
      },
      "ios": {
        "autoIncrement": true
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./google-services-key.json",
        "track": "production"
      }
    }
  }
}
```

各プロファイルの用途を整理しておきます。

| プロファイル | 用途 | 配布方法 |
|---|---|---|
| `development` | 開発用ビルド。`expo-dev-client` が含まれ、ホットリロードが使える | 内部配布（TestFlight / APK直接配布） |
| `preview` | QA・テスト用。本番に近い設定だが内部配布のみ | 内部配布 |
| `production` | App Store / Google Play への提出用 | ストア配布 |

**`appVersionSource: "remote"`** を設定すると、EAS サーバーでビルド番号を管理します。ローカルの `app.json` のバージョンを手動で更新し忘れる心配がなくなります。

**`autoIncrement: true`** を production プロファイルに設定すると、ビルドのたびに自動でビルド番号がインクリメントされます。App Store ではビルド番号が重複するとアップロードが拒否されるため、この設定は必須です。

### 環境変数の設定

本番ビルドで使用するシークレットは、EAS の環境変数機能で管理します。

```bash
eas secret:create --name EXPO_PUBLIC_SUPABASE_URL --value "https://your-project.supabase.co" --scope project
eas secret:create --name EXPO_PUBLIC_SUPABASE_ANON_KEY --value "your-anon-key" --scope project
eas secret:create --name EXPO_PUBLIC_REVENUECAT_APPLE_API_KEY --value "your-api-key" --scope project
```

:::message
`eas secret` で設定した環境変数は、EAS Build のビルドマシン上でのみ利用可能です。ローカル開発では引き続き `.env` ファイルを使います。
:::

## 8.2 app.json の本番設定

ビルドの前に `app.json` の設定を最終確認します。

```json
{
  "expo": {
    "name": "ふくログ",
    "slug": "side-income-tracker",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/icon.png",
    "scheme": "sideincometracker",
    "userInterfaceStyle": "dark",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/images/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#09090B"
    },
    "ios": {
      "supportsTablet": false,
      "bundleIdentifier": "com.fukulog.app",
      "infoPlist": {
        "ITSAppUsesNonExemptEncryption": false
      }
    },
    "android": {
      "package": "com.fukulog.app",
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/adaptive-icon.png",
        "backgroundColor": "#09090B"
      },
      "permissions": [
        "INTERNET",
        "com.android.vending.BILLING"
      ]
    },
    "plugins": [
      "expo-router",
      "expo-secure-store",
      "expo-web-browser"
    ]
  }
}
```

いくつかの設定をピックアップして解説します。

**`ITSAppUsesNonExemptEncryption: false`**: アプリが暗号化通信（HTTPS）以外の暗号化を使用していないことを宣言します。これを設定しないと、App Store Connect にアップロードするたびに暗号化の輸出規制に関する質問に回答する必要があります。Supabase との HTTPS 通信は「exempt（免除対象）」なので `false` で問題ありません。

**`com.android.vending.BILLING`**: Android で IAP（アプリ内課金）を使用するために必要なパーミッションです。RevenueCat を使う場合は必ず追加してください。

**`newArchEnabled: true`**: React Native の新アーキテクチャ（Fabric + TurboModules）を有効にします。パフォーマンスが向上しますが、一部のライブラリが未対応の場合があるので、本番ビルドの前にテストしておきましょう。

## 8.3 iOSビルド

```bash
eas build --platform ios --profile production
```

初回ビルド時に EAS が以下を自動処理します。

1. **Apple Developer アカウントへのログイン**
2. **Distribution Certificate の作成**（アプリの署名に使う証明書）
3. **Provisioning Profile の作成**（デバイスとアプリを紐づけるプロファイル）

EAS に証明書管理を任せると、ローカルマシンに証明書をダウンロードする必要がなくなります。チーム開発でも証明書の共有問題が発生しません。

ビルドが完了すると、EAS ダッシュボードに `.ipa` ファイルが生成されます。

## 8.4 Androidビルド

```bash
eas build --platform android --profile production
```

Android では Keystore（署名用の鍵）が自動生成されます。production プロファイルでは `buildType: "app-bundle"` を指定しているため、`.aab`（Android App Bundle）形式で出力されます。Google Play は `.aab` を推奨しており、デバイスごとに最適化された APK が自動生成されます。

## 8.5 App Store Connectの設定

ビルドが完了したら、App Store Connect でアプリ情報を入力します。

### 基本情報

| 項目 | 内容 | 備考 |
|---|---|---|
| アプリ名 | ふくログ | 30文字以内 |
| サブタイトル | 副業の収入・経費をかんたん管理 | 30文字以内。検索にも影響する |
| カテゴリ | ファイナンス | Primary category |
| 年齢制限 | 4+ | 課金機能があっても4+で可 |
| プライバシーポリシー | URL を記載 | 必須。アカウント作成機能があるアプリは特に必要 |

### 審査への提出

```bash
# iOS の場合
eas submit --platform ios

# Android の場合
eas submit --platform android
```

`eas submit` を実行すると、EAS ダッシュボードの最新ビルドが自動的に App Store Connect / Google Play Console にアップロードされます。

## 8.6 審査で落ちやすいポイントと対策

App Store の審査でリジェクトされやすいポイントと、その対策をまとめます。

### 1. ログイン必須アプリにはゲストアクセスまたはデモアカウントが必要

審査チームがアプリの機能を確認するために、ログイン情報を提供する必要があります。「審査情報」セクションにデモアカウントのメールアドレスとパスワードを記入してください。

### 2. サブスクリプションの説明が不十分

サブスクリプションを含むアプリでは、以下の情報をペイウォール画面に明記する必要があります。

- 価格と課金期間（例: 月額490円）
- 無料トライアル期間がある場合はその期間
- 自動更新される旨の説明
- 利用規約とプライバシーポリシーへのリンク
- 「購入の復元」ボタン

### 3. プライバシーポリシーが必須

アカウント作成機能があるアプリは、プライバシーポリシーの設定が必須です。収集するデータの種類、利用目的、第三者との共有について記載する必要があります。

### 4. アカウント削除機能

2022年6月以降、アカウント作成機能を持つアプリは **アカウント削除機能** の提供が必須になっています。設定画面に「アカウントを削除」ボタンを用意し、Supabase Auth のユーザー削除を実行するようにしてください。

## 8.7 リリース後にやること

### バージョン管理

アプリのバージョンは [Semantic Versioning](https://semver.org/) に従います。

- **パッチ（1.0.x）**: バグ修正
- **マイナー（1.x.0）**: 新機能の追加（後方互換あり）
- **メジャー（x.0.0）**: 大規模な変更（後方互換なし）

`eas.json` で `autoIncrement` を設定しているため、ビルド番号は自動管理されます。`app.json` の `version` フィールドは手動で更新してください。

### OTA（Over-the-Air）アップデート

JavaScript バンドルの変更（UI修正、バグ修正など）は、EAS Update を使えばストア審査なしでユーザーに配信できます。

```bash
eas update --branch production --message "バグ修正: 日付表示の不具合を修正"
```

ただし、ネイティブコードの変更（新しい Config Plugin の追加など）は OTA では配信できず、新しいビルドとストア審査が必要です。

### ユーザーフィードバックの収集

リリース後は App Store のレビューを定期的に確認し、ユーザーの声を拾います。特に低評価レビューには素早く返信し、改善に活かしましょう。バグ報告に対しては修正版のリリーススケジュールを伝えると、ユーザーの信頼を得られます。

## この章の成果物

ここまでで、以下の状態になっているはずです。

- EAS Build でiOS / Android の本番ビルドが生成されている
- App Store Connect / Google Play Console にアプリ情報が入力されている
- `eas submit` でビルドがストアにアップロードされている
- 審査のポイントを理解し、リジェクトを回避する準備ができている

おめでとうございます。第1章から第8章まで、Expo プロジェクトの作成から App Store への公開まで、一通りの開発フローを実装してきました。Expo + Supabase + RevenueCat の3つのサービスを連携させることで、認証・データ管理・サブスクリプション課金を備えた本格的なアプリを、個人開発者でも効率的に作れることを体験していただけたと思います。

## ソースコード

### `app.json`

```json
{
  "expo": {
    "name": "ふくログ",
    "slug": "side-income-tracker",
    "version": "1.0.0",
    "orientation": "portrait",
    "icon": "./assets/images/icon.png",
    "scheme": "sideincometracker",
    "userInterfaceStyle": "dark",
    "newArchEnabled": true,
    "splash": {
      "image": "./assets/images/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#09090B"
    },
    "ios": {
      "supportsTablet": false,
      "bundleIdentifier": "com.fukulog.app",
      "infoPlist": {
        "ITSAppUsesNonExemptEncryption": false
      }
    },
    "android": {
      "package": "com.fukulog.app",
      "adaptiveIcon": {
        "foregroundImage": "./assets/images/adaptive-icon.png",
        "backgroundColor": "#09090B"
      },
      "edgeToEdgeEnabled": true,
      "predictiveBackGestureEnabled": false,
      "permissions": [
        "INTERNET",
        "com.android.vending.BILLING"
      ]
    },
    "web": {
      "bundler": "metro",
      "output": "static",
      "favicon": "./assets/images/favicon.png"
    },
    "plugins": [
      "expo-router",
      "expo-secure-store",
      "expo-web-browser"
    ],
    "experiments": {
      "typedRoutes": true
    },
    "extra": {
      "eas": {
        "projectId": "9ecb0f59-d793-4981-8d13-6c011a1d1278"
      },
      "router": {}
    },
    "owner": "kenty1031"
  }
}
```

### `eas.json`

```json
{
  "cli": {
    "version": ">= 15.0.0",
    "appVersionSource": "remote"
  },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "preview": {
      "distribution": "internal",
      "android": {
        "buildType": "apk"
      }
    },
    "production": {
      "autoIncrement": true,
      "android": {
        "buildType": "app-bundle"
      },
      "ios": {
        "autoIncrement": true
      }
    }
  },
  "submit": {
    "production": {
      "android": {
        "serviceAccountKeyPath": "./google-services-key.json",
        "track": "production"
      }
    }
  }
}
```
