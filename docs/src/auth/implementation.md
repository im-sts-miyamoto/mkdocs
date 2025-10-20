# Next.js + Spring Boot + Cognito 実装ガイド

> **🚀 このドキュメントの目的**
> Next.js、Spring Boot、AWS Cognitoを使った認証機能の実装方法を、セキュリティ対策と共に解説します。BFFパターンを採用し、2025年のベストプラクティスに準拠した実装を提供します。

## 📚 関連ドキュメント

- 前提知識: **[基礎知識](basics.md)** - 認証・認可の基本概念

---

## システム構成

??? note "アーキテクチャ(BFFパターン)"

    ```mermaid
    graph TB
        Browser[🌐 ブラウザ]
        Next[⚡ Next.js BFF<br/>トークン管理<br/>APIプロキシ]
        Cognito[🔐 AWS Cognito<br/>認証・トークン発行]
        Spring[☕ Spring Boot API<br/>ビジネスロジック]

        Browser <-->|Cookie<br/>HttpOnly/Secure| Next
        Next <-->|OAuth 2.0<br/>Code+PKCE| Cognito
        Next -->|Bearer Token| Spring
    ```

    **各コンポーネントの役割:**

    | コンポーネント | 責務 |
    |-------------|------|
    | **Next.js BFF** | 認証フロー管理、トークン管理、APIプロキシ |
    | **AWS Cognito** | 認証・認可、トークン発行、MFA |
    | **Spring Boot** | ビジネスロジック、トークン検証 |

    **BFFパターンの利点:**

    - ✅ トークンがブラウザに露出しない(XSS対策)
    - ✅ サードパーティCookie廃止に対応
    - ✅ CORS設定がシンプル
    - ✅ リフレッシュトークンを安全に管理

---

## 認証フロー

??? note "初回ログイン"

    ```mermaid
    sequenceDiagram
        participant Browser
        participant Next as Next.js BFF
        participant Cognito
        participant Spring as Spring Boot

        Browser->>Next: 1. ログインクリック
        Note over Next: PKCE生成
        Next->>Cognito: 2. 認可リクエスト<br/>(code_challenge, state)
        Cognito->>Browser: 3. ログイン画面
        Browser->>Cognito: 4. 認証情報入力
        Cognito->>Next: 5. 認可コード
        Next->>Cognito: 6. トークン要求<br/>(code, code_verifier)
        Cognito->>Next: 7. トークン発行
        Note over Next: サーバーセッションに保存
        Next->>Browser: 8. HttpOnly Cookie設定
        Browser->>Next: 9. APIリクエスト
        Next->>Spring: 10. Bearer Token
        Spring->>Next: 11. データ
        Next->>Browser: 12. レスポンス
    ```

---

## AWS Cognito 設定

??? note "ユーザープール作成"

    ```bash
    # 認証フロー
    - Email/Password
    - MFA: Optional(TOTP推奨)
    - パスワードポリシー: 強力(12文字以上、大小英数記号)
    ```

    ### アプリクライアント設定

    ```hcl
    # Terraform例
    resource "aws_cognito_user_pool_client" "app" {
      name         = "nextjs-app"
      user_pool_id = aws_cognito_user_pool.main.id

      # Authorization Code + PKCE
      allowed_oauth_flows = ["code"]
      allowed_oauth_flows_user_pool_client = true

      # スコープ
      allowed_oauth_scopes = [
        "openid",
        "email",
        "profile"
      ]

      # コールバックURL
      callback_urls = [
        "http://localhost:3000/api/auth/callback/cognito",
        "https://yourapp.com/api/auth/callback/cognito"
      ]

      # トークン有効期限
      access_token_validity  = 1   # 1時間
      id_token_validity      = 1   # 1時間
      refresh_token_validity = 30  # 30日

      # セキュリティ
      prevent_user_existence_errors = "ENABLED"
      enable_token_revocation       = true
    }
    ```

---

## Next.js 実装

