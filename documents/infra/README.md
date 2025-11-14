# FinanceProject_Infra 基本設計

## 概要

FinanceProject_Infraは、全サブシステムのインフラをAWS CDK（Python）で管理するリポジトリです。

## ディレクトリ構成

```
FinanceProject_Infra/
├── stacks/
│   ├── common/          # 全サブシステム共通リソース（Cognito等）
│   └── dashboard/       # FinanceDashboardProject用インフラ（DynamoDB等）
├── init/                # 初期セットアップ用CloudFormation
│   ├── cfn-execution-policies.yaml  # CDK実行ポリシー
│   └── README.md
├── app.py               # CDKアプリケーションエントリーポイント
├── config.json          # 環境設定
└── buildspec.yml        # CodeBuild設定
```

## リソース管理

### CDK管理リソース
- Cognito User Pool（共通）
- DynamoDB Table（サブシステムごと）
- S3 + CloudFront（予定）

### 手動管理リソース
- Route53ホストゾーン
- ACM証明書
- SSMパラメータ（初期設定）

## デプロイ手順

### 1. CDK実行ポリシーの作成

```bash
cd FinanceProject_Infra/init
AWS_PROFILE=finance aws cloudformation deploy \
  --template-file cfn-execution-policies.yaml \
  --stack-name stack-finance-cdk-execution-policies \
  --capabilities CAPABILITY_NAMED_IAM
```

**このステップで作成されるもの:**
- CloudFormation実行用の4つのマネージドポリシー:
  1. `CognitoDynamoDBPolicy`: Cognito、DynamoDB
  2. `StoragePolicy`: S3、CloudFront
  3. `ConfigPolicy`: SSM、ACM、Lambda、CloudWatch Logs
  4. `IAMPolicy`: IAMロール・ポリシー管理、PassRole

### 2. SSMパラメータの設定

以下のパラメータをSSM Parameter Storeに設定：

```bash
# ACM証明書ARN
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/acm_certificate_arn" \
  --value "arn:aws:acm:us-east-1:ACCOUNT_ID:certificate/CERT_ID" \
  --type String

# DynamoDB Table名
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/dynamodb_table_name" \
  --value "finance-dashboard-table" \
  --type String

# Backend SAMスタック名
AWS_PROFILE=finance aws ssm put-parameter \
  --name "/finance/dashboard/backend_stack_name" \
  --value "stack-finance-dashboard-backend" \
  --type String
```

### 3. CDKブートストラップ

```bash
cd FinanceProject_Infra
AWS_PROFILE=finance cdk bootstrap \
  --cloudformation-execution-policies \
    arn:aws:iam::ACCOUNT_ID:policy/CognitoDynamoDBPolicy \
    arn:aws:iam::ACCOUNT_ID:policy/StoragePolicy \
    arn:aws:iam::ACCOUNT_ID:policy/ConfigPolicy \
    arn:aws:iam::ACCOUNT_ID:policy/IAMPolicy
```

### 4. CDKデプロイ

```bash
AWS_PROFILE=finance cdk deploy --all
```

## CI/CDフロー

**トリガー:** GitHubプッシュ（FinanceProject_Infraリポジトリ）

**処理フロー:**
1. SSMパラメータストアから設定値取得
2. config.jsonに値を置換
3. CDK bootstrap（CloudFormation実行ポリシー適用）
4. CDK deploy --all（全スタック一括デプロイ）

**デプロイ対象:**
- 共通リソース: Cognito User Pool
- サブシステムリソース: DynamoDB Table、S3 + CloudFront

**注意事項:**
- Backend（SAM）が先にデプロイされている必要がある（API Gateway URLを参照するため）
- 複数サブシステムがある場合も、全スタックを一括でデプロイ

## 権限モデル

### CodeBuild実行ロール
CDK関連リソースのみ操作可能：
- STS: CDKブートストラップロールへのAssumeRole
- CloudFormation: スタック管理のみ
- S3: CDKアセットバケット（`cdk-*`）のみ
- ECR: CDKブートストラップリポジトリ（`cdk-*`）のみ
- IAM: CDKブートストラップロール（`cdk-*`）のみ
- SSM: パラメータストアの読み書き

### CloudFormation実行ポリシー
実際のAWSリソース作成権限（4つのポリシーに分割）：
1. **CognitoDynamoDBPolicy**: Cognito、DynamoDB
2. **StoragePolicy**: S3、CloudFront
3. **ConfigPolicy**: SSM、ACM（読み取りのみ）、Lambda（CDKカスタムリソース用）、CloudWatch Logs
4. **IAMPolicy**: IAMロール・ポリシー管理、PassRole（`stack-*`パターンのみ）

この2層分離により、CodeBuild自体は直接AWSリソースを作成できず、CloudFormationを通じてのみリソース操作が可能となります。

## 依存関係

### デプロイ前に必要なもの
1. CDK実行ポリシー（CloudFormation）
2. Backend（SAM）のデプロイ（API Gateway URL出力）
3. SSMパラメータの設定

### 依存されるもの
- Frontend（S3バケットが必要）
- Backend（Cognito、DynamoDBが必要）

## セキュリティ・運用

### アクセス制御

#### アプリケーション層
- **CloudFront**: HTTPS強制、カスタムドメイン
- **S3**: OAI（Origin Access Identity）でCloudFrontからのみアクセス許可
- **API Gateway**: CORS設定、Cognito認証
- **Lambda**: 最小権限の実行ロール

#### データ層
- **Cognito**: User Pool、MFA設定（オプション）
- **DynamoDB**: IAMロールベースのアクセス制御

### データ保護

#### バックアップ
- **DynamoDB**: Point-in-Time Recovery（PITR）有効化
  - 最大35日間の継続的バックアップ
  - 任意の時点への復元が可能

#### 誤削除防止
- **Cognito User Pool**: RemovalPolicy.RETAIN
  - CloudFormationスタック削除時もUser Poolは保持
  - ユーザーデータの誤削除を防止
- **DynamoDB Table**: RemovalPolicy.RETAIN
  - CloudFormationスタック削除時もテーブルは保持
  - データの誤削除を防止

### 監視・ログ

#### CloudWatch Logs
- Lambda関数のログ
- API Gatewayのアクセスログ
- CDKカスタムリソースのログ

#### CloudWatch Metrics
- Lambda: 実行時間、エラー率、同時実行数
- API Gateway: リクエスト数、レイテンシ、エラー率
- DynamoDB: 読み取り/書き込み容量、スロットリング

#### アラート設定（推奨）
- Lambda関数のエラー率が5%を超えた場合
- API Gatewayの5xxエラーが発生した場合
- DynamoDBのスロットリングが発生した場合

### コスト最適化

- **Lambda**: メモリサイズの最適化、予約済み同時実行数の設定
- **DynamoDB**: オンデマンドモードまたはプロビジョニングモードの選択
- **CloudFront**: キャッシュ最適化、不要なオリジンリクエストの削減
- **S3**: ライフサイクルポリシー、不要なオブジェクトの削除
