---
name: java-coding-standards
description: Spring Bootサービス向けJavaコーディング標準：命名規則、イミュータビリティ、Optionalの使用法、ストリーム、例外処理、ジェネリクス、プロジェクト構成。
---

# Javaコーディング標準

Spring Bootサービスにおける、読みやすく保守性の高いJava（17以降）コードのための標準。

## 基本原則

- 巧妙さよりも明確さを優先する
- デフォルトでイミュータブル；共有ミュータブルステートを最小化する
- 意味のある例外で早期に失敗させる
- 一貫した命名とパッケージ構成

## 命名規則

```java
// ✅ クラス/レコード: PascalCase
public class MarketService {}
public record Money(BigDecimal amount, Currency currency) {}

// ✅ メソッド/フィールド: camelCase
private final MarketRepository marketRepository;
public Market findBySlug(String slug) {}

// ✅ 定数: UPPER_SNAKE_CASE
private static final int MAX_PAGE_SIZE = 100;
```

## イミュータビリティ

```java
// ✅ レコードとfinalフィールドを推奨
public record MarketDto(Long id, String name, MarketStatus status) {}

public class Market {
  private final Long id;
  private final String name;
  // ゲッターのみ、セッターなし
}
```

## Optionalの使用法

```java
// ✅ find*メソッドからはOptionalを返す
Optional<Market> market = marketRepository.findBySlug(slug);

// ✅ get()ではなくmap/flatMapを使用
return market
    .map(MarketResponse::from)
    .orElseThrow(() -> new EntityNotFoundException("Market not found"));
```

## ストリームのベストプラクティス

```java
// ✅ 変換にはストリームを使用し、パイプラインを短く保つ
List<String> names = markets.stream()
    .map(Market::name)
    .filter(Objects::nonNull)
    .toList();

// ❌ 複雑なネストされたストリームは避ける；明確さのためにループを使用
```

## 例外処理

- ドメインエラーには非チェック例外を使用；技術的な例外はコンテキストでラップする
- ドメイン固有の例外を作成する（例：`MarketNotFoundException`）
- 再スロー/集中ログ以外での広範な`catch (Exception ex)`は避ける

```java
throw new MarketNotFoundException(slug);
```

## ジェネリクスと型安全性

- 生の型を避ける；ジェネリックパラメータを宣言する
- 再利用可能なユーティリティには境界付きジェネリクスを優先

```java
public <T extends Identifiable> Map<Long, T> indexById(Collection<T> items) { ... }
```

## プロジェクト構成（Maven/Gradle）

```
src/main/java/com/example/app/
  config/
  controller/
  service/
  repository/
  domain/
  dto/
  util/
src/main/resources/
  application.yml
src/test/java/... (mainと同じ構成)
```

## フォーマットとスタイル

- 2または4スペースを一貫して使用（プロジェクト標準に従う）
- 1ファイルに1つのpublicトップレベル型
- メソッドは短く、焦点を絞る；ヘルパーを抽出する
- メンバーの順序：定数、フィールド、コンストラクタ、publicメソッド、protected、private

## 避けるべきコードスメル

- 長いパラメータリスト → DTO/ビルダーを使用
- 深いネスト → 早期リターン
- マジックナンバー → 名前付き定数
- staticなミュータブルステート → 依存性注入を優先
- サイレントなcatchブロック → ログ出力と処理、または再スロー

## ロギング

```java
private static final Logger log = LoggerFactory.getLogger(MarketService.class);
log.info("fetch_market slug={}", slug);
log.error("failed_fetch_market slug={}", slug, ex);
```

## Null処理

- やむを得ない場合のみ`@Nullable`を受け入れる；それ以外は`@NonNull`を使用
- 入力にはBean Validation（`@NotNull`、`@NotBlank`）を使用

## テストの期待事項

- JUnit 5 + AssertJで流暢なアサーション
- モックにはMockito；可能な限り部分モックを避ける
- 決定論的なテストを推奨；隠れたsleepは使用しない

**覚えておくこと**: コードは意図的で、型付けされ、観測可能に保つ。必要性が証明されない限り、マイクロ最適化よりも保守性を優先する。
