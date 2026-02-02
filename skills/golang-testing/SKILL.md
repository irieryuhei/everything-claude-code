---
name: golang-testing
description: テーブル駆動テスト、サブテスト、ベンチマーク、ファジング、テストカバレッジを含むGoテストパターン。慣用的なGoプラクティスによるTDD方法論に従う。
---

# Goテストパターン

TDD方法論に従った、信頼性が高く保守可能なテストを書くための包括的なGoテストパターン。

## 発動条件

- 新しいGo関数やメソッドの作成
- 既存コードへのテストカバレッジ追加
- パフォーマンスクリティカルなコードのベンチマーク作成
- 入力バリデーションのファズテスト実装
- GoプロジェクトでのTDDワークフロー

## GoのためのTDDワークフロー

### RED-GREEN-REFACTORサイクル

```
RED     → 最初に失敗するテストを書く
GREEN   → テストをパスする最小限のコードを書く
REFACTOR → テストをグリーンに保ちながらコードを改善
REPEAT  → 次の要件で繰り返す
```

### GoでのステップバイステップTDD

```go
// ステップ1: インターフェース/シグネチャを定義
// calculator.go
package calculator

func Add(a, b int) int {
    panic("not implemented") // プレースホルダー
}

// ステップ2: 失敗するテストを書く（RED）
// calculator_test.go
package calculator

import "testing"

func TestAdd(t *testing.T) {
    got := Add(2, 3)
    want := 5
    if got != want {
        t.Errorf("Add(2, 3) = %d; want %d", got, want)
    }
}

// ステップ3: テストを実行 - FAILを確認
// $ go test
// --- FAIL: TestAdd (0.00s)
// panic: not implemented

// ステップ4: 最小限のコードを実装（GREEN）
func Add(a, b int) int {
    return a + b
}

// ステップ5: テストを実行 - PASSを確認
// $ go test
// PASS

// ステップ6: 必要に応じてリファクタリング、テストが引き続きパスすることを確認
```

## テーブル駆動テスト

