---
name: python-patterns
description: Pythonイディオム、PEP 8標準、型ヒント、堅牢で効率的かつ保守性の高いPythonアプリケーション構築のためのベストプラクティス。
---

# Python開発パターン

堅牢で効率的かつ保守性の高いアプリケーションを構築するためのPythonイディオムとベストプラクティス。

## 使用タイミング

- 新しいPythonコードを書く時
- Pythonコードをレビューする時
- 既存のPythonコードをリファクタリングする時
- Pythonパッケージ/モジュールを設計する時

## 基本原則

### 1. 可読性が重要

Pythonは可読性を優先します。コードは明確で理解しやすいものにすべきです。

```python
# 良い例: 明確で読みやすい
def get_active_users(users: list[User]) -> list[User]:
    """提供されたリストからアクティブユーザーのみを返す。"""
    return [user for user in users if user.is_active]


# 悪い例: 巧妙だが混乱を招く
def get_active_users(u):
    return [x for x in u if x.a]
```

### 2. 暗黙より明示

マジックを避け、コードが何をしているかを明確にしましょう。

```python
# 良い例: 明示的な設定
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# 悪い例: 隠れた副作用
import some_module
some_module.setup()  # これは何をする？
```

### 3. EAFP - 許可を求めるより許しを請う方が簡単

Pythonは条件チェックよりも例外処理を好みます。

```python
# 良い例: EAFPスタイル
def get_value(dictionary: dict, key: str) -> Any:
    try:
        return dictionary[key]
    except KeyError:
        return default_value

# 悪い例: LBYL（跳ぶ前に見る）スタイル
def get_value(dictionary: dict, key: str) -> Any:
    if key in dictionary:
        return dictionary[key]
    else:
        return default_value
```

## 型ヒント

### 基本的な型アノテーション

```python
from typing import Optional, List, Dict, Any

def process_user(
    user_id: str,
    data: Dict[str, Any],
    active: bool = True
) -> Optional[User]:
    """ユーザーを処理し、更新されたUserまたはNoneを返す。"""
    if not active:
        return None
    return User(user_id, data)
```

### モダンな型ヒント（Python 3.9以降）

```python
# Python 3.9以降 - 組み込み型を使用
def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Python 3.8以前 - typingモジュールを使用
from typing import List, Dict

def process_items(items: List[str]) -> Dict[str, int]:
    return {item: len(item) for item in items}
```

### 型エイリアスとTypeVar

```python
from typing import TypeVar, Union

# 複雑な型の型エイリアス
JSON = Union[dict[str, Any], list[Any], str, int, float, bool, None]

def parse_json(data: str) -> JSON:
    return json.loads(data)

# ジェネリック型
T = TypeVar('T')

def first(items: list[T]) -> T | None:
    """最初の要素を返す。リストが空の場合はNone。"""
    return items[0] if items else None
```

### プロトコルベースのダックタイピング

```python
from typing import Protocol

class Renderable(Protocol):
    def render(self) -> str:
        """オブジェクトを文字列にレンダリングする。"""

def render_all(items: list[Renderable]) -> str:
    """Renderableプロトコルを実装するすべてのアイテムをレンダリングする。"""
    return "\n".join(item.render() for item in items)
```

## エラー処理パターン

### 具体的な例外処理

```python
# 良い例: 具体的な例外をキャッチ
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except FileNotFoundError as e:
        raise ConfigError(f"Config file not found: {path}") from e
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in config: {path}") from e

# 悪い例: 裸のexcept
def load_config(path: str) -> Config:
    try:
        with open(path) as f:
            return Config.from_json(f.read())
    except:
        return None  # サイレントな失敗！
```

### 例外チェーン

```python
def process_data(data: str) -> Result:
    try:
        parsed = json.loads(data)
    except json.JSONDecodeError as e:
        # トレースバックを保持するために例外をチェーン
        raise ValueError(f"Failed to parse data: {data}") from e
```

### カスタム例外階層

```python
class AppError(Exception):
    """すべてのアプリケーションエラーの基底例外。"""
    pass

class ValidationError(AppError):
    """入力バリデーションが失敗した時に発生。"""
    pass

class NotFoundError(AppError):
    """要求されたリソースが見つからない時に発生。"""
    pass

# 使用例
def get_user(user_id: str) -> User:
    user = db.find_user(user_id)
    if not user:
        raise NotFoundError(f"User not found: {user_id}")
    return user
```

## コンテキストマネージャ

### リソース管理

```python
# 良い例: コンテキストマネージャを使用
def process_file(path: str) -> str:
    with open(path, 'r') as f:
        return f.read()

# 悪い例: 手動でリソース管理
def process_file(path: str) -> str:
    f = open(path, 'r')
    try:
        return f.read()
    finally:
        f.close()
```

### カスタムコンテキストマネージャ

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    """コードブロックの実行時間を計測するコンテキストマネージャ。"""
    start = time.perf_counter()
    yield
    elapsed = time.perf_counter() - start
    print(f"{name} took {elapsed:.4f} seconds")

