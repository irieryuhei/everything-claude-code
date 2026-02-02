---
name: tdd-workflow
description: 新機能の作成、バグ修正、コードリファクタリング時に使用。ユニット、統合、E2Eテストを含む80%以上のカバレッジでテスト駆動開発を強制。
---

# テスト駆動開発ワークフロー

このスキルは、すべてのコード開発が包括的なテストカバレッジを伴うTDD原則に従うことを保証します。

## 使用タイミング

- 新機能や機能性を書く時
- バグや問題を修正する時
- 既存コードをリファクタリングする時
- APIエンドポイントを追加する時
- 新しいコンポーネントを作成する時

## 基本原則

### 1. コードより先にテスト
常に最初にテストを書き、次にテストをパスさせるコードを実装します。

### 2. カバレッジ要件
- 最小80%カバレッジ（ユニット + 統合 + E2E）
- すべてのエッジケースをカバー
- エラーシナリオをテスト
- 境界条件を検証

### 3. テストタイプ

#### ユニットテスト
- 個々の関数やユーティリティ
- コンポーネントロジック
- 純粋関数
- ヘルパーやユーティリティ

#### 統合テスト
- APIエンドポイント
- データベース操作
- サービス間連携
- 外部API呼び出し

#### E2Eテスト（Playwright）
- 重要なユーザーフロー
- 完全なワークフロー
- ブラウザ自動化
- UI操作

## TDDワークフローステップ

### ステップ1: ユーザージャーニーを書く
```
[役割]として、[アクション]をしたい、[利益]のために

例：
ユーザーとして、マーケットをセマンティック検索したい、
正確なキーワードがなくても関連するマーケットを見つけられるように。
```

### ステップ2: テストケースを生成
各ユーザージャーニーに対して包括的なテストケースを作成：

```typescript
describe('Semantic Search', () => {
  it('returns relevant markets for query', async () => {
    // テスト実装
  })

  it('handles empty query gracefully', async () => {
    // エッジケースをテスト
  })

  it('falls back to substring search when Redis unavailable', async () => {
    // フォールバック動作をテスト
  })

  it('sorts results by similarity score', async () => {
    // ソートロジックをテスト
  })
})
```

### ステップ3: テストを実行（失敗するはず）
```bash
npm test
# テストは失敗するはず - まだ実装していないので
```

### ステップ4: コードを実装
テストをパスさせる最小限のコードを書く：

```typescript
// テストによってガイドされた実装
export async function searchMarkets(query: string) {
  // ここに実装
}
```

### ステップ5: テストを再実行
```bash
npm test
# テストはパスするはず
```

### ステップ6: リファクタリング
テストをグリーンに保ちながらコード品質を改善：
- 重複を削除
- 命名を改善
- パフォーマンスを最適化
- 可読性を向上

### ステップ7: カバレッジを確認
```bash
npm run test:coverage
# 80%以上のカバレッジを達成したか確認
```

## テストパターン

### ユニットテストパターン（Jest/Vitest）
```typescript
import { render, screen, fireEvent } from '@testing-library/react'
import { Button } from './Button'

describe('Button Component', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByText('Click me')).toBeInTheDocument()
  })

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn()
    render(<Button onClick={handleClick}>Click</Button>)

    fireEvent.click(screen.getByRole('button'))

    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click</Button>)
    expect(screen.getByRole('button')).toBeDisabled()
  })
})
```

### API統合テストパターン
```typescript
import { NextRequest } from 'next/server'
import { GET } from './route'

describe('GET /api/markets', () => {
  it('returns markets successfully', async () => {
    const request = new NextRequest('http://localhost/api/markets')
    const response = await GET(request)
    const data = await response.json()

    expect(response.status).toBe(200)
    expect(data.success).toBe(true)
    expect(Array.isArray(data.data)).toBe(true)
  })

  it('validates query parameters', async () => {
    const request = new NextRequest('http://localhost/api/markets?limit=invalid')
    const response = await GET(request)

    expect(response.status).toBe(400)
  })

  it('handles database errors gracefully', async () => {
    // データベース障害をモック
    const request = new NextRequest('http://localhost/api/markets')
    // エラー処理をテスト
  })
})
```

### E2Eテストパターン（Playwright）
```typescript
import { test, expect } from '@playwright/test'

test('user can search and filter markets', async ({ page }) => {
  // マーケットページに移動
  await page.goto('/')
  await page.click('a[href="/markets"]')

  // ページが読み込まれたことを確認
  await expect(page.locator('h1')).toContainText('Markets')

  // マーケットを検索
  await page.fill('input[placeholder="Search markets"]', 'election')

  // デバウンスと結果を待機
  await page.waitForTimeout(600)

  // 検索結果が表示されたことを確認
  const results = page.locator('[data-testid="market-card"]')
  await expect(results).toHaveCount(5, { timeout: 5000 })

  // 結果に検索語が含まれることを確認
  const firstResult = results.first()
  await expect(firstResult).toContainText('election', { ignoreCase: true })

  // ステータスでフィルター
  await page.click('button:has-text("Active")')

  // フィルターされた結果を確認
  await expect(results).toHaveCount(3)
})

test('user can create a new market', async ({ page }) => {
  // 最初にログイン
  await page.goto('/creator-dashboard')

  // マーケット作成フォームを入力
  await page.fill('input[name="name"]', 'Test Market')
  await page.fill('textarea[name="description"]', 'Test description')
  await page.fill('input[name="endDate"]', '2025-12-31')

  // フォームを送信
  await page.click('button[type="submit"]')

  // 成功メッセージを確認
  await expect(page.locator('text=Market created successfully')).toBeVisible()

  // マーケットページへのリダイレクトを確認
  await expect(page).toHaveURL(/\/markets\/test-market/)
})
```

