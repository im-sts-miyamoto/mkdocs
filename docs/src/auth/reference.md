# 高度なセキュリティ技術とアーキテクチャパターン

> **🔬 このドキュメントの目的**
> 最新のセキュリティ強化技術(DPoP、mTLS、FAPI等)、アーキテクチャパターン、レガシーシステムのモダナイゼーションなど、高度な内容を解説します。

## 📚 関連ドキュメント

- 基礎: **[基礎知識](basics.md)** - 認証・認可の基本
- 実装: **[実装ガイド](implementation.md)** - 具体的な実装方法

---

??? note "2025年の最新動向"

    ### サードパーティCookie廃止の影響

    **影響:**

    - ❌ 従来のサイレントリフレッシュ(隠しiframe)が使えない
    - ❌ IdPのサードパーティCookie依存SSOが困難
    - ❌ クロスサイトでのセッション維持が制限

    **対策:**

    - ✅ **BFFパターン** - サーバーサイドでトークン管理
    - ✅ **同一サイトCookie運用**
    - ✅ **FedCM(Federated Credential Management)** - ブラウザネイティブSSO
    - ✅ **Authorization Code + PKCE**

    ### OAuth 2.1への移行

    **主な変更点:**

    - ✅ **PKCE が全フローで必須**
    - ❌ Implicit Flow 廃止
    - ❌ Resource Owner Password Credentials Flow 廃止
    - ✅ リフレッシュトークンローテーション推奨
    - ✅ リダイレクトURI完全一致必須

---

??? note "最新セキュリティ強化技術"

    ### DPoP (Demonstrating Proof-of-Possession)

    **目的:** トークン窃取後の不正利用を防ぐ

    **仕組み:**

    ```mermaid
    sequenceDiagram
        participant Client
        participant AuthServer
        participant API

        Client->>Client: 1. 秘密鍵・公開鍵ペア生成
        Client->>AuthServer: 2. トークン要求 + DPoP Proof(公開鍵含む)
        AuthServer->>Client: 3. DPoP-bound Token発行
        Client->>API: 4. Token + DPoP Proof(署名付き)
        API->>API: 5. 署名検証・公開鍵一致確認
        API->>Client: 6. データ返却
    ```

    **特徴:**

    - トークンと公開鍵を紐付け
    - リクエストごとに署名付きProofを生成
    - 窃取されたトークンは秘密鍵がないため使用不可

    **実装例:**

    ```javascript
    // クライアント側
    import * as jose from 'jose';

    // 1. 鍵ペア生成
    const { publicKey, privateKey } = await jose.generateKeyPair('ES256');

    // 2. DPoP Proof 生成
    async function createDPoPProof(method, url, accessToken) {
      const dpopProof = await new jose.SignJWT({
        htm: method,
        htu: url,
        iat: Math.floor(Date.now() / 1000),
        jti: crypto.randomUUID(),
      })
        .setProtectedHeader({
          alg: 'ES256',
          typ: 'dpop+jwt',
          jwk: await jose.exportJWK(publicKey)
        })
        .sign(privateKey);

      return dpopProof;
    }

    // 3. API呼び出し
    const dpopProof = await createDPoPProof('GET', 'https://api.example.com/data');
    const response = await fetch('https://api.example.com/data', {
      headers: {
        'Authorization': `DPoP ${accessToken}`,
        'DPoP': dpopProof
      }
    });
    ```

    ---

    ### mTLS (Mutual TLS)

    **目的:** クライアント認証の強化

    **仕組み:**

    - サーバーとクライアント双方が証明書を提示
    - TLSレベルでクライアントの正当性を検証

    **適用例:**

    - 金融機関のAPI
    - 高セキュリティが要求されるB2B連携
    - IoTデバイス認証

    **設定例(Spring Boot):**

    ```yaml
    # application.yml
    server:
      ssl:
        enabled: true
        key-store: classpath:server-keystore.p12
        key-store-password: changeit
        key-store-type: PKCS12
        client-auth: need  # クライアント証明書必須
        trust-store: classpath:truststore.p12
        trust-store-password: changeit
    ```

    ---

    ### PAR (Pushed Authorization Requests)

    **目的:** 認可リクエストを安全に送信

    **仕組み:**

    ```mermaid
    sequenceDiagram
        participant Client
        participant AuthServer

        Client->>AuthServer: 1. POST /as/par<br/>(認可パラメータ)
        AuthServer->>Client: 2. request_uri返却
        Client->>AuthServer: 3. GET /authorize<br/>?request_uri=xxx
        AuthServer->>Client: 4. 認可コード
    ```

    **利点:**

    - ✅ 認可リクエストがURLに露出しない
    - ✅ リクエストの完全性を保証
    - ✅ 大きなリクエストにも対応

    ---

    ### FAPI (Financial-grade API)

    **目的:** 金融グレードのセキュリティ

    **要件:**

    - ✅ **mTLS または DPoP 必須**
    - ✅ **PAR 必須**
    - ✅ **PKCE 必須**
    - ✅ **リフレッシュトークンローテーション**
    - ✅ **短寿命トークン(5分以内)**
    - ✅ **JARM(JWT Secured Authorization Response Mode)**

    **適用:**

    - オープンバンキングAPI
    - 決済サービス
    - 機密性の高い医療データ

