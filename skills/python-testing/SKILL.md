---
name: python-testing
description: pytest、TDD方法論、フィクスチャ、モック、パラメータ化、カバレッジ要件を使用したPythonテスト戦略。
---

# Pythonテストパターン

pytest、TDD方法論、ベストプラクティスを使用したPythonアプリケーションの包括的なテスト戦略。

## 使用タイミング

- 新しいPythonコードを書く時（TDDに従う：red、green、refactor）
- Pythonプロジェクトのテストスイートを設計する時
- Pythonテストカバレッジをレビューする時
- テストインフラストラクチャをセットアップする時

## コアテスト哲学

### テスト駆動開発（TDD）

常にTDDサイクルに従う：

1. **RED**: 望ましい動作の失敗するテストを書く
2. **GREEN**: テストをパスさせる最小限のコードを書く
3. **REFACTOR**: テストをグリーンに保ちながらコードを改善

```python
# ステップ1: 失敗するテストを書く（RED）
def test_add_numbers():
    result = add(2, 3)
    assert result == 5

# ステップ2: 最小限の実装を書く（GREEN）
def add(a, b):
    return a + b

# ステップ3: 必要に応じてリファクタリング（REFACTOR）
```

### カバレッジ要件

- **目標**: 80%以上のコードカバレッジ
- **クリティカルパス**: 100%カバレッジ必須
- `pytest --cov`でカバレッジを計測

```bash
pytest --cov=mypackage --cov-report=term-missing --cov-report=html
```

## pytest基礎

### 基本的なテスト構造

```python
import pytest

def test_addition():
    """基本的な加算をテスト。"""
    assert 2 + 2 == 4

def test_string_uppercase():
    """文字列の大文字化をテスト。"""
    text = "hello"
    assert text.upper() == "HELLO"

def test_list_append():
    """リストのappendをテスト。"""
    items = [1, 2, 3]
    items.append(4)
    assert 4 in items
    assert len(items) == 4
```

### アサーション

```python
# 等価
assert result == expected

# 非等価
assert result != unexpected

# 真偽値
assert result  # Truthy
assert not result  # Falsy
assert result is True  # 正確にTrue
assert result is False  # 正確にFalse
assert result is None  # 正確にNone

# メンバーシップ
assert item in collection
assert item not in collection

# 比較
assert result > 0
assert 0 <= result <= 100

# 型チェック
assert isinstance(result, str)

# 例外テスト（推奨アプローチ）
with pytest.raises(ValueError):
    raise ValueError("error message")

# 例外メッセージをチェック
with pytest.raises(ValueError, match="invalid input"):
    raise ValueError("invalid input provided")

# 例外属性をチェック
with pytest.raises(ValueError) as exc_info:
    raise ValueError("error message")
assert str(exc_info.value) == "error message"
```

## フィクスチャ

### 基本的なフィクスチャ使用法

```python
import pytest

@pytest.fixture
def sample_data():
    """サンプルデータを提供するフィクスチャ。"""
    return {"name": "Alice", "age": 30}

def test_sample_data(sample_data):
    """フィクスチャを使用するテスト。"""
    assert sample_data["name"] == "Alice"
    assert sample_data["age"] == 30
```

### セットアップ/ティアダウン付きフィクスチャ

```python
@pytest.fixture
def database():
    """セットアップとティアダウンを持つフィクスチャ。"""
    # セットアップ
    db = Database(":memory:")
    db.create_tables()
    db.insert_test_data()

    yield db  # テストに提供

    # ティアダウン
    db.close()

def test_database_query(database):
    """データベース操作をテスト。"""
    result = database.query("SELECT * FROM users")
    assert len(result) > 0
```

### フィクスチャスコープ

```python
# 関数スコープ（デフォルト）- 各テストで実行
@pytest.fixture
def temp_file():
    with open("temp.txt", "w") as f:
        yield f
    os.remove("temp.txt")

# モジュールスコープ - モジュールごとに1回実行
@pytest.fixture(scope="module")
def module_db():
    db = Database(":memory:")
    db.create_tables()
    yield db
    db.close()

# セッションスコープ - テストセッションごとに1回実行
@pytest.fixture(scope="session")
def shared_resource():
    resource = ExpensiveResource()
    yield resource
    resource.cleanup()
```

### パラメータ付きフィクスチャ

```python
@pytest.fixture(params=[1, 2, 3])
def number(request):
    """パラメータ化されたフィクスチャ。"""
    return request.param

def test_numbers(number):
    """テストは各パラメータで3回実行される。"""
    assert number > 0
```

### 複数フィクスチャの使用

```python
@pytest.fixture
def user():
    return User(id=1, name="Alice")

@pytest.fixture
def admin():
    return User(id=2, name="Admin", role="admin")

def test_user_admin_interaction(user, admin):
    """複数フィクスチャを使用するテスト。"""
    assert admin.can_manage(user)
```

