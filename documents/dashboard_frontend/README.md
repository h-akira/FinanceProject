# FinanceDashboardProject_Frontend 基本設計

## 概要

FinanceDashboardProject_Frontendは、チャート・経済指標ダッシュボードのフロントエンドです。
シンプルなSPA（Single Page Application）として実装され、認証はBackend（サーバーサイドレンダリング）で管理されます。

## 技術スタック

- **フレームワーク:** Vue.js 3.0.4
- **ルーティング:** Vue Router 4.0.0
- **UIライブラリ:** Bootstrap 5.3.0
- **ビルドツール:** Vite 1.0.0
- **ホスティング:**
  - S3（静的ファイル）
  - CloudFront（CDN + ルーティング）

## アーキテクチャ

### ルーティング構成

CloudFrontで以下のようにルーティングされます：

- `/api/*` → API Gateway（Backend Lambda）
- `/accounts/*` → API Gateway（Backend Lambda - 認証管理）
- その他 → S3（Frontend静的ファイル）

### 認証フロー

1. フロントエンドは`/accounts/status`を呼び出して認証状態を確認
2. 未認証の場合、`/accounts/login`にリダイレクト
3. ログイン処理はBackendのサーバーサイドレンダリングで実行
4. ログイン成功後、元のページにリダイレクト

**重要:** 認証UI（ログイン画面等）はフロントエンドに含まれず、Backendで管理されます。

## 機能一覧

- 金融データダッシュボード
  - Market Summary（市場サマリー）
  - Calendar（経済指標カレンダー）
  - Exchange（為替レート）
  - JP225（日経平均）
  - M2（マネーサプライ）
- 各チャートの表示/非表示切り替え
- レスポンシブレイアウト（横配置数の変更）

## Vue Routerルーティング

フロントエンド内のルーティング（`src/main.js`）:

| パス | コンポーネント | 認証 | 説明 |
|------|--------------|------|------|
| `/` | Home | 必須 | ダッシュボードホーム |

**認証ガード:**
- 全ルートで`/accounts/status`を呼び出して認証状態を確認
- 未認証の場合、`/accounts/login?next=<元のパス>`にリダイレクト
- ログイン後、`next`パラメータで指定されたパスに戻る

## コンポーネント構成

各コンポーネントの詳細設計は[components.md](./components.md)を参照してください。

```
src/
├── components/
│   ├── Calendar.vue        # 経済指標カレンダー
│   ├── Exchange.vue        # 為替レート
│   ├── JP225.vue           # 日経平均
│   ├── M2.vue              # マネーサプライ
│   ├── MarketSummary.vue   # 市場サマリー
│   └── Home.vue            # ホームページ（ダッシュボード）
├── assets/
│   └── logo.png
├── App.vue                 # ルートコンポーネント（ナビゲーションバー含む）
├── main.js                 # エントリーポイント、Vue Router設定
└── index.css               # グローバルスタイル
```

## 認証UI（Backend管理）

以下の認証関連パスはBackendでサーバーサイドレンダリングされます:

| パス | 説明 |
|------|------|
| `/accounts/login` | ログイン画面 |
| `/accounts/logout` | ログアウト処理 |
| `/accounts/profile` | プロフィール画面 |
| `/accounts/status` | 認証状態確認API |

フロントエンドはこれらのエンドポイントを呼び出すだけで、UI実装は含まれません。
