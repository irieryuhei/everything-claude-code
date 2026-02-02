# プロジェクトCLAUDE.mdの例

これはプロジェクトレベルのCLAUDE.mdファイルの例です。プロジェクトのルートに配置してください。

## プロジェクト概要

[プロジェクトの簡単な説明 - 何をするか、技術スタック]

## 重要なルール

### 1. コード構成

- 少数の大きなファイルより多数の小さなファイル
- 高凝集、低結合
- 通常200-400行、ファイルあたり最大800行
- タイプ別ではなく機能/ドメイン別に整理

### 2. コードスタイル

- コード、コメント、ドキュメントに絵文字なし
- 常に不変性 - オブジェクトや配列を変更しない
- 本番コードにconsole.logなし
- try/catchによる適切なエラー処理
- Zodまたは類似のもので入力検証

### 3. テスト

- TDD：最初にテストを書く
- 最低80%カバレッジ
- ユーティリティのユニットテスト
- APIの統合テスト
- 重要なフローのE2Eテスト

### 4. セキュリティ

- ハードコードされたシークレットなし
- 機密データには環境変数
- すべてのユーザー入力を検証
- パラメータ化されたクエリのみ
- CSRF保護を有効化

## ファイル構造

```
src/
|-- app/              # Next.js appルーター
|-- components/       # 再利用可能なUIコンポーネント
|-- hooks/            # カスタムReactフック
|-- lib/              # ユーティリティライブラリ
|-- types/            # TypeScript定義
```

## 主要パターン

### APIレスポンス形式

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
}
```

### エラー処理

```typescript
try {
  const result = await operation()
  return { success: true, data: result }
} catch (error) {
  console.error('Operation failed:', error)
  return { success: false, error: 'ユーザーフレンドリーなメッセージ' }
}
```

## 環境変数

```bash
# 必須
DATABASE_URL=
API_KEY=

# オプション
DEBUG=false
```

## 利用可能なコマンド

- `/tdd` - テスト駆動開発ワークフロー
- `/plan` - 実装計画を作成
- `/code-review` - コード品質をレビュー
- `/build-fix` - ビルドエラーを修正

## Gitワークフロー

- 従来型コミット：`feat:`、`fix:`、`refactor:`、`docs:`、`test:`
- mainに直接コミットしない
- PRにはレビューが必要
- マージ前にすべてのテストが通過する必要がある