# 使用例
with timer("data processing"):
    process_large_dataset()
```

### コンテキストマネージャクラス

```python
class DatabaseTransaction:
    def __init__(self, connection):
        self.connection = connection

    def __enter__(self):
        self.connection.begin_transaction()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.connection.commit()
        else:
            self.connection.rollback()
        return False  # 例外を抑制しない

# 使用例
with DatabaseTransaction(conn):
    user = conn.create_user(user_data)
    conn.create_profile(user.id, profile_data)
```

## 内包表記とジェネレータ

### リスト内包表記

```python
# 良い例: 単純な変換にリスト内包表記
names = [user.name for user in users if user.is_active]

# 悪い例: 手動ループ
names = []
for user in users:
    if user.is_active:
        names.append(user.name)

# 複雑な内包表記は展開すべき
# 悪い例: 複雑すぎる
result = [x * 2 for x in items if x > 0 if x % 2 == 0]

# 良い例: ジェネレータ関数を使用
def filter_and_transform(items: Iterable[int]) -> list[int]:
    result = []
    for x in items:
        if x > 0 and x % 2 == 0:
            result.append(x * 2)
    return result
```

### ジェネレータ式

```python
# 良い例: 遅延評価のためのジェネレータ
total = sum(x * x for x in range(1_000_000))

# 悪い例: 大きな中間リストを作成
total = sum([x * x for x in range(1_000_000)])
```

### ジェネレータ関数

```python
def read_large_file(path: str) -> Iterator[str]:
    """大きなファイルを1行ずつ読み取る。"""
    with open(path) as f:
        for line in f:
            yield line.strip()

# 使用例
for line in read_large_file("huge.txt"):
    process(line)
```

## データクラスとNamedTuple

### データクラス

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class User:
    """自動的に__init__、__repr__、__eq__が生成されるUserエンティティ。"""
    id: str
    name: str
    email: str
    created_at: datetime = field(default_factory=datetime.now)
    is_active: bool = True

# 使用例
user = User(
    id="123",
    name="Alice",
    email="alice@example.com"
)
```

### バリデーション付きデータクラス

```python
@dataclass
class User:
    email: str
    age: int

    def __post_init__(self):
        # メールフォーマットのバリデーション
        if "@" not in self.email:
            raise ValueError(f"Invalid email: {self.email}")
        # 年齢範囲のバリデーション
        if self.age < 0 or self.age > 150:
            raise ValueError(f"Invalid age: {self.age}")
```

### NamedTuple

```python
from typing import NamedTuple

class Point(NamedTuple):
    """イミュータブルな2Dポイント。"""
    x: float
    y: float

    def distance(self, other: 'Point') -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# 使用例
p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance(p2))  # 5.0
```

## デコレータ

### 関数デコレータ

```python
import functools
import time

def timer(func: Callable) -> Callable:
    """関数の実行時間を計測するデコレータ。"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f}s")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)

# slow_function()は出力: slow_function took 1.0012s
```

### パラメータ付きデコレータ

```python
def repeat(times: int):
    """関数を複数回繰り返すデコレータ。"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            results = []
            for _ in range(times):
                results.append(func(*args, **kwargs))
            return results
        return wrapper
    return decorator

@repeat(times=3)
def greet(name: str) -> str:
    return f"Hello, {name}!"

# greet("Alice")は["Hello, Alice!", "Hello, Alice!", "Hello, Alice!"]を返す
```

### クラスベースのデコレータ

```python
class CountCalls:
    """関数が何回呼び出されたかをカウントするデコレータ。"""
    def __init__(self, func: Callable):
        functools.update_wrapper(self, func)
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"{self.func.__name__} has been called {self.count} times")
        return self.func(*args, **kwargs)

@CountCalls
def process():
    pass

# process()を呼び出すたびに呼び出し回数が出力される
```

## 並行処理パターン

### I/Oバウンドタスクにはスレッディング

```python
import concurrent.futures
import threading

def fetch_url(url: str) -> str:
    """URLをフェッチする（I/Oバウンド操作）。"""
    import urllib.request
    with urllib.request.urlopen(url) as response:
        return response.read().decode()

def fetch_all_urls(urls: list[str]) -> dict[str, str]:
    """スレッドを使用して複数のURLを並行フェッチ。"""
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        future_to_url = {executor.submit(fetch_url, url): url for url in urls}
        results = {}
        for future in concurrent.futures.as_completed(future_to_url):
            url = future_to_url[future]
            try:
                results[url] = future.result()
            except Exception as e:
                results[url] = f"Error: {e}"
    return results
```

### CPUバウンドタスクにはマルチプロセッシング

```python
def process_data(data: list[int]) -> int:
    """CPU集約的な計算。"""
    return sum(x ** 2 for x in data)

def process_all(datasets: list[list[int]]) -> list[int]:
    """複数のプロセスを使用して複数のデータセットを処理。"""
    with concurrent.futures.ProcessPoolExecutor() as executor:
        results = list(executor.map(process_data, datasets))
    return results
```