??? note "1. NextAuth 設定"

    ```bash
    npm install next-auth
    ```

    ```typescript
    // app/api/auth/[...nextauth]/route.ts
    import NextAuth from "next-auth";
    import CognitoProvider from "next-auth/providers/cognito";

    export const authOptions = {
      providers: [
        CognitoProvider({
          clientId: process.env.COGNITO_CLIENT_ID!,
          clientSecret: process.env.COGNITO_CLIENT_SECRET!,
          issuer: process.env.COGNITO_ISSUER!,
          checks: ["pkce", "state"], // PKCE + state 自動設定
        })
      ],

      session: {
        strategy: "jwt",
        maxAge: 30 * 24 * 60 * 60, // 30日
      },

      callbacks: {
        async jwt({ token, account }) {
          if (account) {
            token.accessToken = account.access_token;
            token.idToken = account.id_token;
            token.refreshToken = account.refresh_token;
            token.expiresAt = account.expires_at;
          }

          // トークンリフレッシュ
          if (Date.now() < token.expiresAt * 1000) {
            return token;
          }

          return refreshAccessToken(token);
        },

        async session({ session, token }) {
          session.accessToken = token.accessToken;
          session.error = token.error;
          return session;
        }
      },
    };

    async function refreshAccessToken(token: any) {
      try {
        const response = await fetch(
          `${process.env.COGNITO_ISSUER}/oauth2/token`,
          {
            method: "POST",
            headers: { "Content-Type": "application/x-www-form-urlencoded" },
            body: new URLSearchParams({
              grant_type: "refresh_token",
              client_id: process.env.COGNITO_CLIENT_ID!,
              refresh_token: token.refreshToken,
            }),
          }
        );

        const tokens = await response.json();

        return {
          ...token,
          accessToken: tokens.access_token,
          idToken: tokens.id_token,
          expiresAt: Date.now() / 1000 + tokens.expires_in,
        };
      } catch (error) {
        return { ...token, error: "RefreshAccessTokenError" };
      }
    }

    const handler = NextAuth(authOptions);
    export { handler as GET, handler as POST };
    ```

??? note "2. API プロキシ(BFF)"

    ```typescript
    // app/api/users/profile/route.ts
    import { getServerSession } from "next-auth";
    import { authOptions } from "@/app/api/auth/[...nextauth]/route";
    import { NextResponse } from "next/server";

    export async function GET() {
      const session = await getServerSession(authOptions);

      if (!session?.idToken) {
        return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
      }

      try {
        const response = await fetch(
          `${process.env.API_BASE_URL}/users/profile`,
          {
            headers: {
              Authorization: `Bearer ${session.idToken}`,
            },
          }
        );

        const data = await response.json();
        return NextResponse.json(data);
      } catch (error) {
        return NextResponse.json(
          { error: "API request failed" },
          { status: 500 }
        );
      }
    }
    ```

??? note "3. クライアント側"

    ```typescript
    // app/page.tsx
    "use client";

    import { useSession, signIn, signOut } from "next-auth/react";

    export default function Home() {
      const { data: session, status } = useSession();

      if (status === "loading") return <div>Loading...</div>;

      if (!session) {
        return <button onClick={() => signIn("cognito")}>ログイン</button>;
      }

      return (
        <div>
          <p>ようこそ、{session.user?.email}さん</p>
          <button onClick={() => signOut()}>ログアウト</button>
        </div>
      );
    }
    ```

---

## Spring Boot 実装

??? note "1. 依存関係"

    ```xml
    <!-- pom.xml -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    ```

??? note "2. セキュリティ設定"

    ```java
    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {

        @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
        private String issuerUri;

        @Bean
        public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
            http
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .csrf(csrf -> csrf.disable())
                .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/api/**").authenticated()
                )
                .oauth2ResourceServer(oauth2 -> oauth2
                    .jwt(jwt -> jwt.decoder(jwtDecoder()))
                );

            return http.build();
        }

        @Bean
        public JwtDecoder jwtDecoder() {
            return JwtDecoders.fromIssuerLocation(issuerUri);
        }

        @Bean
        public CorsConfigurationSource corsConfigurationSource() {
            CorsConfiguration config = new CorsConfiguration();
            config.setAllowedOrigins(List.of("http://localhost:3000"));
            config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
            config.setAllowedHeaders(List.of("*"));
            config.setAllowCredentials(true);

            UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
            source.registerCorsConfiguration("/**", config);
            return source;
        }
    }
    ```

