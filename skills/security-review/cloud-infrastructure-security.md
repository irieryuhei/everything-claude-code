| name | description |
|------|-------------|
| cloud-infrastructure-security | クラウドプラットフォームへのデプロイ、インフラストラクチャの設定、IAMポリシーの管理、ログ/監視のセットアップ、CI/CDパイプラインの実装時にこのスキルを使用してください。ベストプラクティスに沿ったクラウドセキュリティチェックリストを提供します。 |

# クラウド＆インフラストラクチャセキュリティスキル

このスキルは、クラウドインフラストラクチャ、CI/CDパイプライン、デプロイメント設定がセキュリティのベストプラクティスに従い、業界標準に準拠していることを確認します。

## アクティブ化のタイミング

- クラウドプラットフォーム（AWS、Vercel、Railway、Cloudflare）へのアプリケーションデプロイ時
- IAMロールと権限の設定時
- CI/CDパイプラインのセットアップ時
- Infrastructure as Code（Terraform、CloudFormation）の実装時
- ログと監視の設定時
- クラウド環境でのシークレット管理時
- CDNとエッジセキュリティのセットアップ時
- 災害復旧とバックアップ戦略の実装時

## クラウドセキュリティチェックリスト

### 1. IAM＆アクセス制御

#### 最小権限の原則

```yaml
# ✅ 正しい: 最小限の権限
iam_role:
  permissions:
    - s3:GetObject  # 読み取りアクセスのみ
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*  # 特定のバケットのみ

# ❌ 間違い: 過度に広い権限
iam_role:
  permissions:
    - s3:*  # すべてのS3アクション
  resources:
    - "*"  # すべてのリソース
```

#### 多要素認証（MFA）

```bash
# ルート/管理者アカウントには常にMFAを有効化
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

#### 検証ステップ

- [ ] 本番環境でルートアカウントを使用していない
- [ ] すべての特権アカウントでMFAが有効
- [ ] サービスアカウントは長期間有効な認証情報ではなくロールを使用
- [ ] IAMポリシーは最小権限に従っている
- [ ] 定期的なアクセスレビューを実施
- [ ] 未使用の認証情報をローテーションまたは削除

### 2. シークレット管理

#### クラウドシークレットマネージャー

```typescript
// ✅ 正しい: クラウドシークレットマネージャーを使用
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString).key;

// ❌ 間違い: ハードコードまたは環境変数のみ
const apiKey = process.env.API_KEY; // ローテーションなし、監査なし
```

#### シークレットのローテーション

```bash
# データベース認証情報の自動ローテーションをセットアップ
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

#### 検証ステップ

- [ ] すべてのシークレットがクラウドシークレットマネージャー（AWS Secrets Manager、Vercel Secrets）に保存されている
- [ ] データベース認証情報の自動ローテーションが有効
- [ ] APIキーは少なくとも四半期ごとにローテーション
- [ ] コード、ログ、エラーメッセージにシークレットがない
- [ ] シークレットアクセスの監査ログが有効

### 3. ネットワークセキュリティ

#### VPCとファイアウォール設定

```terraform
# ✅ 正しい: 制限されたセキュリティグループ
resource "aws_security_group" "app" {
  name = "app-sg"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # 内部VPCのみ
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # HTTPSアウトバウンドのみ
  }
}

# ❌ 間違い: インターネットに公開
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # すべてのポート、すべてのIP！
  }
}
```

#### 検証ステップ

- [ ] データベースが公開アクセス可能でない
- [ ] SSH/RDPポートがVPN/踏み台サーバーのみに制限
- [ ] セキュリティグループが最小権限に従っている
- [ ] ネットワークACLが設定されている
- [ ] VPCフローログが有効

### 4. ログ＆監視

#### CloudWatch/ログ設定

```typescript
// ✅ 正しい: 包括的なログ
import { CloudWatchLogsClient, CreateLogStreamCommand } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // 機密データは絶対にログしない
      })
    }]
  });
};
```

#### 検証ステップ

- [ ] すべてのサービスでCloudWatch/ログが有効
- [ ] 認証失敗の試行がログされている
- [ ] 管理者アクションが監査されている
- [ ] ログ保持が設定されている（コンプライアンスで90日以上）
- [ ] 不審なアクティビティのアラートが設定されている
- [ ] ログが一元化され、改ざん防止されている

### 5. CI/CDパイプラインセキュリティ

#### セキュアなパイプライン設定