### 並行I/OにはAsync/Await

```python
import asyncio

async def fetch_async(url: str) -> str:
    """URLを非同期でフェッチ。"""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()

async def fetch_all(urls: list[str]) -> dict[str, str]:
    """複数のURLを並行フェッチ。"""
    tasks = [fetch_async(url) for url in urls]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return dict(zip(urls, results))
```

## パッケージ構成

### 標準プロジェクトレイアウト

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── main.py
│       ├── api/
│       │   ├── __init__.py
│       │   └── routes.py
│       ├── models/
│       │   ├── __init__.py
│       │   └── user.py
│       └── utils/
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_api.py
│   └── test_models.py
├── pyproject.toml
├── README.md
└── .gitignore
```

### インポート規則

```python
# 良い例: インポート順序 - 標準ライブラリ、サードパーティ、ローカル
import os
import sys
from pathlib import Path

import requests
from fastapi import FastAPI

from mypackage.models import User
from mypackage.utils import format_name

# 良い例: 自動インポートソートにisortを使用
# pip install isort
```

### パッケージエクスポート用__init__.py

```python
# mypackage/__init__.py
"""mypackage - サンプルPythonパッケージ。"""

__version__ = "1.0.0"

# パッケージレベルでメインクラス/関数をエクスポート
from mypackage.models import User, Post
from mypackage.utils import format_name

__all__ = ["User", "Post", "format_name"]
```

## メモリとパフォーマンス

### メモリ効率のための__slots__使用

```python
# 悪い例: 通常のクラスは__dict__を使用（メモリ消費大）
class Point:
    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

# 良い例: __slots__でメモリ使用量を削減
class Point:
    __slots__ = ['x', 'y']

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y
```

### 大きなデータにはジェネレータ

```python
# 悪い例: 完全なリストをメモリに返す
def read_lines(path: str) -> list[str]:
    with open(path) as f:
        return [line.strip() for line in f]

# 良い例: 1行ずつyield
def read_lines(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()
```

### ループ内での文字列連結を避ける

```python
# 悪い例: 文字列の不変性によりO(n^2)
result = ""
for item in items:
    result += str(item)

# 良い例: joinを使用してO(n)
result = "".join(str(item) for item in items)

# 良い例: StringIOを使用して構築
from io import StringIO

buffer = StringIO()
for item in items:
    buffer.write(str(item))
result = buffer.getvalue()
```

## Pythonツーリング統合

### 必須コマンド

```bash
# コードフォーマット
black .
isort .

# リンティング
ruff check .
pylint mypackage/

# 型チェック
mypy .

# テスト
pytest --cov=mypackage --cov-report=html

# セキュリティスキャン
bandit -r .

# 依存関係管理
pip-audit
safety check
```

### pyproject.toml設定

```toml
[project]
name = "mypackage"
version = "1.0.0"
requires-python = ">=3.9"
dependencies = [
    "requests>=2.31.0",
    "pydantic>=2.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-cov>=4.1.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.5.0",
]

[tool.black]
line-length = 88
target-version = ['py39']

[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "--cov=mypackage --cov-report=term-missing"
```

## クイックリファレンス: Pythonイディオム

| イディオム | 説明 |
|-------|-------------|
| EAFP | 許可を求めるより許しを請う方が簡単 |
| コンテキストマネージャ | リソース管理には`with`を使用 |
| リスト内包表記 | 単純な変換用 |
| ジェネレータ | 遅延評価と大きなデータセット用 |
| 型ヒント | 関数シグネチャにアノテーション |
| データクラス | 自動生成メソッドを持つデータコンテナ用 |
| `__slots__` | メモリ最適化用 |
| f-strings | 文字列フォーマット用（Python 3.6以降） |
| `pathlib.Path` | パス操作用（Python 3.4以降） |
| `enumerate` | ループでのインデックス-要素ペア用 |

## 避けるべきアンチパターン

```python
# 悪い例: ミュータブルなデフォルト引数
def append_to(item, items=[]):
    items.append(item)
    return items

# 良い例: Noneを使用して新しいリストを作成
def append_to(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items

# 悪い例: type()で型をチェック
if type(obj) == list:
    process(obj)

# 良い例: isinstanceを使用
if isinstance(obj, list):
    process(obj)

# 悪い例: ==でNoneと比較
if value == None:
    process()

# 良い例: isを使用
if value is None:
    process()

# 悪い例: from module import *
from os.path import *

# 良い例: 明示的インポート
from os.path import join, exists

# 悪い例: 裸のexcept
try:
    risky_operation()
except:
    pass

# 良い例: 具体的な例外
try:
    risky_operation()
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
```

__覚えておくこと__: Pythonコードは読みやすく、明示的で、最小驚きの原則に従うべきです。迷った時は、巧妙さよりも明確さを優先しましょう。
