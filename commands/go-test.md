---
description: GoのTDDワークフローを強制します。まずテーブル駆動テストを書き、その後実装します。go test -coverで80%以上のカバレッジを検証します。
---

# Go TDDコマンド

このコマンドは、イディオマティックなGoテストパターンを使用してGoコードのテスト駆動開発方法論を強制します。

## このコマンドの機能

1. **型/インターフェースの定義**: まず関数シグネチャをスキャフォールド
2. **テーブル駆動テストの作成**: 包括的なテストケースを作成（RED）
3. **テストの実行**: テストが正しい理由で失敗することを検証
4. **コードの実装**: 成功するための最小限のコードを作成（GREEN）
5. **リファクタリング**: テストを緑に保ちながら改善
6. **カバレッジの確認**: 80%以上のカバレッジを確保

## 使用するタイミング

以下の場合に `/go-test` を使用：
- 新しいGo関数を実装するとき
- 既存コードにテストカバレッジを追加するとき
- バグ修正時（まず失敗するテストを作成）
- 重要なビジネスロジックを構築するとき
- GoでのTDDワークフローを学ぶとき

## TDDサイクル

```
RED     → 失敗するテーブル駆動テストを作成
GREEN   → 成功するための最小限のコードを実装
REFACTOR → コードを改善、テストは緑のまま
REPEAT  → 次のテストケースへ
```

## セッション例

```text
ユーザー: /go-test メールアドレスを検証する関数が必要です

エージェント:
# TDDセッション: メールバリデーター

## ステップ1: インターフェースの定義

```go
// validator/email.go
package validator

// ValidateEmail は与えられた文字列が有効なメールアドレスかどうかをチェックします。
// 有効な場合はnilを返し、そうでなければ問題を説明するエラーを返します。
func ValidateEmail(email string) error {
    panic("not implemented")
}
```

## ステップ2: テーブル駆動テストの作成（RED）

```go
// validator/email_test.go
package validator

import (
    "testing"
)

func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        // 有効なメール
        {"シンプルなメール", "user@example.com", false},
        {"サブドメイン付き", "user@mail.example.com", false},
        {"プラス付き", "user+tag@example.com", false},
        {"ドット付き", "first.last@example.com", false},

        // 無効なメール
        {"空文字列", "", true},
        {"@記号なし", "userexample.com", true},
        {"ドメインなし", "user@", true},
        {"ローカル部なし", "@example.com", true},
        {"二重@", "user@@example.com", true},
        {"スペース", "user @example.com", true},
        {"TLDなし", "user@example", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if tt.wantErr && err == nil {
                t.Errorf("ValidateEmail(%q) = nil; want error", tt.email)
            }
            if !tt.wantErr && err != nil {
                t.Errorf("ValidateEmail(%q) = %v; want nil", tt.email, err)
            }
        })
    }
}
```

## ステップ3: テストを実行 - 失敗を確認

```bash
$ go test ./validator/...

--- FAIL: TestValidateEmail (0.00s)
    --- FAIL: TestValidateEmail/シンプルなメール (0.00s)
        panic: not implemented

FAIL
```

✓ テストは期待通り失敗（パニック）。

## ステップ4: 最小限のコードを実装（GREEN）

```go
// validator/email.go
package validator

import (
    "errors"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

var (
    ErrEmailEmpty   = errors.New("email cannot be empty")
    ErrEmailInvalid = errors.New("email format is invalid")
)

func ValidateEmail(email string) error {
    if email == "" {
        return ErrEmailEmpty
    }
    if !emailRegex.MatchString(email) {
        return ErrEmailInvalid
    }
    return nil
}
```

## ステップ5: テストを実行 - 成功を確認

```bash
$ go test ./validator/...

PASS
ok      project/validator    0.003s
```

✓ すべてのテストが成功！

## ステップ6: カバレッジの確認

```bash
$ go test -cover ./validator/...

PASS
coverage: 100.0% of statements
ok      project/validator    0.003s
```

✓ カバレッジ: 100%

## TDD完了！
```

## テストパターン

### テーブル駆動テスト
```go
tests := []struct {
    name     string
    input    InputType
    want     OutputType
    wantErr  bool
}{
    {"ケース1", input1, want1, false},
    {"ケース2", input2, want2, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Function(tt.input)
        // アサーション
    })
}
```

### 並列テスト
```go
for _, tt := range tests {
    tt := tt // キャプチャ
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // テスト本体
    })
}
```

### テストヘルパー
```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db := createDB()
    t.Cleanup(func() { db.Close() })
    return db
}
```

## カバレッジコマンド

```bash
# 基本カバレッジ
go test -cover ./...

# カバレッジプロファイル
go test -coverprofile=coverage.out ./...

# ブラウザで表示
go tool cover -html=coverage.out

# 関数別カバレッジ
go tool cover -func=coverage.out

# 競合検出付き
go test -race -cover ./...
```

## カバレッジ目標

| コードタイプ | 目標 |
|-----------|--------|
| 重要なビジネスロジック | 100% |
| 公開API | 90%以上 |
| 一般コード | 80%以上 |
| 生成コード | 除外 |

## TDDベストプラクティス

**推奨:**
- テストを最初に、実装前に作成
- 変更ごとにテストを実行
- 包括的なカバレッジのためにテーブル駆動テストを使用
- 実装の詳細ではなく動作をテスト
- エッジケースを含める（空、nil、最大値）

**非推奨:**
- テスト前に実装を作成
- REDフェーズをスキップ
- プライベート関数を直接テスト
- テストで `time.Sleep` を使用
- フレイキーテストを無視

## 関連コマンド

- `/go-build` - ビルドエラーを修正
- `/go-review` - 実装後にコードをレビュー
- `/verify` - 完全な検証ループを実行

## 関連

- スキル: `skills/golang-testing/`
- スキル: `skills/tdd-workflow/`