```yaml
# ✅ 正しい: セキュアなGitHub Actionsワークフロー
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # 最小限の権限

    steps:
      - uses: actions/checkout@v4

      # シークレットスキャン
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main

      # 依存関係監査
      - name: Audit dependencies
        run: npm audit --audit-level=high

      # 長期間有効なトークンではなくOIDCを使用
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

#### サプライチェーンセキュリティ

```json
// package.json - ロックファイルと整合性チェックを使用
{
  "scripts": {
    "install": "npm ci",  // 再現可能なビルドにはciを使用
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

#### 検証ステップ

- [ ] 長期間有効な認証情報の代わりにOIDCを使用
- [ ] パイプラインでシークレットスキャン
- [ ] 依存関係の脆弱性スキャン
- [ ] コンテナイメージスキャン（該当する場合）
- [ ] ブランチ保護ルールが適用されている
- [ ] マージ前にコードレビューが必要
- [ ] 署名付きコミットが適用されている

### 6. Cloudflare＆CDNセキュリティ

#### Cloudflareセキュリティ設定

```typescript
// ✅ 正しい: セキュリティヘッダー付きのCloudflare Workers
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);

    // セキュリティヘッダーを追加
    const headers = new Headers(response.headers);
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=()');

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

#### WAFルール

```bash
# CloudflareのマネージドWAFルールを有効化
# - OWASPコアルールセット
# - Cloudflareマネージドルールセット
# - レート制限ルール
# - ボット保護
```

#### 検証ステップ

- [ ] OWASPルール付きでWAFが有効
- [ ] レート制限が設定されている
- [ ] ボット保護がアクティブ
- [ ] DDoS保護が有効
- [ ] セキュリティヘッダーが設定されている
- [ ] SSL/TLS厳格モードが有効

### 7. バックアップ＆災害復旧

#### 自動バックアップ

```terraform
# ✅ 正しい: 自動RDSバックアップ
resource "aws_db_instance" "main" {
  allocated_storage     = 20
  engine               = "postgres"

  backup_retention_period = 30  # 30日間保持
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql"]

  deletion_protection = true  # 誤削除を防止
}
```

#### 検証ステップ

- [ ] 毎日の自動バックアップが設定されている
- [ ] バックアップ保持がコンプライアンス要件を満たしている
- [ ] ポイントインタイムリカバリが有効
- [ ] 四半期ごとにバックアップテストを実施
- [ ] 災害復旧計画がドキュメント化されている
- [ ] RPOとRTOが定義されテストされている

## デプロイ前クラウドセキュリティチェックリスト

本番クラウドデプロイの前に必ず確認：

- [ ] **IAM**: ルートアカウント未使用、MFA有効、最小権限ポリシー
- [ ] **シークレット**: ローテーション付きですべてのシークレットがクラウドシークレットマネージャーに保存
- [ ] **ネットワーク**: セキュリティグループが制限され、公開データベースなし
- [ ] **ログ**: 保持付きでCloudWatch/ログが有効
- [ ] **監視**: 異常のアラートが設定されている
- [ ] **CI/CD**: OIDC認証、シークレットスキャン、依存関係監査
- [ ] **CDN/WAF**: OWASPルール付きでCloudflare WAFが有効
- [ ] **暗号化**: 保存時と転送時でデータが暗号化されている
- [ ] **バックアップ**: テスト済みリカバリ付きで自動バックアップ
- [ ] **コンプライアンス**: GDPR/HIPAA要件を満たしている（該当する場合）
- [ ] **ドキュメント**: インフラストラクチャがドキュメント化され、ランブックが作成されている
- [ ] **インシデント対応**: セキュリティインシデント計画が整備されている

## 一般的なクラウドセキュリティ設定ミス

### S3バケットの公開

```bash
# ❌ 間違い: 公開バケット
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# ✅ 正しい: 特定のアクセスを持つプライベートバケット
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### RDSの公開アクセス

```terraform
# ❌ 間違い
resource "aws_db_instance" "bad" {
  publicly_accessible = true  # 絶対にこれをしない！
}

# ✅ 正しい
resource "aws_db_instance" "good" {
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

## リソース

- [AWSセキュリティベストプラクティス](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundationsベンチマーク](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflareセキュリティドキュメント](https://developers.cloudflare.com/security/)
- [OWASPクラウドセキュリティ](https://owasp.org/www-project-cloud-security/)
- [Terraformセキュリティベストプラクティス](https://www.terraform.io/docs/cloud/guides/recommended-practices/)

**覚えておくこと**: クラウドの設定ミスはデータ漏洩の主な原因です。一つの公開されたS3バケットや過度に許可されたIAMポリシーが、インフラストラクチャ全体を危険にさらす可能性があります。常に最小権限の原則と多層防御に従ってください。