### 自動使用フィクスチャ

```python
@pytest.fixture(autouse=True)
def reset_config():
    """各テストの前に自動的に実行。"""
    Config.reset()
    yield
    Config.cleanup()

def test_without_fixture_call():
    # reset_configが自動的に実行される
    assert Config.get_setting("debug") is False
```

### 共有フィクスチャ用conftest.py

```python
# tests/conftest.py
import pytest

@pytest.fixture
def client():
    """すべてのテスト用の共有フィクスチャ。"""
    app = create_app(testing=True)
    with app.test_client() as client:
        yield client

@pytest.fixture
def auth_headers(client):
    """APIテスト用の認証ヘッダーを生成。"""
    response = client.post("/api/login", json={
        "username": "test",
        "password": "test"
    })
    token = response.json["token"]
    return {"Authorization": f"Bearer {token}"}
```

## パラメータ化

### 基本的なパラメータ化

```python
@pytest.mark.parametrize("input,expected", [
    ("hello", "HELLO"),
    ("world", "WORLD"),
    ("PyThOn", "PYTHON"),
])
def test_uppercase(input, expected):
    """テストは異なる入力で3回実行される。"""
    assert input.upper() == expected
```

### 複数パラメータ

```python
@pytest.mark.parametrize("a,b,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
    (100, 200, 300),
])
def test_add(a, b, expected):
    """複数の入力で加算をテスト。"""
    assert add(a, b) == expected
```

### ID付きパラメータ化

```python
@pytest.mark.parametrize("input,expected", [
    ("valid@email.com", True),
    ("invalid", False),
    ("@no-domain.com", False),
], ids=["valid-email", "missing-at", "missing-domain"])
def test_email_validation(input, expected):
    """読みやすいテストIDでメールバリデーションをテスト。"""
    assert is_valid_email(input) is expected
```

### パラメータ化フィクスチャ

```python
@pytest.fixture(params=["sqlite", "postgresql", "mysql"])
def db(request):
    """複数のデータベースバックエンドでテスト。"""
    if request.param == "sqlite":
        return Database(":memory:")
    elif request.param == "postgresql":
        return Database("postgresql://localhost/test")
    elif request.param == "mysql":
        return Database("mysql://localhost/test")

def test_database_operations(db):
    """テストは各データベースで3回実行される。"""
    result = db.query("SELECT 1")
    assert result is not None
```

## マーカーとテスト選択

### カスタムマーカー

```python
# 遅いテストをマーク
@pytest.mark.slow
def test_slow_operation():
    time.sleep(5)

# 統合テストをマーク
@pytest.mark.integration
def test_api_integration():
    response = requests.get("https://api.example.com")
    assert response.status_code == 200

# ユニットテストをマーク
@pytest.mark.unit
def test_unit_logic():
    assert calculate(2, 3) == 5
```

### 特定テストの実行

```bash
# 速いテストのみ実行
pytest -m "not slow"

# 統合テストのみ実行
pytest -m integration

# 統合または遅いテストを実行
pytest -m "integration or slow"

# ユニットとマークされたが遅くないテストを実行
pytest -m "unit and not slow"
```

### pytest.iniでマーカー設定

```ini
[pytest]
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    django: marks tests as requiring Django
```

## モッキングとパッチ

### 関数のモック

```python
from unittest.mock import patch, Mock

@patch("mypackage.external_api_call")
def test_with_mock(api_call_mock):
    """モックされた外部APIでテスト。"""
    api_call_mock.return_value = {"status": "success"}

    result = my_function()

    api_call_mock.assert_called_once()
    assert result["status"] == "success"
```

### 戻り値のモック

```python
@patch("mypackage.Database.connect")
def test_database_connection(connect_mock):
    """モックされたデータベース接続でテスト。"""
    connect_mock.return_value = MockConnection()

    db = Database()
    db.connect()

    connect_mock.assert_called_once_with("localhost")
```

### 例外のモック

```python
@patch("mypackage.api_call")
def test_api_error_handling(api_call_mock):
    """モックされた例外でエラー処理をテスト。"""
    api_call_mock.side_effect = ConnectionError("Network error")

    with pytest.raises(ConnectionError):
        api_call()

    api_call_mock.assert_called_once()
```

### コンテキストマネージャのモック

```python
@patch("builtins.open", new_callable=mock_open)
def test_file_reading(mock_file):
    """モックされたopenでファイル読み取りをテスト。"""
    mock_file.return_value.read.return_value = "file content"

    result = read_file("test.txt")

    mock_file.assert_called_once_with("test.txt", "r")
    assert result == "file content"
```

### autospecの使用

