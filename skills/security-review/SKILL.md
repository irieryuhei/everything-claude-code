---
name: security-review
description: 認証の追加、ユーザー入力の処理、シークレットの扱い、APIエンドポイントの作成、または決済/機密機能の実装時に使用。包括的なセキュリティチェックリストとパターンを提供。
---

# セキュリティレビュースキル

このスキルは、すべてのコードがセキュリティベストプラクティスに従い、潜在的な脆弱性を特定することを保証します。

## 使用タイミング

- 認証または認可の実装時
- ユーザー入力またはファイルアップロードの処理時
- 新しいAPIエンドポイントの作成時
- シークレットまたは資格情報の扱い時
- 決済機能の実装時
- 機密データの保存または転送時
- サードパーティAPIの統合時

## セキュリティチェックリスト

### 1. シークレット管理

#### ❌ これは絶対にしない
```typescript
const apiKey = "sk-proj-xxxxx"  // ハードコードされたシークレット
const dbPassword = "password123" // ソースコード内
```

#### ✅ 常にこうする
```typescript
const apiKey = process.env.OPENAI_API_KEY
const dbUrl = process.env.DATABASE_URL

// シークレットの存在を確認
if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

#### 確認ステップ
- [ ] ハードコードされたAPIキー、トークン、パスワードがない
- [ ] すべてのシークレットが環境変数に格納
- [ ] `.env.local`が.gitignoreに含まれている
- [ ] gitの履歴にシークレットがない
- [ ] 本番シークレットはホスティングプラットフォーム（Vercel、Railway）に格納

### 2. 入力バリデーション

#### 常にユーザー入力をバリデート
```typescript
import { z } from 'zod'

// バリデーションスキーマを定義
const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150)
})

// 処理前にバリデート
export async function createUser(input: unknown) {
  try {
    const validated = CreateUserSchema.parse(input)
    return await db.users.create(validated)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { success: false, errors: error.errors }
    }
    throw error
  }
}
```

#### ファイルアップロードのバリデーション
```typescript
function validateFileUpload(file: File) {
  // サイズチェック（最大5MB）
  const maxSize = 5 * 1024 * 1024
  if (file.size > maxSize) {
    throw new Error('File too large (max 5MB)')
  }

  // 型チェック
  const allowedTypes = ['image/jpeg', 'image/png', 'image/gif']
  if (!allowedTypes.includes(file.type)) {
    throw new Error('Invalid file type')
  }

  // 拡張子チェック
  const allowedExtensions = ['.jpg', '.jpeg', '.png', '.gif']
  const extension = file.name.toLowerCase().match(/\.[^.]+$/)?.[0]
  if (!extension || !allowedExtensions.includes(extension)) {
    throw new Error('Invalid file extension')
  }

  return true
}
```

#### 確認ステップ
- [ ] すべてのユーザー入力がスキーマでバリデートされている
- [ ] ファイルアップロードが制限されている（サイズ、型、拡張子）
- [ ] ユーザー入力がクエリで直接使用されていない
- [ ] ホワイトリストバリデーション（ブラックリストではない）
- [ ] エラーメッセージが機密情報を漏らさない

### 3. SQLインジェクション防止

#### ❌ 絶対にSQLを連結しない
```typescript
// 危険 - SQLインジェクションの脆弱性
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)
```

#### ✅ 常にパラメータ化クエリを使用
```typescript
// 安全 - パラメータ化クエリ
const { data } = await supabase
  .from('users')
  .select('*')
  .eq('email', userEmail)

// または生SQLの場合
await db.query(
  'SELECT * FROM users WHERE email = $1',
  [userEmail]
)
```

#### 確認ステップ
- [ ] すべてのデータベースクエリがパラメータ化クエリを使用
- [ ] SQLで文字列連結がない
- [ ] ORM/クエリビルダーが正しく使用されている
- [ ] Supabaseクエリが適切にサニタイズされている

### 4. 認証と認可

#### JWTトークン処理
```typescript
// ❌ 間違い: localStorage（XSSに脆弱）
localStorage.setItem('token', token)

// ✅ 正しい: httpOnlyクッキー
res.setHeader('Set-Cookie',
  `token=${token}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600`)
