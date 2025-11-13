# Parameter Store 管理

## 概要

Finance Projectでは、コンポーネント間で共有される設定値をAWS Systems Manager Parameter Storeで一元管理します。これにより、各コンポーネントは他のコンポーネントのリソース情報を動的に取得でき、疎結合を保ちながら連携できます。

## パラメータ命名規則

```
/finance/{subsystem}/{parameter_name}
```

- `finance`: プロジェクト名
- `subsystem`: サブシステム名（`common`, `dashboard`など）
- `parameter_name`: パラメータ名（スネークケース）

## 共通パラメータ（common）

### Cognito関連

| パラメータ名 | 説明 | 設定者 | 参照者 |
|------------|------|--------|--------|
| `/finance/common/cognito_user_pool_id` | Cognito User Pool ID | Infrastructure（CDK） | Backend、Frontend |
| `/finance/common/cognito_app_client_id` | Cognito App Client ID | Infrastructure（CDK） | Backend、Frontend |

**設定例:**
```bash
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/common/cognito_user_pool_id" \
  --value "ap-northeast-1_XXXXXXXXX" \
  --type String \
  --overwrite
```

## ダッシュボードサブシステム（dashboard）

### Backend関連

| パラメータ名 | 説明 | 設定者 | 参照者 |
|------------|------|--------|--------|
| `/finance/dashboard/backend_stack_name` | BackendのCloudFormationスタック名 | 手動 | Infrastructure（CDK） |
| `/finance/dashboard/api_gateway_url` | API Gateway URL | Backend（SAM） | Frontend、Infrastructure（CDK） |

**設定例:**
```bash
# Backend stack名（手動設定）
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/backend_stack_name" \
  --value "stack-finance-dashboard-backend" \
  --type String

# API Gateway URL（SAMがCloudFormation Exportsで出力、CDKが読み取り）
# このパラメータはSAMデプロイ時に自動設定される
```

### DynamoDB関連

| パラメータ名 | 説明 | 設定者 | 参照者 |
|------------|------|--------|--------|
| `/finance/dashboard/dynamodb_table_name` | DynamoDB Table名 | 手動 | Infrastructure（CDK）、Backend（SAM） |

**設定例:**
```bash
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/dynamodb_table_name" \
  --value "finance-dashboard-table" \
  --type String
```

### Frontend関連

| パラメータ名 | 説明 | 設定者 | 参照者 |
|------------|------|--------|--------|
| `/finance/dashboard/s3_bucket_name` | Frontend用S3バケット名 | Infrastructure（CDK） | Frontend CI/CD |
| `/finance/dashboard/cloudfront_distribution_id` | CloudFront Distribution ID | Infrastructure（CDK） | Frontend CI/CD |

**設定例:**
```bash
# これらはCDKデプロイ時に自動設定される
```

### 証明書関連

| パラメータ名 | 説明 | 設定者 | 参照者 |
|------------|------|--------|--------|
| `/finance/dashboard/acm_certificate_arn` | ACM証明書ARN | 手動 | Infrastructure（CDK） |

**設定例:**
```bash
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/acm_certificate_arn" \
  --value "arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID" \
  --type String
```

## パラメータの依存関係

### 手動設定が必要なパラメータ（初期セットアップ）

1. `/finance/dashboard/backend_stack_name`
2. `/finance/dashboard/dynamodb_table_name`
3. `/finance/dashboard/acm_certificate_arn`

### 自動設定されるパラメータ

- Infrastructure（CDK）がデプロイ時に設定:
  - `/finance/common/cognito_user_pool_id`
  - `/finance/common/cognito_app_client_id`
  - `/finance/dashboard/s3_bucket_name`
  - `/finance/dashboard/cloudfront_distribution_id`

- Backend（SAM）がCloudFormation Exportsで出力（CDKが参照）:
  - API Gateway URL

## パラメータ参照の仕組み

### Infrastructure（CDK）

```python
# config.jsonのプレースホルダーをSSMパラメータで置換
import boto3

ssm = boto3.client('ssm')

# パラメータ取得
backend_stack = ssm.get_parameter(
    Name='/finance/dashboard/backend_stack_name'
)['Parameter']['Value']

# CloudFormation ExportsからAPI Gateway URL取得
cfn = boto3.client('cloudformation')
exports = cfn.list_exports()
api_url = next(
    e['Value'] for e in exports['Exports']
    if e['Name'] == f'{backend_stack}-ApiGatewayUrl'
)
```

### Backend（SAM）

```yaml
# template.yaml
Parameters:
  CognitoUserPoolId:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /finance/common/cognito_user_pool_id

  DynamoDBTableName:
    Type: AWS::SSM::Parameter::Value<String>
    Default: /finance/dashboard/dynamodb_table_name

Resources:
  LambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Environment:
        Variables:
          COGNITO_USER_POOL_ID: !Ref CognitoUserPoolId
          DYNAMODB_TABLE: !Ref DynamoDBTableName
```

### Frontend（CI/CD）

```yaml
# buildspec.yml
env:
  parameter-store:
    API_BASE_URL: /finance/dashboard/api_gateway_url
    COGNITO_USER_POOL_ID: /finance/common/cognito_user_pool_id
    COGNITO_APP_CLIENT_ID: /finance/common/cognito_app_client_id

phases:
  build:
    commands:
      - echo "VUE_APP_API_BASE_URL=$API_BASE_URL" > .env.production
      - echo "VUE_APP_COGNITO_USER_POOL_ID=$COGNITO_USER_POOL_ID" >> .env.production
      - echo "VUE_APP_COGNITO_APP_CLIENT_ID=$COGNITO_APP_CLIENT_ID" >> .env.production
      - npm run build
```

## パラメータ一覧の確認

### 全パラメータの一覧

```bash
AWS_PROFILE=finance aws ssm get-parameters-by-path \
  --path "/finance" \
  --recursive
```

### 特定サブシステムのパラメータ

```bash
AWS_PROFILE=finance aws ssm get-parameters-by-path \
  --path "/finance/dashboard" \
  --recursive
```

## トラブルシューティング

### パラメータが見つからない

**症状:** デプロイ時に「Parameter not found」エラー

**原因と対処:**
1. パラメータ名のtypo確認
2. 手動設定が必要なパラメータが未設定
3. リージョンが異なる（Parameter Storeはリージョンごと）

### パラメータの権限エラー

**症状:** 「Access Denied」エラー

**原因と対処:**
1. 実行ロールにSSM読み取り権限があるか確認
2. パラメータのKMS暗号化キーへのアクセス権限確認（SecureStringの場合）

### パラメータの更新が反映されない

**症状:** パラメータを更新したが、アプリケーションに反映されない

**原因と対処:**
1. Lambda関数の再起動（新しいコンテナで起動されるまで待つ）
2. CodeBuildの再実行（ビルド時にパラメータを読み込むため）
3. キャッシュのクリア（CloudFormationスタックの更新が必要な場合）
