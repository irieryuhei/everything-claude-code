---
description: Goビルドエラー、go vetの警告、リンターの問題を段階的に修正します。最小限の外科的修正のためにgo-build-resolverエージェントを呼び出します。
---

# Go ビルドと修正

このコマンドは**go-build-resolver**エージェントを呼び出して、最小限の変更でGoビルドエラーを段階的に修正します。

## このコマンドの機能

1. **診断を実行**: `go build`、`go vet`、`staticcheck` を実行
2. **エラーを解析**: ファイルごとにグループ化し、重要度順にソート
3. **段階的に修正**: 一度に1つのエラーを修正
4. **各修正を検証**: 変更ごとにビルドを再実行
5. **サマリーを報告**: 修正されたものと残りのものを表示

## 使用するタイミング

以下の場合に `/go-build` を使用：
- `go build ./...` がエラーで失敗する場合
- `go vet ./...` が問題を報告する場合
- `golangci-lint run` が警告を表示する場合
- モジュール依存関係が壊れている場合
- ビルドを壊す変更をプルした後

## 実行される診断コマンド

```bash
# プライマリビルドチェック
go build ./...

# 静的解析
go vet ./...

# 拡張リンティング（利用可能な場合）
staticcheck ./...
golangci-lint run

# モジュールの問題
go mod verify
go mod tidy -v
```

## セッション例

```text
ユーザー: /go-build

エージェント:
# Goビルド解決

## 初期診断

```bash
$ go build ./...
internal/service/user.go:25:15: undefined: UserRepository
internal/handler/api.go:42:9: cannot use x (type string) as type int
internal/handler/api.go:58:2: missing return at end of function
```

エラー発見: 3件

## 修正1: 未定義の識別子

ファイル: internal/service/user.go:25
エラー: undefined: UserRepository
原因: インポートの欠落

```go
// インポートを追加
import "project/internal/repository"

// 変更前
var repo UserRepository
// 変更後
var repo repository.UserRepository
```

```bash
$ go build ./...
# 残り2件のエラー
```

## 修正2: 型の不一致

ファイル: internal/handler/api.go:42
エラー: cannot use x (type string) as type int

```go
// 変更前
count := params.Get("count")
// 変更後
countStr := params.Get("count")
count, _ := strconv.Atoi(countStr)
```

```bash
$ go build ./...
# 残り1件のエラー
```

## 修正3: return文の欠落

ファイル: internal/handler/api.go:58
エラー: missing return at end of function

```go
func GetUser(id string) (*User, error) {
    if id == "" {
        return nil, ErrInvalidID
    }
    user := findUser(id)
    // 欠落していたreturnを追加
    return user, nil
}
```

```bash
$ go build ./...
# ビルド成功！
```

## 最終検証

```bash
$ go vet ./...
# 問題なし

$ go test ./...
ok      project/internal/service   0.015s
ok      project/internal/handler   0.023s
```

## サマリー

| メトリクス | カウント |
|--------|-------|
| 修正されたビルドエラー | 3 |
| 修正されたvet警告 | 0 |
| 変更されたファイル | 2 |
| 残りの問題 | 0 |

ビルドステータス: ✅ 成功
```

## よくあるエラーと修正

| エラー | 一般的な修正 |
|-------|-------------|
| `undefined: X` | インポートを追加するかタイポを修正 |
| `cannot use X as Y` | 型変換または代入を修正 |
| `missing return` | return文を追加 |
| `X does not implement Y` | 欠落しているメソッドを追加 |
| `import cycle` | パッケージを再構成 |
| `declared but not used` | 変数を削除するか使用 |
| `cannot find package` | `go get` または `go mod tidy` |

## 修正戦略

1. **まずビルドエラー** - コードはコンパイルされる必要がある
2. **次にvet警告** - 疑わしい構造を修正
3. **その後リント警告** - スタイルとベストプラクティス
4. **一度に1つの修正** - 各変更を検証
5. **最小限の変更** - リファクタリングせず、修正のみ

## 停止条件

エージェントは以下の場合に停止して報告します：
- 3回試行しても同じエラーが続く場合
- 修正がより多くのエラーを引き起こす場合
- アーキテクチャの変更が必要な場合
- 外部依存関係が不足している場合

## 関連コマンド

- `/go-test` - ビルド成功後にテストを実行
- `/go-review` - コード品質をレビュー
- `/verify` - 完全な検証ループ

## 関連

- エージェント: `agents/go-build-resolver.md`
- スキル: `skills/golang-patterns/`
