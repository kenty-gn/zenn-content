---
title: "第8章: App Storeに公開する — EAS Build & Submit"
free: false
---

# 第8章: App Storeに公開する — EAS Build & Submit

<!-- TODO: 執筆予定 Week 9-10 -->
<!-- アウトライン:
  8.1 EAS Buildの準備
  8.2 iOSビルド
  8.3 Androidビルド
  8.4 App Store Connectの設定
  8.5 審査への提出
  8.6 リリース後にやること
-->


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
      "supportsTablet": true,
      "bundleIdentifier": "com.fukulog.app"
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
