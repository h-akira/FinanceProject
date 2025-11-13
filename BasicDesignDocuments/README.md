# Finance Project 基本設計書

## 1. プロジェクト概要

Finance Projectは、個人向け財務管理を目的としたWebアプリケーション群です。複数の独立したサブシステムで構成され、各サブシステムは独自のドメイン・インフラを持ちます。すべてのサブシステムは単一のAWSアカウントで管理され、共通の認証基盤を利用します。

**主な特徴:**
- サブシステムごとに独立したドメイン（サブドメイン）
- 共通認証基盤（Cognito）の利用
- 各サブシステムは独立したインフラ構成

## 2. リポジトリ構成

本プロジェクトは以下のリポジトリで構成されます：

### 2.1 FinanceProject_Infra
- **役割:** 全サブシステムのインフラをAWS CDKで管理
- **構成:** 単一リポジトリで、common/およびサブシステムごとのスタックで管理
- **ディレクトリ構成:**
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

### 2.2 FinanceProject_CICD
- **役割:** CI/CDパイプライン（CodeBuild）の管理
- **構成:** CloudFormationで独立管理
- **ディレクトリ構成:**
  ```
  FinanceProject_CICD/
  ├── common/
  │   └── codebuild-infra.yaml         # インフラCI/CD（CDKデプロイ）
  └── dashboard/
      ├── codebuild-backend.yaml       # Dashboardバックエンド（SAMデプロイ）
      └── codebuild-frontend.yaml      # Dashboardフロントエンド（S3 sync）
  ```
- **設計意図:**
  - `common/` - 全サブシステム共通のインフラCI/CD
  - `dashboard/` - FinanceDashboardProject専用のCI/CD
  - サブシステムごとにディレクトリを分離し、拡張性を確保

### 2.3 FinanceDashboardProject_Backend
- **役割:** チャート・経済指標ダッシュボードのバックエンド
- **構成:** AWS SAMでLambda + API Gatewayを管理

### 2.4 FinanceDashboardProject_Frontend
- **役割:** チャート・経済指標ダッシュボードのフロントエンド
- **構成:** Vue.js + Bootstrap

### 2.5 FinanceDashboardProject（レガシー）
- **役割:** Backend/Frontend統合版（廃止予定）
- **状態:** Backend、Frontendリポジトリへの分離が完了

## 3. AWSアカウント構成

- **アカウント:** 単一AWSアカウントで全サブシステムを統合管理
- **リージョン:** ap-northeast-1（東京）
- **AWSプロファイル:** `finance`

## 4. 共通基盤

### 4.1 共通認証基盤（Cognito）
- **概要:** 全サブシステムで共有する認証基盤
- **ドメイン:** なし（マネージドログイン未使用、バックエンドからのアクセスのみ）
- **配置:** `FinanceProject_Infra/stacks/common/cognito_stack.py`
- **機能一覧:**
  - ユーザー認証（サインアップ、ログイン、パスワードリセット）
  - ユーザー情報管理
  - メールアドレス確認
- **技術スタック:**
  - AWS Cognito User Pool
  - SSMパラメータストア（認証情報管理）
- **インフラ:** AWS CDK管理
- **保護:** RemovalPolicy.RETAIN（誤削除防止）

## 5. サブシステム一覧

### 5.1 FinanceDashboardProject
- **概要:** チャート・経済指標ダッシュボード
- **ドメイン:** dashboard.example.com（予定）
- **リポジトリ構成:**
  - FinanceDashboardProject_Backend（Backend）
  - FinanceDashboardProject_Frontend（Frontend）
  - FinanceProject_Infra/stacks/dashboard（DynamoDBインフラ）
- **機能一覧:**
  - ユーザー認証（サインアップ、ログイン、パスワードリセット）
  - アカウント管理（プロフィール表示、パスワード変更、アカウント削除）
  - 金融データダッシュボード（為替レート、経済指標等）
  - 各グラフ・表の表示/非表示切り替え
