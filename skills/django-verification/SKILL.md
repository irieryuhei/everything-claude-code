---
name: django-verification
description: Djangoプロジェクトの検証ループ：マイグレーション、リンティング、カバレッジ付きテスト、セキュリティスキャン、およびリリースまたはPR前のデプロイ準備チェック。
---

# Django検証ループ

PR前、大きな変更後、およびデプロイ前に実行して、Djangoアプリケーションの品質とセキュリティを確保します。

## フェーズ1: 環境チェック

```bash
# Pythonバージョンを確認
python --version  # プロジェクト要件と一致する必要がある

# 仮想環境をチェック
which python
pip list --outdated

# 環境変数を確認
python -c "import os; import environ; print('DJANGO_SECRET_KEY set' if os.environ.get('DJANGO_SECRET_KEY') else 'MISSING: DJANGO_SECRET_KEY')"
```

環境が正しく設定されていない場合は、停止して修正してください。

## フェーズ2: コード品質とフォーマット

```bash
# 型チェック
mypy . --config-file pyproject.toml

# ruffでリンティング
ruff check . --fix

# blackでフォーマット
black . --check
black .  # 自動修正

# インポートのソート
isort . --check-only
isort .  # 自動修正

# Django固有のチェック
python manage.py check --deploy
```

一般的な問題：
- パブリック関数に型ヒントがない
- PEP 8フォーマット違反
- ソートされていないインポート
- 本番設定にデバッグ設定が残っている

## フェーズ3: マイグレーション

```bash
# 未適用のマイグレーションをチェック
python manage.py showmigrations

# 不足しているマイグレーションを作成
python manage.py makemigrations --check

# マイグレーション適用のドライラン
python manage.py migrate --plan

# マイグレーションを適用（テスト環境）
python manage.py migrate

# マイグレーションの競合をチェック
python manage.py makemigrations --merge  # 競合がある場合のみ
```

レポート：
- 保留中のマイグレーション数
- マイグレーションの競合
- マイグレーションのないモデル変更

## フェーズ4: テスト + カバレッジ

```bash
# pytestですべてのテストを実行
pytest --cov=apps --cov-report=html --cov-report=term-missing --reuse-db

# 特定のアプリのテストを実行
pytest apps/users/tests/

# マーカー付きで実行
pytest -m "not slow"  # 遅いテストをスキップ
pytest -m integration  # 統合テストのみ

# カバレッジレポート
open htmlcov/index.html
```

レポート：
- 合計テスト: X パス、Y 失敗、Z スキップ
- 全体カバレッジ: XX%
- アプリ別カバレッジ内訳

カバレッジ目標：

| コンポーネント | 目標 |
|--------------|------|
| モデル | 90%以上 |
| シリアライザー | 85%以上 |
| ビュー | 80%以上 |
| サービス | 90%以上 |
| 全体 | 80%以上 |

## フェーズ5: セキュリティスキャン

```bash
# 依存関係の脆弱性
pip-audit
safety check --full-report

# Djangoセキュリティチェック
python manage.py check --deploy

# Banditセキュリティリンター
bandit -r . -f json -o bandit-report.json

# シークレットスキャン（gitleaksがインストールされている場合）
gitleaks detect --source . --verbose

# 環境変数チェック
python -c "from django.core.exceptions import ImproperlyConfigured; from django.conf import settings; settings.DEBUG"
```

レポート：
- 脆弱な依存関係の発見
- セキュリティ設定の問題
- ハードコードされたシークレットの検出
- DEBUGモードの状態（本番ではFalseであるべき）

## フェーズ6: Django管理コマンド

```bash
# モデルの問題をチェック
python manage.py check

# 静的ファイルを収集
python manage.py collectstatic --noinput --clear

# スーパーユーザーを作成（テストに必要な場合）
echo "from apps.users.models import User; User.objects.create_superuser('admin@example.com', 'admin')" | python manage.py shell

# データベース整合性
python manage.py check --database default

# キャッシュ確認（Redisを使用している場合）
python -c "from django.core.cache import cache; cache.set('test', 'value', 10); print(cache.get('test'))"
```

## フェーズ7: パフォーマンスチェック