```python
@patch("mypackage.DBConnection", autospec=True)
def test_autospec(db_mock):
    """autospecでAPIの誤用をキャッチするテスト。"""
    db = db_mock.return_value
    db.query("SELECT * FROM users")

    # DBConnectionにqueryメソッドがない場合、これは失敗する
    db_mock.assert_called_once()
```

### クラスインスタンスのモック

```python
class TestUserService:
    @patch("mypackage.UserRepository")
    def test_create_user(self, repo_mock):
        """モックされたリポジトリでユーザー作成をテスト。"""
        repo_mock.return_value.save.return_value = User(id=1, name="Alice")

        service = UserService(repo_mock.return_value)
        user = service.create_user(name="Alice")

        assert user.name == "Alice"
        repo_mock.return_value.save.assert_called_once()
```

### プロパティのモック

```python
@pytest.fixture
def mock_config():
    """プロパティを持つモックを作成。"""
    config = Mock()
    type(config).debug = PropertyMock(return_value=True)
    type(config).api_key = PropertyMock(return_value="test-key")
    return config

def test_with_mock_config(mock_config):
    """モックされた設定プロパティでテスト。"""
    assert mock_config.debug is True
    assert mock_config.api_key == "test-key"
```

## 非同期コードのテスト

### pytest-asyncioでの非同期テスト

```python
import pytest

@pytest.mark.asyncio
async def test_async_function():
    """非同期関数をテスト。"""
    result = await async_add(2, 3)
    assert result == 5

@pytest.mark.asyncio
async def test_async_with_fixture(async_client):
    """非同期フィクスチャで非同期をテスト。"""
    response = await async_client.get("/api/users")
    assert response.status_code == 200
```

### 非同期フィクスチャ

```python
@pytest.fixture
async def async_client():
    """非同期テストクライアントを提供する非同期フィクスチャ。"""
    app = create_app()
    async with app.test_client() as client:
        yield client

@pytest.mark.asyncio
async def test_api_endpoint(async_client):
    """非同期フィクスチャを使用するテスト。"""
    response = await async_client.get("/api/data")
    assert response.status_code == 200
```

### 非同期関数のモック

```python
@pytest.mark.asyncio
@patch("mypackage.async_api_call")
async def test_async_mock(api_call_mock):
    """モックで非同期関数をテスト。"""
    api_call_mock.return_value = {"status": "ok"}

    result = await my_async_function()

    api_call_mock.assert_awaited_once()
    assert result["status"] == "ok"
```

## 例外のテスト

### 期待される例外のテスト

```python
def test_divide_by_zero():
    """ゼロ除算がZeroDivisionErrorを発生させることをテスト。"""
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_custom_exception():
    """メッセージ付きカスタム例外をテスト。"""
    with pytest.raises(ValueError, match="invalid input"):
        validate_input("invalid")
```

### 例外属性のテスト

```python
def test_exception_with_details():
    """カスタム属性を持つ例外をテスト。"""
    with pytest.raises(CustomError) as exc_info:
        raise CustomError("error", code=400)

    assert exc_info.value.code == 400
    assert "error" in str(exc_info.value)
```

## 副作用のテスト

### ファイル操作のテスト

```python
import tempfile
import os

def test_file_processing():
    """一時ファイルでファイル処理をテスト。"""
    with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.txt') as f:
        f.write("test content")
        temp_path = f.name

    try:
        result = process_file(temp_path)
        assert result == "processed: test content"
    finally:
        os.unlink(temp_path)
```

### pytestのtmp_pathフィクスチャでテスト

```python
def test_with_tmp_path(tmp_path):
    """pytestの組み込み一時パスフィクスチャを使用するテスト。"""
    test_file = tmp_path / "test.txt"
    test_file.write_text("hello world")

    result = process_file(str(test_file))
    assert result == "hello world"
    # tmp_pathは自動的にクリーンアップされる
```

### tmpdirフィクスチャでテスト

```python
def test_with_tmpdir(tmpdir):
    """pytestのtmpdirフィクスチャを使用するテスト。"""
    test_file = tmpdir.join("test.txt")
    test_file.write("data")

    result = process_file(str(test_file))
    assert result == "data"
```

## テスト構成

### ディレクトリ構造

```
tests/
├── conftest.py                 # 共有フィクスチャ
├── __init__.py
├── unit/                       # ユニットテスト
│   ├── __init__.py
│   ├── test_models.py
│   ├── test_utils.py
│   └── test_services.py
├── integration/                # 統合テスト
│   ├── __init__.py
│   ├── test_api.py
│   └── test_database.py
└── e2e/                        # エンドツーエンドテスト
    ├── __init__.py
    └── test_user_flow.py
```

### テストクラス