- **技術スタック:**
  - Backend: WAMBDA（独自フレームワーク）/ Python 3.13 / AWS SAM
  - Frontend: Vue.js 3.0.4、Vue Router、Bootstrap
- **インフラ:**
  - Lambda + API Gateway（Backend - SAM管理）
  - DynamoDB（CDK管理）
  - S3 + CloudFront（Frontend - 手動作成、将来CDK化予定）
  - CodeBuild（CloudFormation管理）
- **デプロイ:**
  - Backend: CodeBuild → SAM deploy
  - Frontend: CodeBuild → npm build → S3 sync → CloudFront invalidation

**リポジトリ分離状況:**
- ✅ Backend分離完了（FinanceDashboardProject_Backend）
- ✅ Frontend分離完了（FinanceDashboardProject_Frontend）
- ⏳ レガシーリポジトリ（FinanceDashboardProject）は廃止予定

## 6. インフラ構成

### 6.1 インフラ管理方針
- **IaC:** AWS CDK（Python）
- **リポジトリ:** FinanceProject_Infra（単一リポジトリ）
- **構成:** スタック単位で管理（common/、dashboard/など）

### 6.2 リソース管理分類
- **CloudFormation管理（手動）:**
  - CodeBuildプロジェクト（FinanceProject_CICD）
  - CDK CloudFormation実行ポリシー（init/cfn-execution-policies.yaml）
- **CDK管理（FinanceProject_Infra）:**
  - Cognito User Pool
  - DynamoDB Table
  - S3 + CloudFront（予定）
- **SAM管理（FinanceDashboardProject_Backend）:**
  - Lambda関数
  - API Gateway
- **手動作成:**
  - Route53（ホストゾーン）
  - ACM証明書
  - SSMパラメータ（初期設定）

### 6.3 全体アーキテクチャ図

![Finance Dashboard Architecture](finance_architecture.png)

**編集可能なファイル**: [finance_architecture.drawio](finance_architecture.drawio)

**補足:**
- Route53ホストゾーンは全サブシステムで共通
- Cognitoは全サブシステムで共有
- DynamoDBはサブシステムごとに独立

## 7. デプロイ指針

### 7.1 依存関係と構築順序

1. **CDK実行ポリシー**（CloudFormation - 手動）
   - CDKがCloudFormationを通じてリソースを作成するための権限を定義
   - 4つのマネージドポリシーに分割（Cognito/DynamoDB、Storage、Config、IAM）
   - **リポジトリ:** FinanceProject_Infra/init/cfn-execution-policies.yaml
   - **コマンド:** `aws cloudformation deploy`

2. **CodeBuild（Infrastructure）**（CloudFormation - 手動）
   - FinanceProject_Infraの自動デプロイパイプライン
   - CDK bootstrap + deployを実行
   - **リポジトリ:** FinanceProject_CICD/common/codebuild-infra.yaml
   - **依存:** CDK実行ポリシー
   - **コマンド:** `aws cloudformation deploy`

3. **CodeBuild（Backend）**（CloudFormation - 手動）
   - Backend自動デプロイパイプライン（サブシステムごとに作成）
   - **リポジトリ:** FinanceProject_CICD/dashboard/codebuild-backend.yaml
   - **注:** この段階ではCognito/DynamoDBは未作成だが、CodeBuildプロジェクト自体は作成可能

4. **Backend（SAM）**（CodeBuild自動）
   - Lambda関数のデプロイ
   - API Gatewayの作成
   - **リポジトリ:** FinanceDashboardProject_Backend（サブシステムごと）
   - **トリガー:** GitHubプッシュ → CodeBuild
   - **出力:** API Gateway URL（CloudFormation Exports）
   - **注:** Cognito/DynamoDBがないため、この段階では正常に動作しない