---

??? note "アーキテクチャパターン"

    ### パターン1: BFF (Backend for Frontend)

    **構成:**

    ```mermaid
    graph LR
        Browser --> BFF[Next.js BFF]
        BFF --> API[Spring Boot API]
        BFF --> IdP[Cognito]
    ```

    **特徴:**

    - ✅ トークンがブラウザに露出しない
    - ✅ Cookie管理がシンプル
    - ✅ サードパーティCookie不要

    **適用:** SPA、モバイルアプリ(推奨)

    ---

    ### パターン2: トークンリレー

    **構成:**

    ```mermaid
    graph LR
        SPA --> Gateway[API Gateway]
        Gateway --> API[Microservices]
        Gateway --> IdP[IdP]
    ```

    **特徴:**

    - トークンをAPI Gateway経由で各サービスに転送
    - Gateway でトークン検証・変換

    **適用:** マイクロサービスアーキテクチャ

    ---

    ### パターン3: Zero Trust

    **原則:**

    - **決して信頼せず、常に検証**
    - すべてのリクエストで認証・認可
    - 最小権限の原則

    **実装:**

    ```mermaid
    graph TB
        Client --> Gateway
        Gateway --> AuthZ[認可エンジン]
        Gateway --> Service1
        Gateway --> Service2
        Service1 --> AuthZ
        Service2 --> AuthZ
    ```

    **技術:**

    - Service Mesh (Istio、Linkerd)
    - OPA (Open Policy Agent)
    - SPIFFE/SPIRE

---

??? note "WebAuthn / パスキー"

    ### パスキーの仕組み

    ```mermaid
    sequenceDiagram
        participant User
        participant Browser
        participant Server
        participant Authenticator

        User->>Browser: 登録開始
        Browser->>Server: 登録リクエスト
        Server->>Browser: チャレンジ
        Browser->>Authenticator: 生体認証要求
        Authenticator->>User: 指紋/顔認証
        User->>Authenticator: 認証成功
        Authenticator->>Browser: 公開鍵・署名
        Browser->>Server: 公開鍵送信
        Server->>Server: 公開鍵保存
    ```

    **利点:**

    - ✅ **フィッシング耐性** - ドメイン紐付け
    - ✅ **パスワード不要**
    - ✅ **生体認証統合**

    **実装例:**

    ```javascript
    // 登録
    const credential = await navigator.credentials.create({
      publicKey: {
        challenge: Uint8Array.from(challenge, c => c.charCodeAt(0)),
        rp: {
          name: "Example Corp",
          id: "example.com"
        },
        user: {
          id: Uint8Array.from(userId, c => c.charCodeAt(0)),
          name: "user@example.com",
          displayName: "User Name"
        },
        pubKeyCredParams: [{ alg: -7, type: "public-key" }],
        authenticatorSelection: {
          userVerification: "required"
        },
        timeout: 60000
      }
    });

    // 認証
    const assertion = await navigator.credentials.get({
      publicKey: {
        challenge: Uint8Array.from(challenge, c => c.charCodeAt(0)),
        rpId: "example.com",
        userVerification: "required"
      }
    });
    ```

