# Parameter Store 管理

## 概要

Finance Projectでは、AWS Systems Manager Parameter Storeでプロジェクトの設定値を一元管理します。

## パラメータ一覧

| パラメータ名 | 現在値 | Type | 説明 | 設定者 | 用途 |
|------------|-------|------|------|--------|------|
| `/Cognito/user_pool_id` | `ap-northeast-1_wtuYlGL9f` | String | Cognito User Pool ID | Infrastructure（CDK） | コンポーネント間共有 |
| `/Cognito/client_id` | `12bjbqus6mledahk0olds506r7` | String | Cognito App Client ID | Infrastructure（CDK） | コンポーネント間共有 |
| `/Cognito/client_secret` | `1m6ru4cob16ijgt0...` | String | Cognito App Client Secret | Infrastructure（CDK） | コンポーネント間共有 |
| `/Common/ACM/arn` | `arn:aws:acm:us-east-1:...` | String | ACM証明書ARN（手動作成リソース） | 手動 | コンポーネント間共有、環境固有 |
| `/Dashboard/DynamoDB/main/table_name` | `table-finance-dashboard-main` | String | DynamoDB Table名 | 手動 | コンポーネント間共有、環境固有 |
| `/Dashboard/S3/contents/bucket_name` | `s3-finance-dashboard-contents` | String | Frontend用S3バケット名 | Infrastructure（CDK） | コンポーネント間共有、環境固有 |
| `/Dashboard/WAMBDA/DEBUG` | `true` | String | デバッグモード（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |
| `/Dashboard/WAMBDA/USE_MOCK` | `false` | String | モックデータ使用（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |
| `/Dashboard/WAMBDA/NO_AUTH` | `false` | String | 認証バイパス（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |
| `/Dashboard/WAMBDA/DENY_SIGNUP` | `false` | String | サインアップ抑制（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |
| `/Dashboard/WAMBDA/DENY_LOGIN` | `false` | String | ログイン抑制（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |
| `/Dashboard/WAMBDA/LOG_LEVEL` | `DEBUG` | String | ログレベル（デプロイ時初期値） | 手動 | Lambda環境変数初期値 |

**ログレベル指定値:** `DEBUG` / `INFO` / `WARNING` / `ERROR`

**WAMBDA設定について:**
- WAMBDA設定パラメータはCodeBuild実行時にLambda関数の環境変数として設定される初期値です
- 運用中に動作を変更する場合は、Lambda関数の環境変数を直接編集してください

## パラメータ設定タイミング

### 手動設定が必要（初期セットアップ）
- `/Common/ACM/arn` - 手動作成リソース
- `/Dashboard/DynamoDB/main/table_name` - デプロイ前に設定
- `/Dashboard/WAMBDA/*` - デプロイ前に設定（Lambda環境変数の初期値）

### 自動設定（Infrastructure CDKがデプロイ時に設定）
- `/Cognito/user_pool_id`
- `/Cognito/client_id`
- `/Cognito/client_secret`
- `/Dashboard/S3/contents/bucket_name`