5. **共通インフラ + サブシステムインフラ**（CDK - 一括デプロイ）
   - **共通リソース:**
     - Cognito User Pool（全サブシステム共通）
   - **サブシステムごとのリソース:**
     - DynamoDB Table
     - S3 + CloudFront（API GatewayをオリジンとしてOrigin Groupに含む）
   - **リポジトリ:** FinanceProject_Infra
   - **トリガー:** GitHubプッシュ → CodeBuild
   - **依存:**
     - API Gateway URL（SAMスタックのExports参照）
     - ACM証明書ARN（SSMパラメータストア）
   - **重要:** SAMスタックが先に必要（API Gateway URLを参照するため）

6. **Backend再デプロイ**（SAM - 手動またはCodeBuild）
   - Cognito/DynamoDB作成後、Backendを再デプロイして正常動作させる
   - **注:** コードは変更不要（SSMパラメータストアから自動取得）

7. **CodeBuild（Frontend）**（CloudFormation - 手動）
   - Frontend配信の自動デプロイパイプライン（サブシステムごとに作成）
   - **リポジトリ:** FinanceProject_CICD/dashboard/codebuild-frontend.yaml
   - **依存:** S3バケット（CDKで作成済み）

8. **Frontend**（CodeBuild自動）
   - 静的ファイルのビルドとデプロイ
   - **リポジトリ:** FinanceDashboardProject_Frontend（サブシステムごと）
   - **トリガー:** GitHubプッシュ → CodeBuild
   - **依存:** S3バケット（CDKで作成済み）

9. **Route53**（手動）
   - DNSレコード設定
   - CloudFront Distributionへのルーティング

**依存関係図:**
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

**重要な依存関係:**
- Backend（SAM）は、Cognito/DynamoDBなしでもデプロイ可能（ただし動作しない）
- CDK（共通+サブシステムインフラ）は、Backend（SAM）のAPI Gateway URL出力を参照するため、SAM完了後に実行
- CDKは共通リソースとサブシステムリソースを一括デプロイ（Cognito、DynamoDB、S3、CloudFront）
- Backend再デプロイで、Cognito/DynamoDB作成後に正常動作
- Frontendデプロイは、CDKで作成されたS3バケットが必要

**サブシステム追加時の考慮:**
- Backend（SAM）とFrontendは、サブシステムごとに独立したCodeBuildとリポジトリ
- CDKは単一リポジトリで、スタックとしてサブシステムリソースを追加

### 7.2 CI/CDフロー

#### 7.2.1 FinanceProject_Infra（Infrastructure）
- **トリガー:** GitHubプッシュ（FinanceProject_Infraリポジトリ）
- **処理フロー:**
  1. SSMパラメータストアから設定値取得（ACM ARN、DynamoDB名、Backend SAMスタック名）
  2. config.jsonに値を置換
  3. CDK bootstrap（CloudFormation実行ポリシー適用）
  4. CDK deploy --all（全スタック一括デプロイ）
- **デプロイ対象:**
  - **共通リソース:**
    - Cognito User Pool（全サブシステム共通）
  - **サブシステムごとのリソース:**
    - DynamoDB Table
    - S3 + CloudFront（Backend SAMスタックのAPI Gateway URLを参照）
- **注:**
  - Backend（SAM）が先にデプロイされている必要がある（API Gateway URLを参照するため）
  - 複数サブシステムがある場合も、全スタックを一括でデプロイ

#### 7.2.2 Backend（サブシステムごと）
**例: FinanceDashboardProject_Backend**
- **トリガー:** GitHubプッシュ（各サブシステムのBackendリポジトリ）
- **処理フロー:**
  1. SSMパラメータストアから環境変数取得（WAMBDA_*, DYNAMODB_TABLE）
  2. SAM build
  3. SAM deploy
- **デプロイ対象:**
  - Lambda関数
  - API Gateway
- **CloudFormation Exports:**
  - API Gateway URLを出力（CDKで参照される）
- **注:**
  - 初回デプロイ時はCognito/DynamoDBがないため正常動作しない
  - CDKデプロイ後に再デプロイすることで正常動作

#### 7.2.3 Frontend（サブシステムごと）
**例: FinanceDashboardProject_Frontend**
- **トリガー:** GitHubプッシュ（各サブシステムのFrontendリポジトリ）
- **処理フロー:**
  1. npm install
  2. npm run build
  3. S3 sync
  4. CloudFront invalidation