```

#### 認可チェック
```typescript
export async function deleteUser(userId: string, requesterId: string) {
  // 常に最初に認可を確認
  const requester = await db.users.findUnique({
    where: { id: requesterId }
  })

  if (requester.role !== 'admin') {
    return NextResponse.json(
      { error: 'Unauthorized' },
      { status: 403 }
    )
  }

  // 削除を続行
  await db.users.delete({ where: { id: userId } })
}
```

#### Row Level Security（Supabase）
```sql
-- すべてのテーブルでRLSを有効化
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- ユーザーは自分のデータのみ閲覧可能
CREATE POLICY "Users view own data"
  ON users FOR SELECT
  USING (auth.uid() = id);

-- ユーザーは自分のデータのみ更新可能
CREATE POLICY "Users update own data"
  ON users FOR UPDATE
  USING (auth.uid() = id);
```

#### 確認ステップ
- [ ] トークンがhttpOnlyクッキーに保存（localStorageではない）
- [ ] 機密操作前に認可チェック
- [ ] SupabaseでRow Level Securityが有効
- [ ] ロールベースのアクセス制御が実装されている
- [ ] セッション管理がセキュア

### 5. XSS防止

#### HTMLのサニタイズ
```typescript
import DOMPurify from 'isomorphic-dompurify'

// 常にユーザー提供のHTMLをサニタイズ
function renderUserContent(html: string) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}
```

#### Content Security Policy
```typescript
// next.config.js
const securityHeaders = [
  {
    key: 'Content-Security-Policy',
    value: `
      default-src 'self';
      script-src 'self' 'unsafe-eval' 'unsafe-inline';
      style-src 'self' 'unsafe-inline';
      img-src 'self' data: https:;
      font-src 'self';
      connect-src 'self' https://api.example.com;
    `.replace(/\s{2,}/g, ' ').trim()
  }
]
```

#### 確認ステップ
- [ ] ユーザー提供のHTMLがサニタイズされている
- [ ] CSPヘッダーが設定されている
- [ ] バリデートされていない動的コンテンツレンダリングがない
- [ ] Reactの組み込みXSS保護を使用

### 6. CSRF保護

#### CSRFトークン
```typescript
import { csrf } from '@/lib/csrf'

export async function POST(request: Request) {
  const token = request.headers.get('X-CSRF-Token')

  if (!csrf.verify(token)) {
    return NextResponse.json(
      { error: 'Invalid CSRF token' },
      { status: 403 }
    )
  }

  // リクエストを処理
}
```

#### SameSiteクッキー
```typescript
res.setHeader('Set-Cookie',
  `session=${sessionId}; HttpOnly; Secure; SameSite=Strict`)
```

#### 確認ステップ
- [ ] 状態変更操作にCSRFトークン
- [ ] すべてのクッキーでSameSite=Strict
- [ ] 二重送信クッキーパターンが実装されている

### 7. レート制限

#### APIレート制限
```typescript
import rateLimit from 'express-rate-limit'

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // ウィンドウごとに100リクエスト
  message: 'Too many requests'
})

// ルートに適用
app.use('/api/', limiter)
```

#### 高コスト操作
```typescript
// 検索に対するアグレッシブなレート制限
const searchLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 10, // 1分あたり10リクエスト
  message: 'Too many search requests'
})

app.use('/api/search', searchLimiter)
```

#### 確認ステップ
- [ ] すべてのAPIエンドポイントでレート制限
- [ ] 高コスト操作により厳しい制限
- [ ] IPベースのレート制限
- [ ] ユーザーベースのレート制限（認証済み）

### 8. 機密データの露出

#### ロギング
```typescript
// ❌ 間違い: 機密データのログ
console.log('User login:', { email, password })
console.log('Payment:', { cardNumber, cvv })

// ✅ 正しい: 機密データの編集
console.log('User login:', { email, userId })
console.log('Payment:', { last4: card.last4, userId })
```

#### エラーメッセージ
```typescript
// ❌ 間違い: 内部詳細の露出
catch (error) {
  return NextResponse.json(
    { error: error.message, stack: error.stack },
    { status: 500 }
  )
}

// ✅ 正しい: 汎用エラーメッセージ
catch (error) {
  console.error('Internal error:', error)
  return NextResponse.json(
    { error: 'An error occurred. Please try again.' },
    { status: 500 }
  )
}
```

#### 確認ステップ
- [ ] ログにパスワード、トークン、シークレットがない
- [ ] ユーザー向けエラーメッセージが汎用的
- [ ] 詳細エラーはサーバーログのみ
- [ ] スタックトレースがユーザーに露出しない

### 9. ブロックチェーンセキュリティ（Solana）

#### ウォレット検証
```typescript
import { verify } from '@solana/web3.js'

