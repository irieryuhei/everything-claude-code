---
name: django-tdd
description: pytest-djangoを使用したDjangoテスト戦略、TDD方法論、factory_boy、モッキング、カバレッジ、およびDjango REST Framework APIのテスト。
---

# DjangoテストとTDD

pytest、factory_boy、およびDjango REST Frameworkを使用したDjangoアプリケーションのテスト駆動開発。

## 発動条件

- 新しいDjangoアプリケーションの作成
- Django REST Framework APIの実装
- Djangoモデル、ビュー、シリアライザーのテスト
- Djangoプロジェクトのテストインフラストラクチャのセットアップ

## DjangoのためのTDDワークフロー

### Red-Green-Refactorサイクル

```python
# ステップ1: RED - 失敗するテストを書く
def test_user_creation():
    user = User.objects.create_user(email='test@example.com', password='testpass123')
    assert user.email == 'test@example.com'
    assert user.check_password('testpass123')
    assert not user.is_staff

# ステップ2: GREEN - テストをパスさせる
# Userモデルまたはファクトリを作成

# ステップ3: REFACTOR - テストをグリーンに保ちながら改善
```

## セットアップ

### pytest設定

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings.test
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --reuse-db
    --nomigrations
    --cov=apps
    --cov-report=html
    --cov-report=term-missing
    --strict-markers
markers =
    slow: 遅いテストとしてマーク
    integration: 統合テストとしてマーク
```

### テスト設定

```python
# config/settings/test.py
from .base import *

DEBUG = True
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',
    }
}

# 速度のためにマイグレーションを無効化
class DisableMigrations:
    def __contains__(self, item):
        return True

    def __getitem__(self, item):
        return None

MIGRATION_MODULES = DisableMigrations()

# 高速なパスワードハッシュ
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.MD5PasswordHasher',
]

# メールバックエンド
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'

# Celery常に同期実行
CELERY_TASK_ALWAYS_EAGER = True
CELERY_TASK_EAGER_PROPAGATES = True
```

### conftest.py

```python
# tests/conftest.py
import pytest
from django.utils import timezone
from django.contrib.auth import get_user_model

User = get_user_model()

@pytest.fixture(autouse=True)
def timezone_settings(settings):
    """一貫したタイムゾーンを確保。"""
    settings.TIME_ZONE = 'UTC'

@pytest.fixture
def user(db):
    """テストユーザーを作成。"""
    return User.objects.create_user(
        email='test@example.com',
        password='testpass123',
        username='testuser'
    )

@pytest.fixture
def admin_user(db):
    """管理者ユーザーを作成。"""
    return User.objects.create_superuser(
        email='admin@example.com',
        password='adminpass123',
        username='admin'
    )

@pytest.fixture
def authenticated_client(client, user):
    """認証済みクライアントを返す。"""
    client.force_login(user)
    return client

@pytest.fixture
def api_client():
    """DRF APIクライアントを返す。"""
    from rest_framework.test import APIClient
    return APIClient()

@pytest.fixture
def authenticated_api_client(api_client, user):
    """認証済みAPIクライアントを返す。"""
    api_client.force_authenticate(user=user)
    return api_client
```

## Factory Boy

### ファクトリセットアップ

```python
# tests/factories.py
import factory
from factory import fuzzy
from datetime import datetime, timedelta
from django.contrib.auth import get_user_model
from apps.products.models import Product, Category

User = get_user_model()

class UserFactory(factory.django.DjangoModelFactory):
    """Userモデル用のファクトリ。"""

    class Meta:
        model = User

    email = factory.Sequence(lambda n: f"user{n}@example.com")
    username = factory.Sequence(lambda n: f"user{n}")
    password = factory.PostGenerationMethodCall('set_password', 'testpass123')
    first_name = factory.Faker('first_name')
    last_name = factory.Faker('last_name')
    is_active = True

