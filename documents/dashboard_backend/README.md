# FinanceDashboardProject_Backend 基本設計

## 概要

FinanceDashboardProject_Backendは、チャート・経済指標ダッシュボードのバックエンドAPIです。

## 技術スタック

- **フレームワーク:** WAMBDA（独自フレームワーク）
- **言語:** Python 3.13
- **デプロイ:** AWS SAM
- **インフラ:**
  - Lambda関数
  - API Gateway

## 機能一覧

- ユーザー認証（サインアップ、ログイン、パスワードリセット）
- アカウント管理（プロフィール表示、パスワード変更、アカウント削除）
- 金融データダッシュボード（為替レート、経済指標等）

## デプロイ手順

### 前提条件

以下がデプロイ済みであること:
- FinanceProject_CICD/dashboard/codebuild-backend.yaml（CodeBuild）
- SSMパラメータの設定

### SSMパラメータ

以下のパラメータが必要:

```bash
# Cognito User Pool ID
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/cognito_user_pool_id" \
  --value "ap-northeast-1_XXXXXXXXX" \
  --type String

# Cognito App Client ID
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/cognito_app_client_id" \
  --value "XXXXXXXXXXXXXXXXXXXX" \
  --type String

# DynamoDB Table名
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/dynamodb_table_name" \
  --value "finance-dashboard-table" \
  --type String
```

### 手動デプロイ

```bash
cd FinanceDashboardProject_Backend

# ビルド
sam build

# デプロイ
AWS_PROFILE=finance sam deploy \
  --stack-name stack-finance-dashboard-backend \
  --capabilities CAPABILITY_IAM \
  --resolve-s3
```

### 自動デプロイ（CodeBuild）

GitHubリポジトリへのプッシュで自動デプロイされます。

**処理フロー:**
1. SSMパラメータストアから環境変数取得（WAMBDA_*, DYNAMODB_TABLE）
2. SAM build
3. SAM deploy

## CloudFormation Exports

以下の値をエクスポート（CDKで参照）:

```yaml
Outputs:
  ApiGatewayUrl:
    Description: API Gateway URL
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
    Export:
      Name: !Sub "${AWS::StackName}-ApiGatewayUrl"
```

## 依存関係

### デプロイ前に必要なもの
- CodeBuild（Backend）の作成
- SSMパラメータの設定

### デプロイ後に必要なもの
1. **Infrastructure（CDK）のデプロイ**
   - Cognito User Pool
   - DynamoDB Table
2. **Backend再デプロイ**
   - Cognito/DynamoDB作成後、正常動作させるため

**注意:**
- 初回デプロイ時はCognito/DynamoDBがないため正常動作しない
- CDKデプロイ後に再デプロイすることで正常動作する
- コード変更は不要（SSMパラメータストアから自動取得）

## API エンドポイント

### 認証関連
- `POST /auth/signup` - ユーザー登録
- `POST /auth/login` - ログイン
- `POST /auth/logout` - ログアウト
- `POST /auth/password-reset` - パスワードリセット

### アカウント管理
- `GET /accounts/profile` - プロフィール取得
- `PUT /accounts/profile` - プロフィール更新
- `PUT /accounts/password` - パスワード変更
- `DELETE /accounts` - アカウント削除

### 金融データ
- `GET /dashboard/data` - ダッシュボードデータ取得

## トラブルシューティング

### デプロイエラー

**症状:** SAM deployが失敗する

**原因と対処:**
1. **権限不足:** CodeBuild実行ロールの権限を確認
2. **SSMパラメータ未設定:** 必要なパラメータがすべて設定されているか確認
3. **スタック名重複:** 既存のスタックと名前が重複していないか確認

### ランタイムエラー

**症状:** Lambda関数が正常に動作しない

**原因と対処:**
1. **Cognito未作成:** Infrastructure（CDK）をデプロイ
2. **DynamoDB未作成:** Infrastructure（CDK）をデプロイ
3. **環境変数未設定:** SSMパラメータが正しく設定されているか確認
4. **権限不足:** Lambda実行ロールがCognito/DynamoDBにアクセスできるか確認