??? note "3. API実装"

    ```java
    @RestController
    @RequestMapping("/api/users")
    public class UserController {

        @GetMapping("/profile")
        public ResponseEntity<UserProfile> getProfile(
            @AuthenticationPrincipal Jwt jwt
        ) {
            String userId = jwt.getSubject();
            String email = jwt.getClaimAsString("email");

            UserProfile profile = new UserProfile(userId, email);
            return ResponseEntity.ok(profile);
        }
    }
    ```

??? note "4. 設定ファイル"

    ```yaml
    # application.yml
    spring:
      security:
        oauth2:
          resourceserver:
            jwt:
              issuer-uri: https://cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_xxxxx
              jwk-set-uri: https://cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_xxxxx/.well-known/jwks.json
    ```

---

## セキュリティ対策

??? danger "Cookie 設定"

    ```javascript
    // Next.js
    Set-Cookie: session_id=xxx;
      HttpOnly;         // XSS対策
      Secure;           // HTTPS必須
      SameSite=Strict;  // CSRF対策
      Path=/;
      Max-Age=2592000   // 30日
    ```

??? danger "CSP 設定"

    ```typescript
    // next.config.js
    const securityHeaders = [
      {
        key: 'Content-Security-Policy',
        value: "default-src 'self'; script-src 'self' 'unsafe-inline';"
      },
      {
        key: 'X-Frame-Options',
        value: 'DENY'
      },
      {
        key: 'X-Content-Type-Options',
        value: 'nosniff'
      }
    ];
    ```

??? warning "トークン保管場所"

    | 場所 | 用途 | セキュリティ |
    |------|------|------------|
    | **Next.js サーバー** | 全トークン | ✅ 最も安全 |
    | **HttpOnly Cookie** | セッションID | ✅ 安全 |
    | ~~LocalStorage~~ | なし | ❌ 使用禁止 |

??? danger "脅威と対策"

    | 脅威 | 対策 | 実装 |
    |------|------|------|
    | **XSS** | HttpOnly Cookie、CSP | 🔴 必須 |
    | **CSRF** | SameSite=Strict、state | 🔴 必須 |
    | **トークン窃取** | 短寿命(1時間)、HTTPS | 🔴 必須 |
    | **フィッシング** | WebAuthn/パスキー | 🟡 推奨 |

---

## チェックリスト

??? success "AWS Cognito"

    - [ ] ユーザープール作成
    - [ ] アプリクライアント設定(Authorization Code + PKCE)
    - [ ] トークン有効期限設定(1時間/30日)
    - [ ] コールバックURL登録
    - [ ] MFA有効化(推奨)

??? success "Next.js"

    - [ ] NextAuth インストール・設定
    - [ ] 環境変数設定
    - [ ] API プロキシ実装
    - [ ] Cookie セキュリティ設定
    - [ ] CSP ヘッダー設定

??? success "Spring Boot"

    - [ ] OAuth2 Resource Server 設定
    - [ ] JWT Decoder 設定
    - [ ] CORS 設定
    - [ ] API エンドポイント実装

??? success "セキュリティ"

    - [ ] HTTPS 有効化
    - [ ] HttpOnly Cookie 使用
    - [ ] SameSite=Strict 設定
    - [ ] CSP 設定
    - [ ] トークン有効期限確認

---

## トラブルシューティング

??? warning "401 Unauthorized"

    ```bash
    # トークン検証失敗
    - issuer-uri が正しいか確認
    - トークンの有効期限を確認
    - JWKS URI にアクセス可能か確認
    ```

??? warning "CORS エラー"

    ```bash
    # Next.js と Spring Boot の設定確認
    - Spring Boot: allowedOrigins に Next.js URL
    - allowCredentials: true
    ```

??? warning "トークンリフレッシュ失敗"

    ```bash
    # リフレッシュトークンの確認
    - 有効期限内か
    - Cognito でローテーション設定確認
    ```

---

## 次のステップ

この実装ガイドで、Next.js + Spring Boot + Cognito を使った認証機能の実装方法を学びました。さらに理解を深めるには、**[基礎知識](basics.md)** に戻って概念を再確認することをお勧めします。

---

**最終更新:** 2025年10月
**対象読者:** 認証機能を実装する開発者(中級)
