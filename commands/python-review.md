---
description: PEP 8準拠、型ヒント、セキュリティ、Pythonicなイディオムに関する包括的なPythonコードレビュー。python-reviewerエージェントを呼び出します。
---

# Pythonコードレビュー

このコマンドは**python-reviewer**エージェントを呼び出して、Python固有の包括的なコードレビューを行います。

## このコマンドの機能

1. **Python変更の特定**: `git diff` で変更された `.py` ファイルを検出
2. **静的解析の実行**: `ruff`、`mypy`、`pylint`、`black --check` を実行
3. **セキュリティスキャン**: SQLインジェクション、コマンドインジェクション、安全でないデシリアライゼーションをチェック
4. **型安全性レビュー**: 型ヒントとmypyエラーを分析
5. **Pythonicコードチェック**: コードがPEP 8とPythonのベストプラクティスに従っているか検証
6. **レポート生成**: 問題を重要度別にカテゴリ分け

## 使用するタイミング

以下の場合に `/python-review` を使用：
- Pythonコードを書いたり変更した後
- Python変更をコミットする前
- Pythonコードのプルリクエストをレビューするとき
- 新しいPythonコードベースにオンボーディングするとき
- Pythonicなパターンとイディオムを学ぶとき

## レビューカテゴリ

### クリティカル（必須修正）
- SQL/コマンドインジェクションの脆弱性
- 安全でないeval/execの使用
- Pickleの安全でないデシリアライゼーション
- ハードコードされた認証情報
- YAMLの安全でないload
- エラーを隠すベアexcept句

### 高（修正推奨）
- 公開関数に型ヒントがない
- ミュータブルなデフォルト引数
- 例外を黙って飲み込む
- リソースにコンテキストマネージャーを使用していない
- 内包表記の代わりにCスタイルのループ
- isinstance()の代わりにtype()を使用
- ロックなしの競合状態

### 中（検討）
- PEP 8フォーマット違反
- 公開関数にdocstringがない
- ロギングの代わりにprint文
- 非効率な文字列操作
- 名前付き定数なしのマジックナンバー
- フォーマットにf-stringを使用していない
- 不要なリスト作成

## 実行される自動チェック

```bash
# 型チェック
mypy .

# リンティングとフォーマット
ruff check .
black --check .
isort --check-only .

# セキュリティスキャン
bandit -r .

# 依存関係監査
pip-audit
safety check

# テスト
pytest --cov=app --cov-report=term-missing
```

## 使用例

```text
ユーザー: /python-review

エージェント:
# Pythonコードレビューレポート

## レビューされたファイル
- app/routes/user.py（変更）
- app/services/auth.py（変更）

## 静的解析結果
✓ ruff: 問題なし
✓ mypy: エラーなし
⚠️ black: 2ファイルの再フォーマットが必要
✓ bandit: セキュリティ問題なし

## 発見された問題

[クリティカル] SQLインジェクションの脆弱性
ファイル: app/routes/user.py:42
問題: ユーザー入力がSQLクエリに直接埋め込まれている
```python
query = f"SELECT * FROM users WHERE id = {user_id}"  # 悪い例
```
修正: パラメータ化クエリを使用
```python
query = "SELECT * FROM users WHERE id = %s"  # 良い例
cursor.execute(query, (user_id,))
```

[高] ミュータブルなデフォルト引数
ファイル: app/services/auth.py:18
問題: ミュータブルなデフォルト引数が共有状態を引き起こす
```python
def process_items(items=[]):  # 悪い例
    items.append("new")
    return items
```
修正: デフォルトにNoneを使用
```python
def process_items(items=None):  # 良い例
    if items is None:
        items = []
    items.append("new")
    return items
```

[中] 型ヒントの欠落
ファイル: app/services/auth.py:25
問題: 型アノテーションのない公開関数
```python
def get_user(user_id):  # 悪い例
    return db.find(user_id)