- **デプロイ対象:**
  - S3（静的ファイル）
- **依存:**
  - CDKで作成されたS3バケットが必要

## 8. セキュリティ・運用

### 8.1 認証
- **AWS Cognito User Pool** による統合認証
- 全サブシステムで共通のUser Poolを使用
- 認証情報はSSMパラメータストアで一元管理

### 8.2 アクセス制御

#### 8.2.1 アプリケーション層
- CloudFrontでHTTPS強制
- S3バケットはOAI（Origin Access Identity）でCloudFrontからのみアクセス許可

#### 8.2.2 CI/CD権限モデル（CDK）
CDKを使用する場合、権限は2層に分離されます：

**CodeBuild実行ロール（codebuild-infra.yaml）:**
- CDK関連リソースのみ操作可能
  - STS: CDKブートストラップロールへのAssumeRole
  - CloudFormation: スタック管理のみ
  - S3: CDKアセットバケット（`cdk-*`）のみ
  - ECR: CDKブートストラップリポジトリ（`cdk-*`）のみ
  - IAM: CDKブートストラップロール（`cdk-*`）のみ
  - SSM: パラメータストアの読み書き

**CloudFormation実行ポリシー（cfn-execution-policies.yaml）:**
- 実際のAWSリソース作成権限
- 4つのポリシーに分割（最小権限の原則）:
  1. **CognitoDynamoDBPolicy**: Cognito、DynamoDB
  2. **StoragePolicy**: S3、CloudFront
  3. **ConfigPolicy**: SSM、ACM（読み取りのみ）、Lambda（CDKカスタムリソース用）、CloudWatch Logs
  4. **IAMPolicy**: IAMロール・ポリシー管理、PassRole（`stack-*`パターンのみ）

この2層分離により、CodeBuild自体は直接AWSリソースを作成できず、CloudFormationを通じてのみリソース操作が可能となります。

### 8.3 監視
- CloudWatch Logsによるログ管理
- Lambda、API Gatewayのメトリクス監視
- CodeBuildのビルドログ

### 8.4 バックアップ・データ保護
- DynamoDB: Point-in-Time Recovery（PITR）有効化
- Cognito: RemovalPolicy.RETAIN（誤削除防止）
- DynamoDB: RemovalPolicy.RETAIN（誤削除防止）

## 9. 今後の拡張と改善

### 9.1 実施済みリポジトリ分離
- ✅ FinanceDashboardProject → FinanceDashboardProject_Backend（完了）
- ✅ FinanceDashboardProject → FinanceDashboardProject_Frontend（完了）
- ⏳ レガシーリポジトリ（FinanceDashboardProject）の廃止

### 9.2 今後の改善項目
- S3 + CloudFrontのCDK化
- CDK Nagによるセキュリティベストプラクティスのチェック
- レガシーリポジトリ（FinanceDashboardProject）の完全削除

### 9.3 新規サブシステム追加手順（参考）
新しいサブシステムを追加する場合は、以下の手順を参考にしてください：

1. **インフラ（CDK）**
   - FinanceProject_Infra/stacks配下に新規スタック作成
   - 必要なリソース（DynamoDB、S3、CloudFrontなど）を定義

2. **CI/CD**
   - FinanceProject_CICD配下に新規ディレクトリを作成（例: `analytics/`）
   - CodeBuild用CloudFormationテンプレート作成（Backend用とFrontend用）
   ```
   FinanceProject_CICD/
   ├── common/
   ├── dashboard/
   └── analytics/           # 新規サブシステム
       ├── codebuild-backend.yaml
       └── codebuild-frontend.yaml
   ```

3. **アプリケーション**
   - Backend/Frontendの独自リポジトリを作成
   - 既存のFinanceDashboardProjectを参考に実装

4. **DNS**
   - Route53で新規サブドメイン設定
   - ACM証明書の取得