```python
class TestUserService:
    """関連するテストをクラスにグループ化。"""

    @pytest.fixture(autouse=True)
    def setup(self):
        """このクラスの各テストの前に実行されるセットアップ。"""
        self.service = UserService()

    def test_create_user(self):
        """ユーザー作成をテスト。"""
        user = self.service.create_user("Alice")
        assert user.name == "Alice"

    def test_delete_user(self):
        """ユーザー削除をテスト。"""
        user = User(id=1, name="Bob")
        self.service.delete_user(user)
        assert not self.service.user_exists(1)
```

## ベストプラクティス

### やるべきこと

- **TDDに従う**: コードより先にテストを書く（red-green-refactor）
- **1つをテスト**: 各テストは1つの動作を検証
- **説明的な名前**: `test_user_login_with_invalid_credentials_fails`
- **フィクスチャを使用**: フィクスチャで重複を排除
- **外部依存をモック**: 外部サービスに依存しない
- **エッジケースをテスト**: 空入力、None値、境界条件
- **80%以上のカバレッジを目指す**: クリティカルパスに焦点
- **テストを高速に保つ**: マークで遅いテストを分離

### やってはいけないこと

- **実装をテストしない**: 内部ではなく動作をテスト
- **テストで複雑な条件分岐を使わない**: テストをシンプルに保つ
- **テスト失敗を無視しない**: すべてのテストはパスすべき
- **サードパーティコードをテストしない**: ライブラリが動作することを信頼
- **テスト間で状態を共有しない**: テストは独立であるべき
- **テストで例外をキャッチしない**: `pytest.raises`を使用
- **print文を使わない**: アサーションとpytest出力を使用
- **脆いテストを書かない**: 過度に具体的なモックを避ける

## 一般的なパターン

### APIエンドポイントのテスト（FastAPI/Flask）

```python
@pytest.fixture
def client():
    app = create_app(testing=True)
    return app.test_client()

def test_get_user(client):
    response = client.get("/api/users/1")
    assert response.status_code == 200
    assert response.json["id"] == 1

def test_create_user(client):
    response = client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com"
    })
    assert response.status_code == 201
    assert response.json["name"] == "Alice"
```

### データベース操作のテスト

```python
@pytest.fixture
def db_session():
    """テストデータベースセッションを作成。"""
    session = Session(bind=engine)
    session.begin_nested()
    yield session
    session.rollback()
    session.close()

def test_create_user(db_session):
    user = User(name="Alice", email="alice@example.com")
    db_session.add(user)
    db_session.commit()

    retrieved = db_session.query(User).filter_by(name="Alice").first()
    assert retrieved.email == "alice@example.com"
```

### クラスメソッドのテスト

```python
class TestCalculator:
    @pytest.fixture
    def calculator(self):
        return Calculator()

    def test_add(self, calculator):
        assert calculator.add(2, 3) == 5

    def test_divide_by_zero(self, calculator):
        with pytest.raises(ZeroDivisionError):
            calculator.divide(10, 0)
```

## pytest設定

### pytest.ini

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --strict-markers
    --disable-warnings
    --cov=mypackage
    --cov-report=term-missing
    --cov-report=html
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

### pyproject.toml

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = [
    "--strict-markers",
    "--cov=mypackage",
    "--cov-report=term-missing",
    "--cov-report=html",
]
markers = [
    "slow: marks tests as slow",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

## テストの実行

```bash
# すべてのテストを実行
pytest

# 特定ファイルを実行
pytest tests/test_utils.py

# 特定テストを実行
pytest tests/test_utils.py::test_function

# 詳細出力で実行
pytest -v

# カバレッジ付きで実行
pytest --cov=mypackage --cov-report=html

# 速いテストのみ実行
pytest -m "not slow"

# 最初の失敗まで実行
pytest -x

# N回失敗で停止
pytest --maxfail=3

# 最後に失敗したテストを実行
pytest --lf

# パターンでテストを実行
pytest -k "test_user"

# 失敗時にデバッガで実行
pytest --pdb
```

## クイックリファレンス

| パターン | 使用法 |
|---------|-------|
| `pytest.raises()` | 期待される例外をテスト |
| `@pytest.fixture()` | 再利用可能なテストフィクスチャを作成 |
| `@pytest.mark.parametrize()` | 複数の入力でテストを実行 |
| `@pytest.mark.slow` | 遅いテストをマーク |
| `pytest -m "not slow"` | 遅いテストをスキップ |
| `@patch()` | 関数とクラスをモック |
| `tmp_path`フィクスチャ | 自動一時ディレクトリ |
| `pytest --cov` | カバレッジレポートを生成 |
| `assert` | シンプルで読みやすいアサーション |

**覚えておくこと**: テストもコードです。クリーンで、読みやすく、保守しやすく保ちましょう。良いテストはバグをキャッチします；優れたテストはバグを防ぎます。
