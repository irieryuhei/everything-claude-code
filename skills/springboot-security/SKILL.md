---
name: springboot-security
description: Spring Securityベストプラクティス：Java Spring Bootサービスにおける認証/認可、バリデーション、CSRF、シークレット、ヘッダー、レート制限、依存関係セキュリティ。
---

# Spring Bootセキュリティレビュー

認証の追加、入力処理、エンドポイント作成、シークレット処理時に使用。

## 認証

- ステートレスJWTまたは失効リスト付きオペークトークンを優先
- セッションには`httpOnly`、`Secure`、`SameSite=Strict`クッキーを使用
- `OncePerRequestFilter`またはリソースサーバーでトークンを検証

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;

  public JwtAuthFilter(JwtService jwtService) {
    this.jwtService = jwtService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader(HttpHeaders.AUTHORIZATION);
    if (header != null && header.startsWith("Bearer ")) {
      String token = header.substring(7);
      Authentication auth = jwtService.authenticate(token);
      SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
  }
}
```

## 認可

- メソッドセキュリティを有効化: `@EnableMethodSecurity`
- `@PreAuthorize("hasRole('ADMIN')")`または`@PreAuthorize("@authz.canEdit(#id)")`を使用
- デフォルトで拒否；必要なスコープのみを公開

## 入力バリデーション

- コントローラで`@Valid`を使用したBean Validationを使用
- DTOに制約を適用: `@NotBlank`、`@Email`、`@Size`、カスタムバリデータ
- レンダリング前にHTMLをホワイトリストでサニタイズ

## SQLインジェクション防止

- Spring Dataリポジトリまたはパラメータ化クエリを使用
- ネイティブクエリには`:param`バインディングを使用；文字列連結は絶対にしない

## CSRF保護

- ブラウザセッションアプリでは、CSRFを有効のまま保持；フォーム/ヘッダーにトークンを含める
- Bearerトークン付きの純粋なAPIでは、CSRFを無効化しステートレス認証に依存

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## シークレット管理

- ソースにシークレットを入れない；環境変数またはvaultから読み込む
- `application.yml`は認証情報フリーに；プレースホルダーを使用
- トークンとDB認証情報を定期的にローテーション

## セキュリティヘッダー

```java
http
  .headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
      .policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## レート制限

- 高コストエンドポイントにBucket4jまたはゲートウェイレベル制限を適用
- バーストをログ出力しアラート；リトライヒント付きで429を返す

## 依存関係セキュリティ

- CIでOWASP Dependency Check / Snykを実行
- Spring BootとSpring Securityをサポートバージョンに保つ
- 既知のCVEでビルドを失敗させる

## ロギングとPII

- シークレット、トークン、パスワード、完全なPANデータを絶対にログ出力しない
- 機密フィールドを編集；構造化JSONロギングを使用

## ファイルアップロード

- サイズ、コンテンツタイプ、拡張子をバリデート
- Webルート外に保存；必要に応じてスキャン

## リリース前チェックリスト

- [ ] 認証トークンが正しく検証・有効期限切れ
- [ ] すべての機密パスに認可ガード
- [ ] すべての入力がバリデート・サニタイズ済み
- [ ] 文字列連結SQLがない
- [ ] アプリタイプに応じた適切なCSRF態勢
- [ ] シークレットが外部化；コミットなし
- [ ] セキュリティヘッダー設定済み
- [ ] APIにレート制限
- [ ] 依存関係がスキャン・最新
- [ ] ログに機密データなし

**覚えておくこと**: デフォルトで拒否、入力をバリデート、最小権限、設定によるセキュアファースト。