```bash
# Django Debug Toolbar出力（N+1クエリをチェック）
# DEBUG=Trueでdev modeで実行し、ページにアクセス
# SQLパネルで重複クエリを確認

# クエリ数分析
django-admin debugsqlshell  # django-debug-sqlshellがインストールされている場合

# 不足しているインデックスをチェック
python manage.py shell << EOF
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT table_name, index_name FROM information_schema.statistics WHERE table_schema = 'public'")
    print(cursor.fetchall())
EOF
```

レポート：
- ページあたりのクエリ数（通常のページでは50未満であるべき）
- 不足しているデータベースインデックス
- 重複クエリの検出

## フェーズ8: 静的アセット

```bash
# npm依存関係をチェック（npmを使用している場合）
npm audit
npm audit fix

# 静的ファイルをビルド（webpack/viteを使用している場合）
npm run build

# 静的ファイルを確認
ls -la staticfiles/
python manage.py findstatic css/style.css
```

## フェーズ9: 設定レビュー

```python
# Pythonシェルで実行して設定を確認
python manage.py shell << EOF
from django.conf import settings
import os

# 重要なチェック
checks = {
    'DEBUG is False': not settings.DEBUG,
    'SECRET_KEY set': bool(settings.SECRET_KEY and len(settings.SECRET_KEY) > 30),
    'ALLOWED_HOSTS set': len(settings.ALLOWED_HOSTS) > 0,
    'HTTPS enabled': getattr(settings, 'SECURE_SSL_REDIRECT', False),
    'HSTS enabled': getattr(settings, 'SECURE_HSTS_SECONDS', 0) > 0,
    'Database configured': settings.DATABASES['default']['ENGINE'] != 'django.db.backends.sqlite3',
}

for check, result in checks.items():
    status = '✓' if result else '✗'
    print(f"{status} {check}")
EOF
```

## フェーズ10: ロギング設定

```bash
# ロギング出力をテスト
python manage.py shell << EOF
import logging
logger = logging.getLogger('django')
logger.warning('Test warning message')
logger.error('Test error message')
EOF

# ログファイルをチェック（設定されている場合）
tail -f /var/log/django/django.log
```

## フェーズ11: APIドキュメント（DRFの場合）

```bash
# スキーマを生成
python manage.py generateschema --format openapi-json > schema.json

# スキーマを検証
# schema.jsonが有効なJSONかチェック
python -c "import json; json.load(open('schema.json'))"

# Swagger UIにアクセス（drf-yasgを使用している場合）
# ブラウザでhttp://localhost:8000/swagger/にアクセス
```

## フェーズ12: 差分レビュー

```bash
# 差分統計を表示
git diff --stat

# 実際の変更を表示
git diff

# 変更されたファイルを表示
git diff --name-only

# 一般的な問題をチェック
git diff | grep -i "todo\|fixme\|hack\|xxx"
git diff | grep "print("  # デバッグステートメント
git diff | grep "DEBUG = True"  # デバッグモード
git diff | grep "import pdb"  # デバッガー
```

チェックリスト：
- デバッグステートメントなし（print、pdb、breakpoint()）
- 重要なコードにTODO/FIXMEコメントなし
- ハードコードされたシークレットや認証情報なし
- モデル変更にデータベースマイグレーションが含まれている
- 設定変更が文書化されている
- 外部呼び出しにエラーハンドリングがある
- 必要な場所にトランザクション管理がある

## 出力テンプレート

