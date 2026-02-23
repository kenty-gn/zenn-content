---
title: "はじめに"
free: true
---

# はじめに

## この本で作るもの

この本では、実際にApp Storeで公開している「ふくログ」というアプリの実装を通じて、Expo + Supabase + RevenueCat の3技術を統合したモバイルアプリの作り方を学びます。

<!-- TODO: ふくログのスクショを挿入 -->

ふくログは、副業をしているエンジニア向けの収入・経費管理アプリです。「20万円を超えたら確定申告が必要」のラインをプログレスバーでリアルタイムに可視化します。

### 主な機能
- 認証（メール/パスワード、Google、Apple OAuth）
- 収入・経費のCRUD
- 20万円ダッシュボード（リアルタイムプログレスバー）
- 月次・年次レポート
- サブスクリプション課金（RevenueCat）
- サーバーサイド課金検証（Supabase Edge Functions + Webhook）

## この本の対象読者

- React / React Nativeの基礎がわかる人
- Expoを触ったことがある or これから始めたい人
- アプリで収益を得たい人

## この本で学べること

1. ExpoプロジェクトのセットアップからApp Store公開まで
2. SupabaseでDB設計・認証・RLS（行レベルセキュリティ）
3. RevenueCatでサブスクリプション課金の実装
4. 3つのサービスを連携させるアーキテクチャ

## 使用する技術スタック

- Expo SDK 54 / Expo Router v6
- React Native 0.81 / React 19
- Supabase（PostgreSQL + Auth + Edge Functions）
- RevenueCat SDK
- TypeScript / Zustand / Zod

## サンプルコードについて

各章のコードはふくログの実際のコードをベースにしています。そのままコピペして動く形で掲載しています。

<!-- TODO: GitHubリポジトリURLを追加 -->