class CategoryFactory(factory.django.DjangoModelFactory):
    """Categoryモデル用のファクトリ。"""

    class Meta:
        model = Category

    name = factory.Faker('word')
    slug = factory.LazyAttribute(lambda obj: obj.name.lower())
    description = factory.Faker('text')

class ProductFactory(factory.django.DjangoModelFactory):
    """Productモデル用のファクトリ。"""

    class Meta:
        model = Product

    name = factory.Faker('sentence', nb_words=3)
    slug = factory.LazyAttribute(lambda obj: obj.name.lower().replace(' ', '-'))
    description = factory.Faker('text')
    price = fuzzy.FuzzyDecimal(10.00, 1000.00, 2)
    stock = fuzzy.FuzzyInteger(0, 100)
    is_active = True
    category = factory.SubFactory(CategoryFactory)
    created_by = factory.SubFactory(UserFactory)

    @factory.post_generation
    def tags(self, create, extracted, **kwargs):
        """製品にタグを追加。"""
        if not create:
            return
        if extracted:
            for tag in extracted:
                self.tags.add(tag)
```

### ファクトリの使用

```python
# tests/test_models.py
import pytest
from tests.factories import ProductFactory, UserFactory

def test_product_creation():
    """ファクトリを使用した製品作成テスト。"""
    product = ProductFactory(price=100.00, stock=50)
    assert product.price == 100.00
    assert product.stock == 50
    assert product.is_active is True

def test_product_with_tags():
    """タグ付き製品のテスト。"""
    tags = [TagFactory(name='electronics'), TagFactory(name='new')]
    product = ProductFactory(tags=tags)
    assert product.tags.count() == 2

def test_multiple_products():
    """複数製品の作成テスト。"""
    products = ProductFactory.create_batch(10)
    assert len(products) == 10
```

## モデルテスト

### モデルテスト

```python
# tests/test_models.py
import pytest
from django.core.exceptions import ValidationError
from tests.factories import UserFactory, ProductFactory

class TestUserModel:
    """Userモデルのテスト。"""

    def test_create_user(self, db):
        """通常ユーザーの作成テスト。"""
        user = UserFactory(email='test@example.com')
        assert user.email == 'test@example.com'
        assert user.check_password('testpass123')
        assert not user.is_staff
        assert not user.is_superuser

    def test_create_superuser(self, db):
        """スーパーユーザーの作成テスト。"""
        user = UserFactory(
            email='admin@example.com',
            is_staff=True,
            is_superuser=True
        )
        assert user.is_staff
        assert user.is_superuser

    def test_user_str(self, db):
        """ユーザーの文字列表現テスト。"""
        user = UserFactory(email='test@example.com')
        assert str(user) == 'test@example.com'

class TestProductModel:
    """Productモデルのテスト。"""

    def test_product_creation(self, db):
        """製品の作成テスト。"""
        product = ProductFactory()
        assert product.id is not None
        assert product.is_active is True
        assert product.created_at is not None

    def test_product_slug_generation(self, db):
        """自動スラグ生成テスト。"""
        product = ProductFactory(name='Test Product')
        assert product.slug == 'test-product'

    def test_product_price_validation(self, db):
        """価格が負にならないことをテスト。"""
        product = ProductFactory(price=-10)
        with pytest.raises(ValidationError):
            product.full_clean()

    def test_product_manager_active(self, db):
        """アクティブマネージャーメソッドのテスト。"""
        ProductFactory.create_batch(5, is_active=True)
        ProductFactory.create_batch(3, is_active=False)

        active_count = Product.objects.active().count()
        assert active_count == 5

    def test_product_stock_management(self, db):
        """在庫管理のテスト。"""
        product = ProductFactory(stock=10)
        product.reduce_stock(5)
        product.refresh_from_db()
        assert product.stock == 5

        with pytest.raises(ValueError):
            product.reduce_stock(10)  # 在庫不足
```

## ビューテスト

### Djangoビューテスト

```python
# tests/test_views.py
import pytest
from django.urls import reverse
from tests.factories import ProductFactory, UserFactory