Goテストの標準パターン。最小限のコードで包括的なカバレッジを実現。

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"zero values", 0, 0, 0},
        {"mixed signs", -1, 1, 0},
        {"large numbers", 1000000, 2000000, 3000000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d",
                    tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### エラーケース付きテーブル駆動テスト

```go
func TestParseConfig(t *testing.T) {
    tests := []struct {
        name    string
        input   string
        want    *Config
        wantErr bool
    }{
        {
            name:  "valid config",
            input: `{"host": "localhost", "port": 8080}`,
            want:  &Config{Host: "localhost", Port: 8080},
        },
        {
            name:    "invalid JSON",
            input:   `{invalid}`,
            wantErr: true,
        },
        {
            name:    "empty input",
            input:   "",
            wantErr: true,
        },
        {
            name:  "minimal config",
            input: `{}`,
            want:  &Config{}, // ゼロ値config
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := ParseConfig(tt.input)

            if tt.wantErr {
                if err == nil {
                    t.Error("expected error, got nil")
                }
                return
            }

            if err != nil {
                t.Fatalf("unexpected error: %v", err)
            }

            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("got %+v; want %+v", got, tt.want)
            }
        })
    }
}
```

## サブテストとサブベンチマーク

### 関連テストの整理

```go
func TestUser(t *testing.T) {
    // すべてのサブテストで共有されるセットアップ
    db := setupTestDB(t)

    t.Run("Create", func(t *testing.T) {
        user := &User{Name: "Alice"}
        err := db.CreateUser(user)
        if err != nil {
            t.Fatalf("CreateUser failed: %v", err)
        }
        if user.ID == "" {
            t.Error("expected user ID to be set")
        }
    })

    t.Run("Get", func(t *testing.T) {
        user, err := db.GetUser("alice-id")
        if err != nil {
            t.Fatalf("GetUser failed: %v", err)
        }
        if user.Name != "Alice" {
            t.Errorf("got name %q; want %q", user.Name, "Alice")
        }
    })

    t.Run("Update", func(t *testing.T) {
        // ...
    })

    t.Run("Delete", func(t *testing.T) {
        // ...
    })
}
```

### 並列サブテスト

```go
func TestParallel(t *testing.T) {
    tests := []struct {
        name  string
        input string
    }{
        {"case1", "input1"},
        {"case2", "input2"},
        {"case3", "input3"},
    }

    for _, tt := range tests {
        tt := tt // レンジ変数をキャプチャ
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // サブテストを並列実行
            result := Process(tt.input)
            // アサーション...
            _ = result
        })
    }
}
```

## テストヘルパー

### ヘルパー関数

```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper() // これをヘルパー関数としてマーク

    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open database: %v", err)
    }

    // テスト終了時にクリーンアップ
    t.Cleanup(func() {
        db.Close()
    })

    // マイグレーションを実行
    if _, err := db.Exec(schema); err != nil {
        t.Fatalf("failed to create schema: %v", err)
    }

    return db
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func assertEqual[T comparable](t *testing.T, got, want T) {
    t.Helper()
    if got != want {
        t.Errorf("got %v; want %v", got, want)
    }
}
```

### 一時ファイルとディレクトリ

```go
func TestFileProcessing(t *testing.T) {
    // 一時ディレクトリを作成 - 自動的にクリーンアップ
    tmpDir := t.TempDir()

    // テストファイルを作成
    testFile := filepath.Join(tmpDir, "test.txt")
    err := os.WriteFile(testFile, []byte("test content"), 0644)
    if err != nil {
        t.Fatalf("failed to create test file: %v", err)
    }

    // テストを実行
    result, err := ProcessFile(testFile)
    if err != nil {
        t.Fatalf("ProcessFile failed: %v", err)
    }

    // アサート...
    _ = result
}
```

## ゴールデンファイル

`testdata/`に保存された期待出力ファイルに対するテスト。

```go
var update = flag.Bool("update", false, "update golden files")

func TestRender(t *testing.T) {
    tests := []struct {
        name  string
        input Template
    }{
        {"simple", Template{Name: "test"}},
        {"complex", Template{Name: "test", Items: []string{"a", "b"}}},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Render(tt.input)

            golden := filepath.Join("testdata", tt.name+".golden")

            if *update {
                // ゴールデンファイルを更新: go test -update
                err := os.WriteFile(golden, got, 0644)
                if err != nil {
                    t.Fatalf("failed to update golden file: %v", err)
                }
            }

            want, err := os.ReadFile(golden)
            if err != nil {
                t.Fatalf("failed to read golden file: %v", err)
            }

            if !bytes.Equal(got, want) {
                t.Errorf("output mismatch:\ngot:\n%s\nwant:\n%s", got, want)
            }
        })
    }
}
```

## インターフェースベースのモッキング

### インターフェースベースモッキング

```go
// 依存関係用のインターフェースを定義
type UserRepository interface {
    GetUser(id string) (*User, error)
    SaveUser(user *User) error
}

// 本番実装
type PostgresUserRepository struct {
    db *sql.DB
}

func (r *PostgresUserRepository) GetUser(id string) (*User, error) {
    // 実際のデータベースクエリ
}

// テスト用のモック実装
type MockUserRepository struct {
    GetUserFunc  func(id string) (*User, error)
    SaveUserFunc func(user *User) error
}

func (m *MockUserRepository) GetUser(id string) (*User, error) {
    return m.GetUserFunc(id)
}

func (m *MockUserRepository) SaveUser(user *User) error {
    return m.SaveUserFunc(user)
}

// モックを使用したテスト
func TestUserService(t *testing.T) {
    mock := &MockUserRepository{
        GetUserFunc: func(id string) (*User, error) {
            if id == "123" {
                return &User{ID: "123", Name: "Alice"}, nil
            }
            return nil, ErrNotFound
        },
    }

    service := NewUserService(mock)

    user, err := service.GetUserProfile("123")
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("got name %q; want %q", user.Name, "Alice")
    }
}
```

## ベンチマーク

### 基本ベンチマーク

```go
func BenchmarkProcess(b *testing.B) {
    data := generateTestData(1000)
    b.ResetTimer() // セットアップ時間をカウントしない

    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

// 実行: go test -bench=BenchmarkProcess -benchmem
// 出力: BenchmarkProcess-8   10000   105234 ns/op   4096 B/op   10 allocs/op
```

### 異なるサイズでのベンチマーク

```go
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000, 100000}

    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := generateRandomSlice(size)
            b.ResetTimer()

            for i := 0; i < b.N; i++ {
                // 既にソートされたデータのソートを避けるためにコピー
                tmp := make([]int, len(data))
                copy(tmp, data)
                sort.Ints(tmp)
            }
        })
    }
}
```

### メモリアロケーションベンチマーク

```go
func BenchmarkStringConcat(b *testing.B) {
    parts := []string{"hello", "world", "foo", "bar", "baz"}

    b.Run("plus", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var s string
            for _, p := range parts {
                s += p
            }
            _ = s
        }
    })

    b.Run("builder", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var sb strings.Builder
            for _, p := range parts {
                sb.WriteString(p)
            }
            _ = sb.String()
        }
    })

    b.Run("join", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            _ = strings.Join(parts, "")
        }
    })
}
```

## ファジング（Go 1.18+）

### 基本ファズテスト

```go
func FuzzParseJSON(f *testing.F) {
    // シードコーパスを追加
    f.Add(`{"name": "test"}`)
    f.Add(`{"count": 123}`)
    f.Add(`[]`)
    f.Add(`""`)

    f.Fuzz(func(t *testing.T, input string) {
        var result map[string]interface{}
        err := json.Unmarshal([]byte(input), &result)

        if err != nil {
            // ランダム入力では無効なJSONが期待される
            return
        }

        // パースが成功した場合、再エンコードは動作すべき
        _, err = json.Marshal(result)
        if err != nil {
            t.Errorf("Marshal failed after successful Unmarshal: %v", err)
        }
    })
}

// 実行: go test -fuzz=FuzzParseJSON -fuzztime=30s
```

### 複数入力のファズテスト

```go
func FuzzCompare(f *testing.F) {
    f.Add("hello", "world")
    f.Add("", "")
    f.Add("abc", "abc")

    f.Fuzz(func(t *testing.T, a, b string) {
        result := Compare(a, b)

        // プロパティ: Compare(a, a)は常に0であるべき
        if a == b && result != 0 {
            t.Errorf("Compare(%q, %q) = %d; want 0", a, b, result)
        }

        // プロパティ: Compare(a, b)とCompare(b, a)は逆符号であるべき
        reverse := Compare(b, a)
        if (result > 0 && reverse >= 0) || (result < 0 && reverse <= 0) {
            if result != 0 || reverse != 0 {
                t.Errorf("Compare(%q, %q) = %d, Compare(%q, %q) = %d; inconsistent",
                    a, b, result, b, a, reverse)
            }
        }
    })
}
```

## テストカバレッジ

### カバレッジの実行

```bash
# 基本カバレッジ
go test -cover ./...

# カバレッジプロファイルを生成
go test -coverprofile=coverage.out ./...

# ブラウザでカバレッジを表示
go tool cover -html=coverage.out

# 関数別カバレッジを表示
go tool cover -func=coverage.out

# レース検出付きカバレッジ
go test -race -coverprofile=coverage.out ./...
```

### カバレッジ目標

| コードタイプ | 目標 |
|------------|------|
| 重要なビジネスロジック | 100% |
| パブリックAPI | 90%以上 |
| 一般的なコード | 80%以上 |
| 生成されたコード | 除外 |

### 生成コードをカバレッジから除外

```go
//go:generate mockgen -source=interface.go -destination=mock_interface.go

// カバレッジプロファイルで、ビルドタグで除外:
// go test -cover -tags=!generate ./...
```

## HTTPハンドラーテスト

```go
func TestHealthHandler(t *testing.T) {
    // リクエストを作成
    req := httptest.NewRequest(http.MethodGet, "/health", nil)
    w := httptest.NewRecorder()

    // ハンドラーを呼び出し
    HealthHandler(w, req)

    // レスポンスをチェック
    resp := w.Result()
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        t.Errorf("got status %d; want %d", resp.StatusCode, http.StatusOK)
    }

    body, _ := io.ReadAll(resp.Body)
    if string(body) != "OK" {
        t.Errorf("got body %q; want %q", body, "OK")
    }
}

func TestAPIHandler(t *testing.T) {
    tests := []struct {
        name       string
        method     string
        path       string
        body       string
        wantStatus int
        wantBody   string
    }{
        {
            name:       "get user",
            method:     http.MethodGet,
            path:       "/users/123",
            wantStatus: http.StatusOK,
            wantBody:   `{"id":"123","name":"Alice"}`,
        },
        {
            name:       "not found",
            method:     http.MethodGet,
            path:       "/users/999",
            wantStatus: http.StatusNotFound,
        },
        {
            name:       "create user",
            method:     http.MethodPost,
            path:       "/users",
            body:       `{"name":"Bob"}`,
            wantStatus: http.StatusCreated,
        },
    }

    handler := NewAPIHandler()

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            var body io.Reader
            if tt.body != "" {
                body = strings.NewReader(tt.body)
            }

            req := httptest.NewRequest(tt.method, tt.path, body)
            req.Header.Set("Content-Type", "application/json")
            w := httptest.NewRecorder()

            handler.ServeHTTP(w, req)

            if w.Code != tt.wantStatus {
                t.Errorf("got status %d; want %d", w.Code, tt.wantStatus)
            }

            if tt.wantBody != "" && w.Body.String() != tt.wantBody {
                t.Errorf("got body %q; want %q", w.Body.String(), tt.wantBody)
            }
        })
    }
}
```

## テストコマンド

```bash
# すべてのテストを実行
go test ./...

# 詳細出力付きでテスト実行
go test -v ./...

# 特定のテストを実行
go test -run TestAdd ./...

# パターンにマッチするテストを実行
go test -run "TestUser/Create" ./...

# レースディテクター付きでテスト実行
go test -race ./...

# カバレッジ付きでテスト実行
go test -cover -coverprofile=coverage.out ./...

# 短いテストのみ実行
go test -short ./...

# タイムアウト付きでテスト実行
go test -timeout 30s ./...

# ベンチマークを実行
go test -bench=. -benchmem ./...

# ファジングを実行
go test -fuzz=FuzzParse -fuzztime=30s ./...

# テスト実行回数をカウント（フレーキーテスト検出用）
go test -count=10 ./...
```

## ベストプラクティス

**やるべきこと:**
- テストを先に書く（TDD）
- 包括的なカバレッジのためにテーブル駆動テストを使用
- 実装ではなく動作をテスト
- ヘルパー関数で`t.Helper()`を使用
- 独立したテストには`t.Parallel()`を使用
- `t.Cleanup()`でリソースをクリーンアップ
- シナリオを説明する意味のあるテスト名を使用

**やらないこと:**
- プライベート関数を直接テストしない（パブリックAPIを通じてテスト）
- テストで`time.Sleep()`を使用しない（チャネルまたは条件を使用）
- フレーキーテストを無視しない（修正または削除）
- すべてをモックしない（可能な場合は統合テストを優先）
- エラーパスのテストをスキップしない

## CI/CDとの統合

```yaml
# GitHub Actionsの例
test:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v5
      with:
        go-version: '1.22'

    - name: テストを実行
      run: go test -race -coverprofile=coverage.out ./...

    - name: カバレッジをチェック
      run: |
        go tool cover -func=coverage.out | grep total | awk '{print $3}' | \
        awk -F'%' '{if ($1 < 80) exit 1}'
```

**覚えておくこと**: テストはドキュメントです。コードがどのように使用されるべきかを示します。明確に書き、最新の状態に保ちましょう。
