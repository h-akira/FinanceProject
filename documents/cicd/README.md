# FinanceProject_CICD 基本設計

## 概要

FinanceProject_CICDは、CI/CDパイプライン（CodeBuild）をCloudFormationで管理するリポジトリです。

## ディレクトリ構成

```
FinanceProject_CICD/
├── common/
│   └── codebuild-infra.yaml         # インフラCI/CD（CDKデプロイ）
└── dashboard/
    ├── codebuild-backend.yaml       # Dashboardバックエンド（SAMデプロイ）
    └── codebuild-frontend.yaml      # Dashboardフロントエンド（S3 sync）
```

**設計意図:**
- `common/` - 全サブシステム共通のインフラCI/CD
- `dashboard/` - FinanceDashboardProject専用のCI/CD
- サブシステムごとにディレクトリを分離し、拡張性を確保

## CodeBuildプロジェクト一覧

### 1. Infrastructure（common/codebuild-infra.yaml）

**役割:** FinanceProject_InfraのCDKデプロイを自動化

**トリガー:** GitHubプッシュ（FinanceProject_Infraリポジトリ）

**処理フロー:**
1. SSMパラメータストアから設定値取得
2. config.jsonに値を置換
3. CDK bootstrap
4. CDK deploy --all

**必要な権限:**
- CDK関連リソースの操作（STS、CloudFormation、S3、ECR、IAM、SSM）

### 2. Backend（dashboard/codebuild-backend.yaml）

**役割:** FinanceDashboardProject_BackendのSAMデプロイを自動化

**トリガー:** GitHubプッシュ（FinanceDashboardProject_Backendリポジトリ）

**処理フロー:**
1. SSMパラメータストアから環境変数取得
2. SAM build
3. SAM deploy

**必要な権限:**
- SAM関連リソースの操作（CloudFormation、Lambda、API Gateway、S3、IAM）

### 3. Frontend（dashboard/codebuild-frontend.yaml）

**役割:** FinanceDashboardProject_Frontendのビルドとデプロイを自動化

**トリガー:** GitHubプッシュ（FinanceDashboardProject_Frontendリポジトリ）

**処理フロー:**
1. npm install
2. npm run build
3. S3 sync
4. CloudFront invalidation

**必要な権限:**
- S3、CloudFront操作権限

## デプロイ手順

### 1. Infrastructure用CodeBuild

```bash
cd FinanceProject_CICD/common
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-infra.yaml \
  --stack-name stack-finance-cicd-infra \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    GitHubToken=<YOUR_GITHUB_TOKEN> \
    RepositoryUrl=https://github.com/YOUR_ORG/FinanceProject_Infra
```

### 2. Backend用CodeBuild

```bash
cd FinanceProject_CICD/dashboard
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-backend.yaml \
  --stack-name stack-finance-dashboard-cicd-backend \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    GitHubToken=<YOUR_GITHUB_TOKEN> \
    RepositoryUrl=https://github.com/YOUR_ORG/FinanceDashboardProject_Backend
```

### 3. Frontend用CodeBuild

```bash
cd FinanceProject_CICD/dashboard
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file codebuild-frontend.yaml \
  --stack-name stack-finance-dashboard-cicd-frontend \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    GitHubToken=<YOUR_GITHUB_TOKEN> \
    RepositoryUrl=https://github.com/YOUR_ORG/FinanceDashboardProject_Frontend \
    S3BucketName=<YOUR_S3_BUCKET> \
    CloudFrontDistributionId=<YOUR_DISTRIBUTION_ID>
```

## GitHubとの連携

### Webhookの設定

各CodeBuildプロジェクトは、GitHubリポジトリのプッシュイベントをトリガーとして自動実行されます。

**Webhook設定:**
- イベント: PUSH
- ブランチフィルタ: main（デフォルト）

### 必要なGitHubトークン権限
- `repo`: リポジトリへのフルアクセス
- `admin:repo_hook`: Webhookの作成・管理

## 依存関係

### Infrastructure CodeBuild
**デプロイ前に必要:**
- CDK実行ポリシー（CloudFormation）

**依存されるもの:**
- Backend、Frontend（インフラリソースが必要）

### Backend CodeBuild
**デプロイ前に必要:**
- なし（プロジェクト作成のみ）

**依存されるもの:**
- Infrastructure（API Gateway URL出力）

### Frontend CodeBuild
**デプロイ前に必要:**
- Infrastructure（S3バケット作成済み）

**依存されるもの:**
- なし

## 構築順序

### 詳細な依存関係

```
1. CDK実行ポリシー
   ↓
2. CodeBuild（Infrastructure）
   ↓
3. CodeBuild（Backend）※サブシステムごと
   ↓
4. Backend（SAM）※サブシステムごと
   │  ↓ API Gateway URL出力（CloudFormation Exports）
   │  ↓ ※この時点ではCognito/DynamoDBがなく正常動作しない
   ↓
5. 共通インフラ + サブシステムインフラ（CDK 一括）
   │  - Cognito（共通）
   │  - DynamoDB（サブシステムごと）
   │  - S3 + CloudFront（サブシステムごと）← API Gateway URL参照
   ↓
6. Backend再デプロイ（SAM）※正常動作のため
   ↓
7. CodeBuild（Frontend）※サブシステムごと
   ↓
8. Frontend（npm build & S3 sync）※サブシステムごと
   ↓
9. Route53（DNS設定）
```

### 重要な依存関係

- **Backend（SAM）は、Cognito/DynamoDBなしでもデプロイ可能**（ただし動作しない）
- **CDK（共通+サブシステムインフラ）は、Backend（SAM）のAPI Gateway URL出力を参照**するため、SAM完了後に実行
- **CDKは共通リソースとサブシステムリソースを一括デプロイ**（Cognito、DynamoDB、S3、CloudFront）
- **Backend再デプロイで、Cognito/DynamoDB作成後に正常動作**
- **Frontendデプロイは、CDKで作成されたS3バケットが必要**

### サブシステム追加時の考慮

- Backend（SAM）とFrontendは、サブシステムごとに独立したCodeBuildとリポジトリ
- CDKは単一リポジトリで、スタックとしてサブシステムリソースを追加

### 実際の構築手順

1. Infrastructure用CodeBuild作成
2. Backend用CodeBuild作成
3. Backend初回デプロイ（自動またはビルド手動トリガー）
4. Infrastructure初回デプロイ（自動またはビルド手動トリガー）
5. Backend再デプロイ（Cognito/DynamoDB作成後）
6. Frontend用CodeBuild作成
7. Frontend初回デプロイ（自動またはビルド手動トリガー）

## トラブルシューティング

### ビルドが失敗する場合

1. **権限エラー:** CodeBuild実行ロールの権限を確認
2. **SSMパラメータ未設定:** 必要なパラメータがすべて設定されているか確認
3. **GitHubトークン期限切れ:** GitHubトークンを再発行してパラメータ更新

### Webhookが動作しない場合

1. GitHubリポジトリのWebhook設定を確認
2. CodeBuildプロジェクトのソース設定でWebhookが有効か確認
3. GitHubトークンの権限を確認
