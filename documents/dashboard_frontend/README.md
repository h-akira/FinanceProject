# FinanceDashboardProject_Frontend 基本設計

## 概要

FinanceDashboardProject_Frontendは、チャート・経済指標ダッシュボードのフロントエンドです。

## 技術スタック

- **フレームワーク:** Vue.js 3.0.4
- **ルーティング:** Vue Router
- **UIライブラリ:** Bootstrap
- **ビルドツール:** npm
- **ホスティング:**
  - S3（静的ファイル）
  - CloudFront（CDN）

## 機能一覧

- ユーザー認証UI（サインアップ、ログイン、パスワードリセット）
- アカウント管理UI（プロフィール表示、パスワード変更、アカウント削除）
- 金融データダッシュボード
  - 為替レートチャート
  - 経済指標表示
  - 各グラフ・表の表示/非表示切り替え

## 開発環境セットアップ

### 前提条件
- Node.js 16.x以上
- npm 8.x以上

### インストール

```bash
cd FinanceDashboardProject_Frontend
npm install
```

### ローカル開発サーバー

```bash
npm run serve
```

ブラウザで `http://localhost:8080` にアクセス

### ビルド

```bash
npm run build
```

ビルド成果物は `dist/` ディレクトリに生成されます。

## デプロイ手順

### 前提条件

以下がデプロイ済みであること:
- Infrastructure（CDK）: S3バケット、CloudFront
- FinanceProject_CICD/dashboard/codebuild-frontend.yaml（CodeBuild）

### 手動デプロイ

```bash
# ビルド
npm run build

# S3へアップロード
AWS_PROFILE=finance aws s3 sync dist/ s3://YOUR_BUCKET_NAME/ --delete

# CloudFront キャッシュクリア
AWS_PROFILE=finance aws cloudfront create-invalidation \
  --distribution-id YOUR_DISTRIBUTION_ID \
  --paths "/*"
```

### 自動デプロイ（CodeBuild）

GitHubリポジトリへのプッシュで自動デプロイされます。

**処理フロー:**
1. npm install
2. npm run build
3. S3 sync
4. CloudFront invalidation

## 環境変数

フロントエンドでは以下の環境変数を使用（ビルド時に埋め込み）:

```bash
# .env.production
VUE_APP_API_BASE_URL=https://api.dashboard.example.com
VUE_APP_COGNITO_USER_POOL_ID=ap-northeast-1_XXXXXXXXX
VUE_APP_COGNITO_APP_CLIENT_ID=XXXXXXXXXXXXXXXXXXXX
```

## ルーティング

| パス | コンポーネント | 説明 |
|------|--------------|------|
| `/` | Home | トップページ |
| `/login` | Login | ログイン |
| `/signup` | Signup | ユーザー登録 |
| `/dashboard` | Dashboard | ダッシュボード（認証必須） |
| `/profile` | Profile | プロフィール（認証必須） |

## コンポーネント構成

```
src/
├── components/
│   ├── auth/
│   │   ├── LoginForm.vue
│   │   └── SignupForm.vue
│   ├── dashboard/
│   │   ├── ChartComponent.vue
│   │   └── DataTable.vue
│   └── layout/
│       ├── Header.vue
│       └── Footer.vue
├── views/
│   ├── Home.vue
│   ├── Login.vue
│   ├── Signup.vue
│   ├── Dashboard.vue
│   └── Profile.vue
├── router/
│   └── index.js
├── store/
│   └── index.js
├── services/
│   ├── api.js
│   └── auth.js
├── App.vue
└── main.js
```

## API連携

### APIクライアント

```javascript
// services/api.js
import axios from 'axios'

const apiClient = axios.create({
  baseURL: process.env.VUE_APP_API_BASE_URL,
  headers: {
    'Content-Type': 'application/json'
  }
})

// 認証トークンの追加
apiClient.interceptors.request.use(config => {
  const token = localStorage.getItem('auth_token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

export default apiClient
```

### Cognito認証

```javascript
// services/auth.js
import {
  CognitoUserPool,
  CognitoUser,
  AuthenticationDetails
} from 'amazon-cognito-identity-js'

const userPool = new CognitoUserPool({
  UserPoolId: process.env.VUE_APP_COGNITO_USER_POOL_ID,
  ClientId: process.env.VUE_APP_COGNITO_APP_CLIENT_ID
})

export const login = (username, password) => {
  // ログイン処理
}

export const signup = (username, password, email) => {
  // サインアップ処理
}
```

## 依存関係

### デプロイ前に必要なもの
1. Infrastructure（CDK）: S3バケット、CloudFront
2. Backend（SAM）: API Gateway URL
3. CodeBuild（Frontend）の作成

### 依存されるもの
- なし

## トラブルシューティング

### ビルドエラー

**症状:** npm run buildが失敗する

**原因と対処:**
1. **依存関係エラー:** `npm install` を実行して依存パッケージを再インストール
2. **環境変数未設定:** `.env.production` ファイルが正しく設定されているか確認
3. **Node.jsバージョン不一致:** Node.js 16.x以上を使用しているか確認

### デプロイエラー

**症状:** S3 syncまたはCloudFront invalidationが失敗する

**原因と対処:**
1. **権限不足:** CodeBuild実行ロールにS3/CloudFront権限があるか確認
2. **バケット名誤り:** S3バケット名が正しいか確認
3. **DistributionID誤り:** CloudFront Distribution IDが正しいか確認

### 認証エラー

**症状:** ログインやサインアップができない

**原因と対処:**
1. **Cognito未作成:** Infrastructure（CDK）をデプロイ
2. **環境変数誤り:** Cognito User Pool IDやApp Client IDが正しいか確認
3. **CORS設定:** API GatewayでCORS設定が正しいか確認
4. **ネットワークエラー:** ブラウザの開発者ツールでネットワークエラーを確認