---

??? note "レガシーシステムのモダナイゼーション"

    ### セッションベースからトークンベースへ

    **段階的移行:**

    ```mermaid
    graph TD
        Phase1[Phase 1: 両方サポート] --> Phase2[Phase 2: トークン優先]
        Phase2 --> Phase3[Phase 3: セッション廃止]
    ```

    **Phase 1: ハイブリッド**

    ```java
    // セッションとトークン両方をサポート
    @Component
    public class HybridAuthFilter extends OncePerRequestFilter {
        @Override
        protected void doFilterInternal(HttpServletRequest request,
                                        HttpServletResponse response,
                                        FilterChain filterChain) {
            // 1. トークン認証を試行
            String token = extractToken(request);
            if (token != null && validateToken(token)) {
                setAuthenticationFromToken(token);
                filterChain.doFilter(request, response);
                return;
            }

            // 2. セッション認証にフォールバック
            HttpSession session = request.getSession(false);
            if (session != null) {
                setAuthenticationFromSession(session);
            }

            filterChain.doFilter(request, response);
        }
    }
    ```

    **Phase 2: トークン優先**

    - 新機能はトークンのみ
    - 既存機能は両方サポート

    **Phase 3: セッション廃止**

    - すべてトークンベース
    - セッションストレージ削除

    ---

    ### 自前認証からOAuth/OIDCへ

    **移行戦略:**

    1. **IdP導入** (Cognito、Auth0等)
    2. **既存ユーザーのマイグレーション**
    3. **段階的カットオーバー**

    **ユーザーマイグレーション:**

    ```javascript
    // Cognito Migration Lambda
    export const handler = async (event) => {
      const { userName, password } = event.request;

      // 既存DBで認証
      const user = await authenticateFromLegacyDB(userName, password);

      if (user) {
        return {
          response: {
            userAttributes: {
              email: user.email,
              email_verified: "true"
            },
            finalUserStatus: "CONFIRMED",
            messageAction: "SUPPRESS"
          }
        };
      }

      throw new Error("Invalid credentials");
    };
    ```

---

??? note "トラブルシューティング"

    ### トークン検証エラー

    ```bash
    # 原因
    - 署名検証失敗: JWKS URI 誤り
    - issuer 不一致: 設定確認
    - 有効期限切れ: exp クレーム確認
    - audience 不一致: aud クレーム確認

    # 対処
    - JWTデバッグツール使用 (jwt.io)
    - ログで詳細エラー確認
    ```

    ### CORS エラー

    ```bash
    # 原因
    - allowedOrigins 設定漏れ
    - Preflight リクエスト失敗

    # 対処
    - Spring Boot: CorsConfiguration 確認
    - allowCredentials: true 設定
    ```

    ### リフレッシュトークン失敗

    ```bash
    # 原因
    - ローテーション設定誤り
    - 有効期限切れ
    - Cognito 側の設定

    # 対処
    - Cognito: enable_token_revocation 確認
    - リフレッシュトークン有効期限確認
    ```

---

## 参考資料

### 仕様・標準

- [OAuth 2.1 Draft](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [RFC 9449 - DPoP](https://datatracker.ietf.org/doc/html/rfc9449)
- [FAPI 2.0](https://openid.net/specs/fapi-2_0.html)

### ツール

- [jwt.io](https://jwt.io/) - JWT デバッガ
- [OAuth Debugger](https://oauthdebugger.com/) - OAuth フローテスト

---

**最終更新:** 2025年10月
**対象読者:** セキュリティアーキテクト、上級開発者