class TestProductViews:
    """製品ビューのテスト。"""

    def test_product_list(self, client, db):
        """製品一覧ビューのテスト。"""
        ProductFactory.create_batch(10)

        response = client.get(reverse('products:list'))

        assert response.status_code == 200
        assert len(response.context['products']) == 10

    def test_product_detail(self, client, db):
        """製品詳細ビューのテスト。"""
        product = ProductFactory()

        response = client.get(reverse('products:detail', kwargs={'slug': product.slug}))

        assert response.status_code == 200
        assert response.context['product'] == product

    def test_product_create_requires_login(self, client, db):
        """製品作成に認証が必要なことをテスト。"""
        response = client.get(reverse('products:create'))

        assert response.status_code == 302
        assert response.url.startswith('/accounts/login/')

    def test_product_create_authenticated(self, authenticated_client, db):
        """認証済みユーザーとしての製品作成テスト。"""
        response = authenticated_client.get(reverse('products:create'))

        assert response.status_code == 200

    def test_product_create_post(self, authenticated_client, db, category):
        """POSTによる製品作成テスト。"""
        data = {
            'name': 'Test Product',
            'description': 'A test product',
            'price': '99.99',
            'stock': 10,
            'category': category.id,
        }

        response = authenticated_client.post(reverse('products:create'), data)

        assert response.status_code == 302
        assert Product.objects.filter(name='Test Product').exists()
```

## DRF APIテスト

### シリアライザーテスト

```python
# tests/test_serializers.py
import pytest
from rest_framework.exceptions import ValidationError
from apps.products.serializers import ProductSerializer
from tests.factories import ProductFactory

class TestProductSerializer:
    """ProductSerializerのテスト。"""

    def test_serialize_product(self, db):
        """製品のシリアライズテスト。"""
        product = ProductFactory()
        serializer = ProductSerializer(product)

        data = serializer.data

        assert data['id'] == product.id
        assert data['name'] == product.name
        assert data['price'] == str(product.price)

    def test_deserialize_product(self, db):
        """製品データのデシリアライズテスト。"""
        data = {
            'name': 'Test Product',
            'description': 'Test description',
            'price': '99.99',
            'stock': 10,
            'category': 1,
        }

        serializer = ProductSerializer(data=data)

        assert serializer.is_valid()
        product = serializer.save()

        assert product.name == 'Test Product'
        assert float(product.price) == 99.99

    def test_price_validation(self, db):
        """価格バリデーションテスト。"""
        data = {
            'name': 'Test Product',
            'price': '-10.00',
            'stock': 10,
        }

        serializer = ProductSerializer(data=data)

        assert not serializer.is_valid()
        assert 'price' in serializer.errors

    def test_stock_validation(self, db):
        """在庫が負にならないことをテスト。"""
        data = {
            'name': 'Test Product',
            'price': '99.99',
            'stock': -5,
        }

        serializer = ProductSerializer(data=data)

        assert not serializer.is_valid()
        assert 'stock' in serializer.errors
```

### API ViewSetテスト

```python
# tests/test_api.py
import pytest
from rest_framework.test import APIClient
from rest_framework import status
from django.urls import reverse
from tests.factories import ProductFactory, UserFactory

