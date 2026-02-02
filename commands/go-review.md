---
description: Go言語のイディオマティックなパターン、並行性の安全性、エラーハンドリング、セキュリティに関する包括的なGoコードレビュー。go-reviewerエージェントを呼び出します。
---

# Goコードレビュー

このコマンドは**go-reviewer**エージェントを呼び出して、Go固有の包括的なコードレビューを行います。

## このコマンドの機能

1. **Go変更の特定**: `git diff` で変更された `.go` ファイルを検出
2. **静的解析の実行**: `go vet`、`staticcheck`、`golangci-lint` を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、競合状態をチェック
4. **並行性レビュー**: goroutineの安全性、チャネルの使用、mutexパターンを分析
5. **イディオマティックGoチェック**: コードがGo規約とベストプラクティスに従っているか検証
6. **レポート生成**: 問題を重要度別にカテゴリ分け

## 使用するタイミング

以下の場合に `/go-review` を使用：
- Goコードを書いたり変更した後
- Go変更をコミットする前
- Goコードのプルリクエストをレビューするとき
- 新しいGoコードベースにオンボーディングするとき
- イディオマティックなGoパターンを学ぶとき

## レビューカテゴリ

### クリティカル（必須修正）
- SQL/コマンドインジェクションの脆弱性
- 同期なしの競合状態
- goroutineリーク
- ハードコードされた認証情報
- 安全でないポインタの使用
- 重要なパスでのエラー無視

### 高（修正推奨）
- コンテキストを伴うエラーラッピングの欠落
- エラーを返す代わりにパニック
- コンテキストが伝播されていない
- デッドロックを引き起こすバッファなしチャネル
- インターフェースが満たされていないエラー
- mutex保護の欠落

### 中（検討）
- イディオマティックでないコードパターン
- エクスポートにgodocコメントがない
- 非効率な文字列連結
- スライスの事前割り当てがない
- テーブル駆動テストが使用されていない

## 実行される自動チェック

```bash
# 静的解析
go vet ./...

# 高度なチェック（インストールされている場合）
staticcheck ./...
golangci-lint run

# 競合検出
go build -race ./...

# セキュリティ脆弱性
govulncheck ./...
```

## 使用例

```text
ユーザー: /go-review

エージェント:
# Goコードレビューレポート

## レビューされたファイル
- internal/handler/user.go（変更）
- internal/service/auth.go（変更）

## 静的解析結果
✓ go vet: 問題なし
✓ staticcheck: 問題なし

## 発見された問題

[クリティカル] 競合状態
ファイル: internal/service/auth.go:45
問題: 同期なしで共有マップにアクセス
```go
var cache = map[string]*Session{}  // 並行アクセス！

func GetSession(id string) *Session {
    return cache[id]  // 競合状態
}
```
修正: sync.RWMutex または sync.Map を使用
```go
var (
    cache   = map[string]*Session{}
    cacheMu sync.RWMutex
)

func GetSession(id string) *Session {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[id]
}
```

[高] エラーコンテキストの欠落
ファイル: internal/handler/user.go:28
問題: コンテキストなしでエラーを返している
```go
return err  // コンテキストなし
```
修正: コンテキストでラップ
```go
return fmt.Errorf("get user %s: %w", userID, err)
```

## サマリー
- クリティカル: 1
- 高: 1
- 中: 0

推奨: ❌ クリティカルな問題が修正されるまでマージをブロック
```

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| ✅ 承認 | クリティカルまたは高の問題なし |
| ⚠️ 警告 | 中の問題のみ（注意してマージ） |
| ❌ ブロック | クリティカルまたは高の問題あり |

## 他のコマンドとの統合

- まず `/go-test` を使用してテストが成功することを確認
- ビルドエラーが発生した場合は `/go-build` を使用
- コミット前に `/go-review` を使用
- Go固有でない懸念事項には `/code-review` を使用

## 関連

- エージェント: `agents/go-reviewer.md`
- スキル: `skills/golang-patterns/`、`skills/golang-testing/`