## テストファイル構成

```
src/
├── components/
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx          # ユニットテスト
│   │   └── Button.stories.tsx       # Storybook
│   └── MarketCard/
│       ├── MarketCard.tsx
│       └── MarketCard.test.tsx
├── app/
│   └── api/
│       └── markets/
│           ├── route.ts
│           └── route.test.ts         # 統合テスト
└── e2e/
    ├── markets.spec.ts               # E2Eテスト
    ├── trading.spec.ts
    └── auth.spec.ts
```

## 外部サービスのモック

### Supabaseモック
```typescript
jest.mock('@/lib/supabase', () => ({
  supabase: {
    from: jest.fn(() => ({
      select: jest.fn(() => ({
        eq: jest.fn(() => Promise.resolve({
          data: [{ id: 1, name: 'Test Market' }],
          error: null
        }))
      }))
    }))
  }
}))
```

### Redisモック
```typescript
jest.mock('@/lib/redis', () => ({
  searchMarketsByVector: jest.fn(() => Promise.resolve([
    { slug: 'test-market', similarity_score: 0.95 }
  ])),
  checkRedisHealth: jest.fn(() => Promise.resolve({ connected: true }))
}))
```

### OpenAIモック
```typescript
jest.mock('@/lib/openai', () => ({
  generateEmbedding: jest.fn(() => Promise.resolve(
    new Array(1536).fill(0.1) // 1536次元のモック埋め込み
  ))
}))
```

## テストカバレッジ確認

### カバレッジレポートを実行
```bash
npm run test:coverage
```

### カバレッジ閾値
```json
{
  "jest": {
    "coverageThresholds": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

## よくあるテストの間違い

### ❌ 間違い: 実装の詳細をテスト
```typescript
// 内部状態をテストしない
expect(component.state.count).toBe(5)
```

### ✅ 正しい: ユーザーに見える動作をテスト
```typescript
// ユーザーが見るものをテスト
expect(screen.getByText('Count: 5')).toBeInTheDocument()
```

### ❌ 間違い: 脆いセレクター
```typescript
// 壊れやすい
await page.click('.css-class-xyz')
```

### ✅ 正しい: セマンティックセレクター
```typescript
// 変更に強い
await page.click('button:has-text("Submit")')
await page.click('[data-testid="submit-button"]')
```

### ❌ 間違い: テスト分離なし
```typescript
// テストが相互依存
test('creates user', () => { /* ... */ })
test('updates same user', () => { /* 前のテストに依存 */ })
```

### ✅ 正しい: 独立したテスト
```typescript
// 各テストが独自のデータを設定
test('creates user', () => {
  const user = createTestUser()
  // テストロジック
})

test('updates user', () => {
  const user = createTestUser()
  // 更新ロジック
})
```

## 継続的テスト

### 開発中のウォッチモード
```bash
npm test -- --watch
# ファイル変更時にテストが自動実行
```

### プリコミットフック
```bash
# コミット前に毎回実行
npm test && npm run lint
```

### CI/CD統合
```yaml
# GitHub Actions
- name: Run Tests
  run: npm test -- --coverage
- name: Upload Coverage
  uses: codecov/codecov-action@v3
```

## ベストプラクティス

1. **まずテストを書く** - 常にTDD
2. **テストごとに1つのアサート** - 単一の動作に焦点
3. **説明的なテスト名** - 何をテストしているか説明
4. **Arrange-Act-Assert** - 明確なテスト構造
5. **外部依存をモック** - ユニットテストを分離
6. **エッジケースをテスト** - Null、undefined、空、大きい
7. **エラーパスをテスト** - ハッピーパスだけでなく
8. **テストを高速に保つ** - ユニットテストは各50ms未満
9. **テスト後にクリーンアップ** - 副作用なし
10. **カバレッジレポートをレビュー** - ギャップを特定

## 成功指標

- 80%以上のコードカバレッジを達成
- すべてのテストがパス（グリーン）
- スキップまたは無効化されたテストなし
- 高速なテスト実行（ユニットテストは30秒未満）
- E2Eテストが重要なユーザーフローをカバー
- テストが本番前にバグをキャッチ

---

**覚えておくこと**: テストはオプションではありません。自信を持ったリファクタリング、迅速な開発、本番の信頼性を可能にするセーフティネットです。