class TestProductAPI:
    """製品APIエンドポイントのテスト。"""

    @pytest.fixture
    def api_client(self):
        """APIクライアントを返す。"""
        return APIClient()

    def test_list_products(self, api_client, db):
        """製品一覧テスト。"""
        ProductFactory.create_batch(10)

        url = reverse('api:product-list')
        response = api_client.get(url)

        assert response.status_code == status.HTTP_200_OK
        assert response.data['count'] == 10

    def test_retrieve_product(self, api_client, db):
        """製品取得テスト。"""
        product = ProductFactory()

        url = reverse('api:product-detail', kwargs={'pk': product.id})
        response = api_client.get(url)

        assert response.status_code == status.HTTP_200_OK
        assert response.data['id'] == product.id

    def test_create_product_unauthorized(self, api_client, db):
        """認証なしでの製品作成テスト。"""
        url = reverse('api:product-list')
        data = {'name': 'Test Product', 'price': '99.99'}

        response = api_client.post(url, data)

        assert response.status_code == status.HTTP_401_UNAUTHORIZED

    def test_create_product_authorized(self, authenticated_api_client, db):
        """認証済みユーザーとしての製品作成テスト。"""
        url = reverse('api:product-list')
        data = {
            'name': 'Test Product',
            'description': 'Test',
            'price': '99.99',
            'stock': 10,
        }

        response = authenticated_api_client.post(url, data)

        assert response.status_code == status.HTTP_201_CREATED
        assert response.data['name'] == 'Test Product'

    def test_update_product(self, authenticated_api_client, db):
        """製品更新テスト。"""
        product = ProductFactory(created_by=authenticated_api_client.user)

        url = reverse('api:product-detail', kwargs={'pk': product.id})
        data = {'name': 'Updated Product'}

        response = authenticated_api_client.patch(url, data)

        assert response.status_code == status.HTTP_200_OK
        assert response.data['name'] == 'Updated Product'

    def test_delete_product(self, authenticated_api_client, db):
        """製品削除テスト。"""
        product = ProductFactory(created_by=authenticated_api_client.user)

        url = reverse('api:product-detail', kwargs={'pk': product.id})
        response = authenticated_api_client.delete(url)

        assert response.status_code == status.HTTP_204_NO_CONTENT

    def test_filter_products_by_price(self, api_client, db):
        """価格による製品フィルタテスト。"""
        ProductFactory(price=50)
        ProductFactory(price=150)

        url = reverse('api:product-list')
        response = api_client.get(url, {'price_min': 100})

        assert response.status_code == status.HTTP_200_OK
        assert response.data['count'] == 1

    def test_search_products(self, api_client, db):
        """製品検索テスト。"""
        ProductFactory(name='Apple iPhone')
        ProductFactory(name='Samsung Galaxy')

        url = reverse('api:product-list')
        response = api_client.get(url, {'search': 'Apple'})

        assert response.status_code == status.HTTP_200_OK
        assert response.data['count'] == 1
```

## モッキングとパッチング

### 外部サービスのモッキング

```python
# tests/test_views.py
from unittest.mock import patch, Mock
import pytest

class TestPaymentView:
    """モックした決済ゲートウェイでの決済ビューテスト。"""

    @patch('apps.payments.services.stripe')
    def test_successful_payment(self, mock_stripe, client, user, product):
        """モックしたStripeでの成功した決済テスト。"""
        # モックを設定
        mock_stripe.Charge.create.return_value = {
            'id': 'ch_123',
            'status': 'succeeded',
            'amount': 9999,
        }

        client.force_login(user)
        response = client.post(reverse('payments:process'), {
            'product_id': product.id,
            'token': 'tok_visa',
        })

        assert response.status_code == 302
        mock_stripe.Charge.create.assert_called_once()

    @patch('apps.payments.services.stripe')
    def test_failed_payment(self, mock_stripe, client, user, product):
        """失敗した決済テスト。"""
        mock_stripe.Charge.create.side_effect = Exception('Card declined')

        client.force_login(user)
        response = client.post(reverse('payments:process'), {
            'product_id': product.id,
            'token': 'tok_visa',
        })

        assert response.status_code == 302
        assert 'error' in response.url
```

### メール送信のモッキング

```python
# tests/test_email.py
from django.core import mail
from django.test import override_settings

@override_settings(EMAIL_BACKEND='django.core.mail.backends.locmem.EmailBackend')
def test_order_confirmation_email(db, order):
    """注文確認メールのテスト。"""
    order.send_confirmation_email()

    assert len(mail.outbox) == 1
    assert order.user.email in mail.outbox[0].to
    assert 'Order Confirmation' in mail.outbox[0].subject
