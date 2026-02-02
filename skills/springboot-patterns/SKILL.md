---
name: springboot-patterns
description: Spring Bootアーキテクチャパターン、REST API設計、レイヤードサービス、データアクセス、キャッシング、非同期処理、ロギング。Java Spring Bootバックエンド作業に使用。
---

# Spring Boot開発パターン

スケーラブルで本番グレードのサービスのためのSpring Bootアーキテクチャとパターン。

## REST API構造

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create(@Valid @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## リポジトリパターン（Spring Data JPA）

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive(@Param("status") MarketStatus status, Pageable pageable);
}
```

## トランザクション付きサービス層

```java
@Service
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTOとバリデーション

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## 例外処理

```java
@ControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    return ResponseEntity.badRequest().body(ApiError.validation(message));
  }

  @ExceptionHandler(AccessDeniedException.class)
  ResponseEntity<ApiError> handleAccessDenied() {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ApiError.of("Forbidden"));
  }

  @ExceptionHandler(Exception.class)
  ResponseEntity<ApiError> handleGeneric(Exception ex) {
    // 予期しないエラーはスタックトレース付きでログ
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiError.of("Internal server error"));
  }
}
```

## キャッシング

設定クラスに`@EnableCaching`が必要。

```java
@Service
public class MarketCacheService {
  private final MarketRepository repo;

  public MarketCacheService(MarketRepository repo) {
    this.repo = repo;
  }

  @Cacheable(value = "market", key = "#id")
  public Market getById(Long id) {
    return repo.findById(id)
        .map(Market::from)
        .orElseThrow(() -> new EntityNotFoundException("Market not found"));
  }

  @CacheEvict(value = "market", key = "#id")
  public void evict(Long id) {}
}
```

## 非同期処理

設定クラスに`@EnableAsync`が必要。

```java
@Service
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // メール/SMS送信
    return CompletableFuture.completedFuture(null);
  }
}
```

## ロギング（SLF4J）

```java
@Service
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  public Report generate(Long marketId) {
    log.info("generate_report marketId={}", marketId);
    try {
      // ロジック
    } catch (Exception ex) {
      log.error("generate_report_failed marketId={}", marketId, ex);
      throw ex;
    }
    return new Report();
  }
}
```

## ミドルウェア / フィルター

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## ページネーションとソート

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## エラー耐性のある外部呼び出し

```java
public <T> T withRetry(Supplier<T> supplier, int maxRetries) {
  int attempts = 0;
  while (true) {
    try {
      return supplier.get();
    } catch (Exception ex) {
      attempts++;
      if (attempts >= maxRetries) {
        throw ex;
      }
      try {
        Thread.sleep((long) Math.pow(2, attempts) * 100L);
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ex;
      }
    }
  }
}
```

## レート制限（Filter + Bucket4j）

**セキュリティ注意**: `X-Forwarded-For`ヘッダーはデフォルトで信頼できません。クライアントが偽装できるためです。
以下の条件を満たす場合のみ転送ヘッダーを使用してください：
1. 信頼できるリバースプロキシ（nginx、AWS ALBなど）の背後にある
2. `ForwardedHeaderFilter`をBeanとして登録している
3. application propertiesで`server.forward-headers-strategy=NATIVE`または`FRAMEWORK`を設定している
4. プロキシが`X-Forwarded-For`ヘッダーを追加ではなく上書きするよう設定されている

`ForwardedHeaderFilter`が適切に設定されている場合、`request.getRemoteAddr()`は転送ヘッダーから正しいクライアントIPを自動的に返します。この設定がない場合は`request.getRemoteAddr()`を直接使用してください - 直接接続のIPが唯一信頼できる値です。

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  /*
   * セキュリティ: このフィルターはレート制限のクライアント識別にrequest.getRemoteAddr()を使用します。
   *
   * アプリケーションがリバースプロキシ（nginx、AWS ALBなど）の背後にある場合、正確なクライアントIP検出のために
   * Springが転送ヘッダーを適切に処理するよう設定する必要があります：
   *
   * 1. application.properties/yamlでserver.forward-headers-strategy=NATIVE（クラウドプラットフォーム用）
   *    またはFRAMEWORKを設定
   * 2. FRAMEWORK戦略を使用する場合、ForwardedHeaderFilterを登録：
   *
   *    @Bean
   *    ForwardedHeaderFilter forwardedHeaderFilter() {
   *        return new ForwardedHeaderFilter();
   *    }
   *
   * 3. プロキシがスプーフィング防止のためにX-Forwarded-Forヘッダーを追加ではなく上書きするよう確認
   * 4. server.tomcat.remoteip.trusted-proxiesまたはコンテナ相当を設定
   *
   * この設定がない場合、request.getRemoteAddr()はクライアントIPではなくプロキシIPを返します。
   * X-Forwarded-Forを直接読み取らないでください - 信頼できるプロキシ処理なしでは簡単に偽装できます。
   */
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // getRemoteAddr()を使用 - ForwardedHeaderFilterが設定されていれば正しいクライアントIPを返し、
    // そうでなければ直接接続IPを返します。X-Forwarded-Forヘッダーを適切なプロキシ設定なしで
    // 直接信頼しないでください。
    String clientIp = request.getRemoteAddr();

    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build());

    if (bucket.tryConsume(1)) {
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
    }
  }
}
```

## バックグラウンドジョブ

Springの`@Scheduled`を使用するか、キュー（Kafka、SQS、RabbitMQ）と統合。ハンドラは冪等で観測可能に保つ。

## オブザーバビリティ

- 構造化ロギング（JSON）はLogbackエンコーダ経由
- メトリクス: Micrometer + Prometheus/OTel
- トレーシング: Micrometer Tracing（OpenTelemetryまたはBraveバックエンド）

## 本番デフォルト

- コンストラクタインジェクションを優先、フィールドインジェクションを避ける
- RFC 7807エラー用に`spring.mvc.problemdetails.enabled=true`を有効化（Spring Boot 3以降）
- HikariCPプールサイズをワークロードに合わせて設定、タイムアウトを設定
- クエリには`@Transactional(readOnly = true)`を使用
- 適切な場所で`@NonNull`と`Optional`でnull安全性を強制

**覚えておくこと**: コントローラは薄く、サービスは焦点を絞り、リポジトリはシンプルに、エラーは集中処理。保守性とテスト容易性を最適化する。