```
DJANGO検証レポート
==========================

フェーズ1: 環境チェック
  ✓ Python 3.11.5
  ✓ 仮想環境がアクティブ
  ✓ すべての環境変数が設定済み

フェーズ2: コード品質
  ✓ mypy: 型エラーなし
  ✗ ruff: 3件の問題を発見（自動修正済み）
  ✓ black: フォーマット問題なし
  ✓ isort: インポートが適切にソート
  ✓ manage.py check: 問題なし

フェーズ3: マイグレーション
  ✓ 未適用のマイグレーションなし
  ✓ マイグレーションの競合なし
  ✓ すべてのモデルにマイグレーションあり

フェーズ4: テスト + カバレッジ
  テスト: 247パス、0失敗、5スキップ
  カバレッジ:
    全体: 87%
    users: 92%
    products: 89%
    orders: 85%
    payments: 91%

フェーズ5: セキュリティスキャン
  ✗ pip-audit: 2件の脆弱性を発見（修正が必要）
  ✓ safety check: 問題なし
  ✓ bandit: セキュリティ問題なし
  ✓ シークレット検出なし
  ✓ DEBUG = False

フェーズ6: Djangoコマンド
  ✓ collectstatic完了
  ✓ データベース整合性OK
  ✓ キャッシュバックエンド到達可能

フェーズ7: パフォーマンス
  ✓ N+1クエリ検出なし
  ✓ データベースインデックス設定済み
  ✓ クエリ数は許容範囲内

フェーズ8: 静的アセット
  ✓ npm audit: 脆弱性なし
  ✓ アセットビルド成功
  ✓ 静的ファイル収集済み

フェーズ9: 設定
  ✓ DEBUG = False
  ✓ SECRET_KEY設定済み
  ✓ ALLOWED_HOSTS設定済み
  ✓ HTTPS有効
  ✓ HSTS有効
  ✓ データベース設定済み

フェーズ10: ロギング
  ✓ ロギング設定済み
  ✓ ログファイル書き込み可能

フェーズ11: APIドキュメント
  ✓ スキーマ生成済み
  ✓ Swagger UIアクセス可能

フェーズ12: 差分レビュー
  変更ファイル: 12
  +450、-120行
  ✓ デバッグステートメントなし
  ✓ ハードコードされたシークレットなし
  ✓ マイグレーション含む

推奨事項: ⚠️ デプロイ前にpip-auditの脆弱性を修正してください

次のステップ:
1. 脆弱な依存関係を更新
2. セキュリティスキャンを再実行
3. 最終テストのためにステージングにデプロイ
```

## デプロイ前チェックリスト

- [ ] すべてのテストがパス
- [ ] カバレッジ ≥ 80%
- [ ] セキュリティ脆弱性なし
- [ ] 未適用のマイグレーションなし
- [ ] 本番設定でDEBUG = False
- [ ] SECRET_KEYが適切に設定
- [ ] ALLOWED_HOSTSが正しく設定
- [ ] データベースバックアップが有効
- [ ] 静的ファイルが収集され提供されている
- [ ] ロギングが設定され動作中
- [ ] エラーモニタリング（Sentryなど）が設定済み
- [ ] CDNが設定済み（該当する場合）
- [ ] Redis/キャッシュバックエンドが設定済み
- [ ] Celeryワーカーが実行中（該当する場合）
- [ ] HTTPS/SSLが設定済み
- [ ] 環境変数が文書化済み

## 継続的インテグレーション

### GitHub Actionsの例

```yaml
# .github/workflows/django-verification.yml
name: Django検証

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Pythonをセットアップ
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: pipをキャッシュ
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

      - name: 依存関係をインストール
        run: |
          pip install -r requirements.txt
          pip install ruff black mypy pytest pytest-django pytest-cov bandit safety pip-audit

      - name: コード品質チェック
        run: |
          ruff check .
          black . --check
          isort . --check-only
          mypy .

      - name: セキュリティスキャン
        run: |
          bandit -r . -f json -o bandit-report.json
          safety check --full-report
          pip-audit

      - name: テストを実行
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          DJANGO_SECRET_KEY: test-secret-key
        run: |
          pytest --cov=apps --cov-report=xml --cov-report=term-missing

      - name: カバレッジをアップロード
        uses: codecov/codecov-action@v3
```

## クイックリファレンス

| チェック | コマンド |
|---------|---------|
| 環境 | `python --version` |
| 型チェック | `mypy .` |
| リンティング | `ruff check .` |
| フォーマット | `black . --check` |
| マイグレーション | `python manage.py makemigrations --check` |
| テスト | `pytest --cov=apps` |
| セキュリティ | `pip-audit && bandit -r .` |
| Djangoチェック | `python manage.py check --deploy` |
| Collectstatic | `python manage.py collectstatic --noinput` |
| 差分統計 | `git diff --stat` |

覚えておくこと: 自動検証は一般的な問題を検出しますが、手動コードレビューとステージング環境でのテストに取って代わるものではありません。