```

## 統合テスト

### フルフローテスト

```python
# tests/test_integration.py
import pytest
from django.urls import reverse
from tests.factories import UserFactory, ProductFactory

class TestCheckoutFlow:
    """完全なチェックアウトフローのテスト。"""

    def test_guest_to_purchase_flow(self, client, db):
        """ゲストから購入までの完全フローテスト。"""
        # ステップ1: 登録
        response = client.post(reverse('users:register'), {
            'email': 'test@example.com',
            'password': 'testpass123',
            'password_confirm': 'testpass123',
        })
        assert response.status_code == 302

        # ステップ2: ログイン
        response = client.post(reverse('users:login'), {
            'email': 'test@example.com',
            'password': 'testpass123',
        })
        assert response.status_code == 302

        # ステップ3: 製品を閲覧
        product = ProductFactory(price=100)
        response = client.get(reverse('products:detail', kwargs={'slug': product.slug}))
        assert response.status_code == 200

        # ステップ4: カートに追加
        response = client.post(reverse('cart:add'), {
            'product_id': product.id,
            'quantity': 1,
        })
        assert response.status_code == 302

        # ステップ5: チェックアウト
        response = client.get(reverse('checkout:review'))
        assert response.status_code == 200
        assert product.name in response.content.decode()

        # ステップ6: 購入完了
        with patch('apps.checkout.services.process_payment') as mock_payment:
            mock_payment.return_value = True
            response = client.post(reverse('checkout:complete'))

        assert response.status_code == 302
        assert Order.objects.filter(user__email='test@example.com').exists()
```

## テストベストプラクティス

### やるべきこと

- **ファクトリを使用**: 手動オブジェクト作成の代わりに
- **テストごとに1つのアサーション**: テストを集中させる
- **説明的なテスト名**: `test_user_cannot_delete_others_post`
- **エッジケースをテスト**: 空の入力、None値、境界条件
- **外部サービスをモック**: 外部APIに依存しない
- **フィクスチャを使用**: 重複を排除
- **パーミッションをテスト**: 認可が機能することを確認
- **テストを高速に保つ**: `--reuse-db`と`--nomigrations`を使用

### やらないこと

- **Django内部をテストしない**: Djangoが動作することを信頼
- **サードパーティコードをテストしない**: ライブラリが動作することを信頼
- **失敗するテストを無視しない**: すべてのテストがパスする必要がある
- **テストを依存させない**: テストはどの順序でも実行できるべき
- **過度にモックしない**: 外部依存関係のみをモック
- **プライベートメソッドをテストしない**: パブリックインターフェースをテスト
- **本番データベースを使用しない**: 常にテストデータベースを使用

## カバレッジ

### カバレッジ設定

```bash
# カバレッジ付きでテストを実行
pytest --cov=apps --cov-report=html --cov-report=term-missing

# HTMLレポートを生成
open htmlcov/index.html
```

### カバレッジ目標

| コンポーネント | 目標カバレッジ |
|--------------|--------------|
| モデル | 90%以上 |
| シリアライザー | 85%以上 |
| ビュー | 80%以上 |
| サービス | 90%以上 |
| ユーティリティ | 80%以上 |
| 全体 | 80%以上 |

## クイックリファレンス

| パターン | 使用法 |
|---------|-------|
| `@pytest.mark.django_db` | データベースアクセスを有効化 |
| `client` | Djangoテストクライアント |
| `api_client` | DRF APIクライアント |
| `factory.create_batch(n)` | 複数オブジェクトを作成 |
| `patch('module.function')` | 外部依存関係をモック |
| `override_settings` | 一時的に設定を変更 |
| `force_authenticate()` | テストで認証をバイパス |
| `assertRedirects` | リダイレクトをチェック |
| `assertTemplateUsed` | テンプレート使用を確認 |
| `mail.outbox` | 送信済みメールをチェック |

覚えておくこと: テストはドキュメントです。良いテストはコードがどのように動作すべきかを説明します。シンプルで、読みやすく、保守可能に保ちましょう。