async function verifyWalletOwnership(
  publicKey: string,
  signature: string,
  message: string
) {
  try {
    const isValid = verify(
      Buffer.from(message),
      Buffer.from(signature, 'base64'),
      Buffer.from(publicKey, 'base64')
    )
    return isValid
  } catch (error) {
    return false
  }
}
```

#### トランザクション検証
```typescript
async function verifyTransaction(transaction: Transaction) {
  // 受取人を検証
  if (transaction.to !== expectedRecipient) {
    throw new Error('Invalid recipient')
  }

  // 金額を検証
  if (transaction.amount > maxAmount) {
    throw new Error('Amount exceeds limit')
  }

  // ユーザーの残高を確認
  const balance = await getBalance(transaction.from)
  if (balance < transaction.amount) {
    throw new Error('Insufficient balance')
  }

  return true
}
```

#### 確認ステップ
- [ ] ウォレット署名が検証されている
- [ ] トランザクション詳細がバリデートされている
- [ ] トランザクション前に残高チェック
- [ ] ブラインドなトランザクション署名がない

### 10. 依存関係セキュリティ

#### 定期的なアップデート
```bash
# 脆弱性をチェック
npm audit

# 自動修正可能な問題を修正
npm audit fix

# 依存関係をアップデート
npm update

# 古いパッケージをチェック
npm outdated
```

#### ロックファイル
```bash
# 常にロックファイルをコミット
git add package-lock.json

# CI/CDで再現可能なビルドに使用
npm ci  # npm installの代わりに
```

#### 確認ステップ
- [ ] 依存関係が最新
- [ ] 既知の脆弱性がない（npm auditクリーン）
- [ ] ロックファイルがコミットされている
- [ ] GitHubでDependabotが有効
- [ ] 定期的なセキュリティアップデート

## セキュリティテスト

### 自動セキュリティテスト
```typescript
// 認証をテスト
test('requires authentication', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

// 認可をテスト
test('requires admin role', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${userToken}` }
  })
  expect(response.status).toBe(403)
})

// 入力バリデーションをテスト
test('rejects invalid input', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    body: JSON.stringify({ email: 'not-an-email' })
  })
  expect(response.status).toBe(400)
})

// レート制限をテスト
test('enforces rate limits', async () => {
  const requests = Array(101).fill(null).map(() =>
    fetch('/api/endpoint')
  )

  const responses = await Promise.all(requests)
  const tooManyRequests = responses.filter(r => r.status === 429)

  expect(tooManyRequests.length).toBeGreaterThan(0)
})
```

## 本番デプロイ前セキュリティチェックリスト

本番デプロイ前に必ず確認：

- [ ] **シークレット**: ハードコードされたシークレットがなく、すべてが環境変数
- [ ] **入力バリデーション**: すべてのユーザー入力がバリデートされている
- [ ] **SQLインジェクション**: すべてのクエリがパラメータ化されている
- [ ] **XSS**: ユーザーコンテンツがサニタイズされている
- [ ] **CSRF**: 保護が有効
- [ ] **認証**: 適切なトークン処理
- [ ] **認可**: ロールチェックが配置されている
- [ ] **レート制限**: すべてのエンドポイントで有効
- [ ] **HTTPS**: 本番で強制
- [ ] **セキュリティヘッダー**: CSP、X-Frame-Optionsが設定
- [ ] **エラー処理**: エラーに機密データがない
- [ ] **ロギング**: 機密データがログに記録されない
- [ ] **依存関係**: 最新で脆弱性がない
- [ ] **Row Level Security**: Supabaseで有効
- [ ] **CORS**: 適切に設定
- [ ] **ファイルアップロード**: バリデート済み（サイズ、型）
- [ ] **ウォレット署名**: 検証済み（ブロックチェーンの場合）

## リソース

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Next.js Security](https://nextjs.org/docs/security)
- [Supabase Security](https://supabase.com/docs/guides/auth)
- [Web Security Academy](https://portswigger.net/web-security)

---

**覚えておくこと**: セキュリティはオプションではありません。1つの脆弱性がプラットフォーム全体を危険にさらす可能性があります。迷った時は、安全側に倒すこと。
