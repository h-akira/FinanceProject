# FinanceDashboardProject_Backend 基本設計

## 概要

FinanceDashboardProject_Backendは、チャート・経済指標ダッシュボードのバックエンドです。
WAMBDAフレームワークを使用し、認証UIのサーバーサイドレンダリングとREST APIの両方を提供します。

## 技術スタック

- **フレームワーク:** WAMBDA（独自Pythonフレームワーク、Django風）
- **言語:** Python 3.13
- **デプロイツール:** AWS SAM
- **インフラ:**
  - Lambda関数（単一関数）
  - API Gateway（Regional）
  - CloudWatch Logs

## アーキテクチャ

### WAMBDAフレームワーク

WAMBDAはLambda上で動作するDjango風のWebフレームワークです。

**詳細:** https://github.com/h-akira/wambda

### 環境変数

以下の環境変数が使用されます（SAMテンプレートのParametersで定義）:

- `WAMBDA_DEBUG` - デバッグモード
- `WAMBDA_USE_MOCK` - モックデータ使用
- `WAMBDA_NO_AUTH` - 認証バイパス
- `WAMBDA_DENY_SIGNUP` - サインアップ抑制
- `WAMBDA_DENY_LOGIN` - ログイン抑制
- `WAMBDA_LOG_LEVEL` - ログレベル
- `DYNAMODB_TABLE` - DynamoDBテーブル名

## 機能一覧

### 認証機能（サーバーサイドレンダリング）

`/accounts/*` パスで提供される認証UI：
- ログイン画面（`/accounts/login`）
- サインアップ画面（`/accounts/signup`）
- メール確認画面（`/accounts/verify`）
- パスワード変更画面（`/accounts/change-password`）
- パスワードリセット画面（`/accounts/forgot-password`, `/accounts/reset-password`）
- プロフィール画面（`/accounts/profile`）
- アカウント削除画面（`/accounts/delete-account`）
- ログアウト処理（`/accounts/logout`）
- 認証状態確認API（`/accounts/status`）

### REST API

`/api/*` パスで提供されるJSON API：
- `/api/hello` - サンプルAPI


## ディレクトリ構成

```
FinanceDashboardProject_Backend/
├── template.yaml           # SAMテンプレート
├── samconfig.toml          # SAM設定
├── buildspec.yml           # CodeBuild設定
└── Lambda/
    ├── lambda_function.py  # Lambda エントリーポイント
    ├── requirements.txt    # Python依存関係
    ├── project/
    │   ├── settings.py     # WAMBDA設定
    │   ├── urls.py         # ルートURLパターン
    │   └── views.py        # ホームビュー
    ├── accounts/
    │   ├── urls.py         # 認証URLパターン
    │   ├── views.py        # 認証ビュー
    │   └── forms.py        # 認証フォーム
    ├── api/
    │   ├── urls.py         # API URLパターン
    │   └── views.py        # API ビュー
    ├── templates/          # HTMLテンプレート
    │   ├── base.html
    │   └── accounts/
    │       ├── login.html
    │       ├── signup.html
    │       └── ...
    └── mock/
        ├── ssm.py          # SSMモック
        └── dynamodb.py     # DynamoDBモック
```

## URLルーティング

WAMBDAフレームワークのルーティング構成:

- `/accounts/*` - 認証関連（accounts appにルーティング）
- `/api/*` - REST API（api appにルーティング）
- `/` - Backend単体テスト用ホームページ（本番環境ではCloudFrontでFrontendにルーティング）

## CloudFormation Outputs

SAMデプロイ後、API Gateway エンドポイントURLがCloudFormation Exportsとして出力されます。
この値はCDK（Infrastructure）でCloudFrontのオリジンとして参照されます。