```
修正: 型ヒントを追加
```python
def get_user(user_id: str) -> Optional[User]:  # 良い例
    return db.find(user_id)
```

[中] コンテキストマネージャーを使用していない
ファイル: app/routes/user.py:55
問題: 例外時にファイルが閉じられない
```python
f = open("config.json")  # 悪い例
data = f.read()
f.close()
```
修正: コンテキストマネージャーを使用
```python
with open("config.json") as f:  # 良い例
    data = f.read()
```

## サマリー
- クリティカル: 1
- 高: 1
- 中: 2

推奨: ❌ クリティカルな問題が修正されるまでマージをブロック

## 必要なフォーマット
実行: `black app/routes/user.py app/services/auth.py`
```

## 承認基準

| ステータス | 条件 |
|--------|-----------|
| ✅ 承認 | クリティカルまたは高の問題なし |
| ⚠️ 警告 | 中の問題のみ（注意してマージ） |
| ❌ ブロック | クリティカルまたは高の問題あり |

## 他のコマンドとの統合

- まず `/python-test` を使用してテストが成功することを確認
- Python固有でない懸念事項には `/code-review` を使用
- コミット前に `/python-review` を使用
- 静的解析ツールが失敗した場合は `/build-fix` を使用

## フレームワーク固有のレビュー

### Djangoプロジェクト
レビューアーは以下をチェックします：
- N+1クエリ問題（`select_related` と `prefetch_related` を使用）
- モデル変更のマイグレーション欠落
- ORMで処理できる場合の生SQL使用
- 複数ステップ操作での `transaction.atomic()` 欠落

### FastAPIプロジェクト
レビューアーは以下をチェックします：
- CORSの誤設定
- リクエスト検証用のPydanticモデル
- レスポンスモデルの正確性
- 適切なasync/awaitの使用
- 依存性注入パターン

### Flaskプロジェクト
レビューアーは以下をチェックします：
- コンテキスト管理（アプリコンテキスト、リクエストコンテキスト）
- 適切なエラーハンドリング
- Blueprintの構成
- 設定管理

## 関連

- エージェント: `agents/python-reviewer.md`
- スキル: `skills/python-patterns/`、`skills/python-testing/`

## よくある修正

### 型ヒントの追加
```python
# 変更前
def calculate(x, y):
    return x + y

# 変更後
from typing import Union

def calculate(x: Union[int, float], y: Union[int, float]) -> Union[int, float]:
    return x + y
```

### コンテキストマネージャーの使用
```python
# 変更前
f = open("file.txt")
data = f.read()
f.close()

# 変更後
with open("file.txt") as f:
    data = f.read()
```

### リスト内包表記の使用
```python
# 変更前
result = []
for item in items:
    if item.active:
        result.append(item.name)

# 変更後
result = [item.name for item in items if item.active]
```

### ミュータブルデフォルトの修正
```python
# 変更前
def append(value, items=[]):
    items.append(value)
    return items

# 変更後
def append(value, items=None):
    if items is None:
        items = []
    items.append(value)
    return items
```

### f-stringの使用（Python 3.6+）
```python
# 変更前
name = "Alice"
greeting = "Hello, " + name + "!"
greeting2 = "Hello, {}".format(name)

# 変更後
greeting = f"Hello, {name}!"
```

### ループ内の文字列連結の修正
```python
# 変更前
result = ""
for item in items:
    result += str(item)

# 変更後
result = "".join(str(item) for item in items)
```

## Pythonバージョンの互換性

レビューアーは新しいPythonバージョンの機能を使用している場合に注記します：

| 機能 | 最小Python |
|---------|----------------|
| 型ヒント | 3.5+ |
| f-string | 3.6+ |
| ウォルラス演算子 (`:=`) | 3.8+ |
| 位置専用パラメータ | 3.8+ |
| match文 | 3.10+ |
| 型ユニオン (`x | None`) | 3.10+ |

プロジェクトの `pyproject.toml` または `setup.py` で正しい最小Pythonバージョンが指定されていることを確認してください。
