# 認証・認可の基礎知識

> **🎯 このドキュメントの目的**
> 認証・認可の基本概念から、OAuth 2.0/OpenID Connectの仕組みまで、体系的に理解できます。実装に必要な本質的な知識に絞り、簡潔に解説します。

## 📚 関連ドキュメント

- 前提知識: **[Webアプリケーションの基礎](../web-basics/index.md)** - サーバーサイド・クライアントサイドの違い、アーキテクチャパターン
- 次のステップ: **[実装ガイド](implementation.md)** - Next.js + Spring Boot + Cognito での実装

---

## 認証と認可

??? info "認証(Authentication) - 「誰か」を確認"

    **「あなたは誰ですか?」** という問いに答えるプロセス。

    - 例: ログイン、パスワード入力
    - HTTP ステータス: `401 Unauthorized`

??? info "認可(Authorization) - 「何ができるか」を確認"

    **「あなたは何ができますか?」** という問いに答えるプロセス。

    - 例: 管理者画面へのアクセス、ファイルの編集権限
    - HTTP ステータス: `403 Forbidden`

??? example "実際の流れ"

    ### 認証(Authentication)の流れ

    ユーザーがログイン画面でIDとパスワードを入力します。アプリケーションはその情報が正しいかを確認し、正しければ認証成功、間違っていれば`401 Unauthorized`エラーを返します。

    ```mermaid
    sequenceDiagram
        participant User as ユーザー
        participant App as アプリ

        User->>App: ログイン(ID/パスワード)
        alt 認証成功
            App->>User: ✅ 認証成功<br/>(ホーム画面へ遷移等の処理)
        else 認証失敗
            App->>User: ❌ 401 Unauthorized<br/>(ID/パスワードが間違っている)
        end
    ```

    ### 認可(Authorization)の流れ

    認証後、ユーザーが特定のリソース（データや機能）にアクセスしようとします。アプリケーションはそのユーザーに権限があるかをチェックし、権限があればデータを表示、なければ`403 Forbidden`エラーを返します。

    ```mermaid
    sequenceDiagram
        participant User as ユーザー
        participant App as アプリ

        User->>App: リソースにアクセス
        App->>App: 権限チェック(認可)
        alt 権限あり
            App->>User: ✅ データ表示<br/>(権限がある)
        else 権限なし
            App->>User: ❌ 403 Forbidden<br/>(権限がない)
        end
    ```

---

## セッション vs トークン

??? success "セッションベース認証"

    ```mermaid
    graph LR
        Browser[ブラウザ] -->|Cookie: session_id| Server[サーバー]
        Server -->|セッション取得| Store[(セッションストア)]
    ```

    **特徴:**

    - サーバー側でセッション情報を保持(Stateful)
    - クライアントにはセッションIDのみ送信
    - **認証自体は外部IdPに委譲可能**（推奨）

    **メリット:**

    - ✅ サーバーで即座に無効化可能
    - ✅ セッション情報を完全管理

    **デメリット:**

    - ❌ ストレージが必要
    - ❌ 水平スケーリング時に共有が必要

    ### セッションの保存場所

    Webサーバーのセッション管理は、保存場所によって特性が異なります。

    ??? example "1. メモリ(開発環境向け)"

        多くのフレームワークのデフォルト設定。開発時や単一サーバー構成で使用。

        ```javascript
        // Express.js
        const session = require('express-session');
        app.use(session({
          secret: 'secret-key',
          // デフォルト: メモリに保存
        }));
        ```

        **メリット:**

        - ✅ 高速アクセス
        - ✅ 設定不要

        **デメリット:**

        - ❌ サーバー再起動でセッション消失
        - ❌ 水平スケーリング不可
        - ❌ メモリ枯渇のリスク

        **適用:** 開発環境、単一サーバー構成

    ??? success "2. インメモリDB - Redis(本番推奨)"

        **最も一般的な本番環境の構成。**高速で永続化も可能。

        ```javascript
        // Express + Redis
        const RedisStore = require('connect-redis').default;
        app.use(session({
          store: new RedisStore({ client: redisClient }),
          secret: 'secret-key'
        }));
        ```

        ```java
        // Spring Boot + Redis
        spring:
          session:
            store-type: redis
        ```

        **メリット:**

        - ✅ 超高速（メモリベース）
        - ✅ 水平スケーリング対応
        - ✅ 永続化可能
        - ✅ TTL（自動削除）サポート

        **デメリット:**

        - ⚠️ 別途Redis環境が必要

        **適用:** 本番環境、複数サーバー構成

    ??? example "3. インメモリDB - Memcached"

        Redisと似ているが、永続化機能がないシンプルなキャッシュ。

        **Redisとの違い:**

        | 項目 | Redis | Memcached |
        |------|-------|-----------|
        | 永続化 | ✅ 可能 | ❌ 不可 |
        | データ構造 | 豊富(リスト、セット等) | シンプル(Key-Value) |
        | レプリケーション | ✅ 対応 | ❌ 非対応 |
        | 用途 | セッション、キャッシュ | 単純なキャッシュ |

        **適用:** シンプルなキャッシュ用途、永続化不要な場合

    ??? example "4. RDB - PostgreSQL、MySQL等"

        リレーショナルデータベースにセッションを保存。

        **メリット:**

        - ✅ 既存DBを活用可能
        - ✅ 永続化・バックアップが容易
        - ✅ トランザクション対応

        **デメリット:**

        - ❌ Redisより遅い
        - ❌ 頻繁な読み書きでDB負荷

        **適用:** 既にRDBを使用している環境、厳密なデータ管理が必要

    ??? example "5. NoSQL - MongoDB、DynamoDB等"

        NoSQLデータベースにセッションを保存。

        **MongoDB:**

        - ドキュメント指向DB
        - TTL機能でセッション自動削除可能
        - 既にMongoDBを使用している場合に最適

        **DynamoDB(AWS):**

        - AWSのマネージドNoSQL
        - TTL機能あり
        - AWS環境なら運用が容易
        - サーバーレス構成に適合

        **適用:** NoSQL環境、AWS/クラウド環境

    ### 水平スケーリング時の課題

    ```mermaid
    graph TB
        User[ユーザー]
        LB[ロードバランサー]
        Server1[Server 1<br/>メモリ]
        Server2[Server 2<br/>メモリ]
        Redis[(Redis<br/>共有)]

        User -->|1回目| LB
        LB --> Server1
        User -->|2回目| LB
        LB --> Server2

        Server1 <-.->|❌ 共有不可| Server2
        Server1 <-->|✅ 共有可能| Redis
        Server2 <-->|✅ 共有可能| Redis
    ```

    **問題:** メモリ保存では、別サーバーでセッション取得不可

    **解決策:**

    - ✅ **Redis等の外部ストレージ**(推奨)
    - ⚠️ **スティッキーセッション**(非推奨: 障害時に問題)

    **環境別の推奨:**

    | 環境 | 推奨保存先 | 理由 |
    |------|----------|------|
    | **開発環境** | メモリ | シンプル、高速 |
    | **本番(単一サーバー)** | メモリ or Redis | 要件次第 |
    | **本番(複数サーバー)** | Redis | 水平スケーリング必須 |
    | **マイクロサービス** | JWT(ステートレス) | セッションストア不要 |

??? success "トークンベース認証(JWT)"

    ```mermaid
    graph LR
        Browser[ブラウザ] -->|Bearer<br/>Token| Server[サーバー]
        Server -->|公開鍵取得| AuthServer[認可サーバー]
        Server -->|署名検証| Server
    ```

    **特徴:**

    - サーバーは状態を持たない(Stateless)
    - トークン自体にユーザー情報を含む

    **JWT の構造:**

    ```
    Header.Payload.Signature
    ```

    - **Header**: アルゴリズム情報(例: HS256, RS256)
    - **Payload**: ユーザー情報、有効期限
    - **Signature**: 改ざん検知用の署名

    **メリット:**

    - ✅ ステートレス(状態不要)
    - ✅ 水平スケーリングが容易
    - ✅ マイクロサービス間で共有可能

    **デメリット:**

    - ❌ 即座な無効化が困難
    - ❌ トークンサイズが大きい

??? abstract "比較表"

    | 項目 | セッション | トークン(JWT) |
    |------|----------|--------------|
    | **状態管理** | Stateful | Stateless |
    | **保存場所(サーバー)** | 必要(Redis等) | 不要 |
    | **無効化** | 即座に可能 | 困難 |
    | **水平スケーリング** | 共有ストア必要 | 容易 |
    | **典型的なアーキテクチャ** | モノリシック<br/>サーバーサイドレンダリング | SPA<br/>マイクロサービス |
    | **構成例** | フロント+バック統合<br/>(Next.js SSR, Rails, Django) | 静的HTML + API分離<br/>(React + REST API) |
    | **認証方式** | Cookie | Authorization Header |

??? question "セキュリティ: どちらが安全？"

    **結論: 適切に実装すれば両方とも安全です**

    ### 実際の通信

    **セッションベース:**
    ```http
    GET /api/users HTTP/1.1
    Cookie: session_id=abc123xyz789
    ```

    **トークンベース:**
    ```http
    GET /api/users HTTP/1.1
    Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
    ```

    **どちらもリクエストごとに認証情報を送信** - この点でのリスクは同じ

    ### セキュリティリスク比較

    | リスク | セッション | トークン(JWT) | 対策 |
    |-------|----------|--------------|-----|
    | **ネットワーク盗聴** | ⚠️ 同じ | ⚠️ 同じ | ✅ HTTPS必須 |
    | **盗まれた場合の無効化** | ✅ 即座に可能 | ❌ 困難 | 短い有効期限 |
    | **XSS攻撃** | ✅ HttpOnly Cookieで防御 | ❌ LocalStorageは危険 | HttpOnly Cookie or メモリ保存 |
    | **CSRF攻撃** | ⚠️ 対策必要 | ✅ Authorization Headerは安全 | SameSite属性 |

    ### トークンベース認証の注意点

    ❌ **危険な実装:**
    ```javascript
    // LocalStorageに保存 - XSSで盗まれる！
    localStorage.setItem('token', token);
    ```

    ✅ **安全な実装:**
    ```javascript
    // メモリ保存（React State等）
    const [token, setToken] = useState(null);

    // またはHttpOnly Cookieで保存
    Set-Cookie: token=xxx; HttpOnly; Secure; SameSite=Strict
    ```

    ### 安全な実装のチェックリスト

    どちらの方式でも以下を守る:

    - ✅ **HTTPS必須** - 通信を暗号化
    - ✅ **HttpOnly Cookie** - JavaScriptからアクセス不可
    - ✅ **短い有効期限** - 盗まれても影響を最小化（15分〜1時間）
    - ✅ **Secure属性** - HTTPS通信のみ
    - ✅ **SameSite属性** - CSRF対策

    **重要:** リクエストに認証情報が含まれること自体は問題ではありません。セッションIDもトークンも同様に含まれます。問題は**保存方法**と**有効期限管理**です。

---

## OAuth 2.0 と OpenID Connect

!!! tip "参考資料"
    **日本語の分かりやすい解説:**

    - 📘 **[一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)** - OAuth 2.0の基本を丁寧に解説（Authlete）
    - 📘 **[OAuth 2.0 全フローの図解と動画](https://qiita.com/TakahikoKawasaki/items/200951e5b5929f840a1f)** - 各フローを図解で理解
    - 📘 **[OpenID Connect 全フロー解説](https://qiita.com/TakahikoKawasaki/items/4ee9b55db9f7ef352b47)** - OIDCの詳細解説
    - 📘 **[OAuth 2.0 / OIDC 入門編](https://www.authlete.com/ja/developers/tutorial/)** - Authlete公式チュートリアル

    **公式仕様（英語）:**

    - 📖 [RFC 6749 - OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
    - 📖 [RFC 7636 - PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
    - 📖 [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)

??? note "OAuth 2.0 と OpenID Connect の基本"

    **OAuth 2.0:** パスワードを共有せずに、他のアプリに限定的なアクセス権を与える**認可プロトコル**

    **OpenID Connect (OIDC):** OAuth 2.0に**認証機能**を追加したプロトコル（ユーザー情報の取得が可能）

    ??? example "具体例で理解する"

        **OAuth 2.0の例:**
        ```
        ✅ 写真管理アプリがGoogleフォトにアクセス
        → Googleのパスワードは渡さない
        → 「写真の閲覧のみ」の権限だけ付与
        → いつでも権限を取り消せる
        ```

        **OpenID Connectの例:**
        ```
        ✅ 「Googleでログイン」ボタン
        → ユーザー情報（名前、メールアドレス）を取得
        → パスワードは一切共有しない
        ```

??? abstract "Webアプリケーション向け認証フロー比較"

    | フロー | 推奨度 | 説明 | 適用 |
    |--------|--------|------|------|
    | **Authorization Code + PKCE** | 🟢 **必須** | 最も安全な認証フロー。<br/>認可コード横取り攻撃を防止。 | Web、SPA、Mobile |
    | **Client Credentials** | 🟢 推奨 | サーバー間通信用。<br/>ユーザー不在のM2M認証。 | マイクロサービス、<br/>バッチ処理 |
    | **Implicit** | 🔴 **禁止** | トークンがURLに露出。<br/>セキュリティリスクが高い。 | ❌ 使用しない |
    | **Password Credentials** | 🔴 **禁止** | パスワードをアプリに渡す。<br/>OAuth 2.0の思想に反する。 | ❌ 使用しない |

    **2025年の推奨:**

    - ✅ **Web/SPA/Mobile**: Authorization Code + PKCE フローを使用
    - ✅ **サーバー間通信**: Client Credentials フローを使用
    - ❌ **Implicit / Password Credentials**: 使用禁止

??? note "OAuth 2.0/OIDC の登場人物"

    | 役割 | 説明 | 例 |
    |------|------|-----|
    | **ユーザー** | リソースの所有者 | あなた |
    | **クライアント** | ユーザーが使用するアプリ | Next.js フロントエンド |
    | **認可サーバー/IdP** | 認証・トークン発行 | AWS Cognito, Auth0 |
    | **リソースサーバー** | 保護されたデータ提供 | Spring Boot API |

??? note "トークンの種類"

    ??? info "アクセストークン"

        **用途:** API アクセスの認可

        ```http
        Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
        ```

        - **有効期限:** 短命(15分〜1時間)
        - **内容:** スコープ(権限)を含む

    ??? info "IDトークン (OIDC)"

        **用途:** ユーザー情報の取得

        ```json
        {
          "sub": "user-id-12345",
          "email": "user@example.com",
          "name": "John Doe",
          "exp": 1616239022
        }
        ```

        - **有効期限:** 短命(15分〜1時間)
        - **内容:** ユーザー情報

    ??? info "リフレッシュトークン"

        **用途:** アクセストークンの再発行

        - **有効期限:** 長命(数日〜数ヶ月)
        - **保管:** 厳重に管理(HttpOnly Cookie推奨)

??? danger "OAuth 2.0/OIDC の重要なセキュリティパラメータ"

    | パラメータ | 目的 | 必須度 |
    |----------|------|--------|
    | **PKCE** | 認可コード横取り防止 | 🔴 必須 |
    | **state** | CSRF 対策 | 🔴 必須 |
    | **nonce** | IDトークンのリプレイ攻撃防止 | 🟡 推奨 |

    ### 具体的な実装方法

    ??? example "1. PKCE（ピクシー）の実装"

        **目的:** 認可コードが横取りされても、トークンを取得できないようにする

        **仕組み:**

        ```mermaid
        sequenceDiagram
            participant Client as クライアント
            participant AuthServer as 認可サーバー

            Note over Client: 1. ランダム値生成
            Client->>Client: code_verifier = ランダム文字列(43-128文字)
            Client->>Client: code_challenge = SHA256(code_verifier)

            Note over Client,AuthServer: 2. 認可リクエスト
            Client->>AuthServer: code_challenge を送信
            AuthServer->>AuthServer: code_challenge を保存

            Note over Client,AuthServer: 3. トークンリクエスト
            Client->>AuthServer: code + code_verifier を送信
            AuthServer->>AuthServer: SHA256(code_verifier) == code_challenge?
            alt 一致
                AuthServer->>Client: ✅ トークン発行
            else 不一致
                AuthServer->>Client: ❌ エラー
            end
        ```

        **実装例（JavaScript）:**

        ```javascript
        // 1. code_verifier を生成（ランダム文字列）
        function generateCodeVerifier() {
          const array = new Uint8Array(32);
          crypto.getRandomValues(array);
          return base64UrlEncode(array);
        }

        // 2. code_challenge を生成（SHA256ハッシュ）
        async function generateCodeChallenge(verifier) {
          const encoder = new TextEncoder();
          const data = encoder.encode(verifier);
          const hash = await crypto.subtle.digest('SHA-256', data);
          return base64UrlEncode(hash);
        }

        // 3. 認可リクエスト
        const codeVerifier = generateCodeVerifier();
        const codeChallenge = await generateCodeChallenge(codeVerifier);

        // セッションに保存（後で使用）
        sessionStorage.setItem('code_verifier', codeVerifier);

        // 認可サーバーへリダイレクト
        window.location.href = `https://auth-server.com/authorize?
          client_id=xxx&
          redirect_uri=https://myapp.com/callback&
          response_type=code&
          code_challenge=${codeChallenge}&
          code_challenge_method=S256`;

        // 4. コールバック後、トークンリクエスト
        const code = new URLSearchParams(window.location.search).get('code');
        const verifier = sessionStorage.getItem('code_verifier');

        fetch('https://auth-server.com/token', {
          method: 'POST',
          body: new URLSearchParams({
            grant_type: 'authorization_code',
            code: code,
            redirect_uri: 'https://myapp.com/callback',
            client_id: 'xxx',
            code_verifier: verifier  // ← ここで使用
          })
        });
        ```

        **攻撃者が認可コードを盗んでも:**

        - `code_verifier` を知らない
        - → `code_challenge` の検証に失敗
        - → トークンを取得できない ✅

    ??? example "2. state パラメータの実装"

        **目的:** CSRF攻撃を防ぐ（悪意のあるサイトからの認証リクエストを防止）

        **仕組み:**

        ```mermaid
        sequenceDiagram
            participant Client as クライアント
            participant AuthServer as 認可サーバー

            Note over Client: 1. ランダム値生成
            Client->>Client: state = ランダム文字列
            Client->>Client: セッションに保存

            Note over Client,AuthServer: 2. 認可リクエスト
            Client->>AuthServer: state を送信

            Note over Client,AuthServer: 3. コールバック
            AuthServer->>Client: code + state を返却
            Client->>Client: セッションのstateと一致？
            alt 一致
                Client->>Client: ✅ 処理続行
            else 不一致
                Client->>Client: ❌ CSRF攻撃の可能性
            end
        ```

        **実装例（JavaScript）:**

        ```javascript
        // 1. state を生成
        function generateState() {
          const array = new Uint8Array(32);
          crypto.getRandomValues(array);
          return base64UrlEncode(array);
        }

        // 2. 認可リクエスト前
        const state = generateState();
        sessionStorage.setItem('oauth_state', state);

        window.location.href = `https://auth-server.com/authorize?
          client_id=xxx&
          redirect_uri=https://myapp.com/callback&
          response_type=code&
          state=${state}`;  // ← state を送信

        // 3. コールバック処理
        const returnedState = new URLSearchParams(window.location.search).get('state');
        const savedState = sessionStorage.getItem('oauth_state');

        if (returnedState !== savedState) {
          // CSRF攻撃の可能性
          throw new Error('Invalid state parameter');
        }

        // state検証成功 → トークンリクエスト
        ```

        **CSRF攻撃のシナリオ:**

        1. 攻撃者が自分のアカウントで認可リクエストを開始
        2. 被害者に認可コードを含むURLをクリックさせる
        3. `state` がないと、被害者のセッションに攻撃者のアカウントが紐付けられる
        4. `state` があれば、セッションの値と一致しないため防げる ✅

    ??? example "3. nonce パラメータの実装（OIDC）"

        **目的:** IDトークンのリプレイ攻撃を防ぐ

        **仕組み:**

        ```mermaid
        sequenceDiagram
            participant Client as クライアント
            participant AuthServer as 認可サーバー

            Note over Client: 1. ランダム値生成
            Client->>Client: nonce = ランダム文字列
            Client->>Client: セッションに保存

            Note over Client,AuthServer: 2. 認可リクエスト
            Client->>AuthServer: nonce を送信

            Note over Client,AuthServer: 3. IDトークン取得
            AuthServer->>Client: IDトークン(nonce含む)
            Client->>Client: IDトークンのnonceと一致？
            alt 一致
                Client->>Client: ✅ トークン受理
            else 不一致
                Client->>Client: ❌ リプレイ攻撃の可能性
            end
        ```

        **実装例（JavaScript）:**

        ```javascript
        // 1. nonce を生成
        function generateNonce() {
          const array = new Uint8Array(32);
          crypto.getRandomValues(array);
          return base64UrlEncode(array);
        }

        // 2. 認可リクエスト
        const nonce = generateNonce();
        sessionStorage.setItem('oidc_nonce', nonce);

        window.location.href = `https://auth-server.com/authorize?
          client_id=xxx&
          response_type=code&
          scope=openid email profile&
          nonce=${nonce}`;  // ← nonce を送信

        // 3. IDトークン検証
        const idToken = jwt.decode(response.id_token);
        const savedNonce = sessionStorage.getItem('oidc_nonce');

        if (idToken.nonce !== savedNonce) {
          // リプレイ攻撃の可能性
          throw new Error('Invalid nonce');
        }
        ```

        **IDトークンの中身（nonce含む）:**

        ```json
        {
          "sub": "user-id-12345",
          "email": "user@example.com",
          "nonce": "abc123xyz789",  // ← セッションの値と一致を確認
          "iat": 1616239022,
          "exp": 1616242622
        }
        ```

    ### セキュリティパラメータの組み合わせ

    **最小限の実装（必須）:**

    ```javascript
    // PKCE + state は必ず実装
    const params = {
      client_id: 'xxx',
      redirect_uri: 'https://myapp.com/callback',
      response_type: 'code',
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      state: state
    };
    ```

    **推奨実装（OIDC使用時）:**

    ```javascript
    // PKCE + state + nonce
    const params = {
      client_id: 'xxx',
      redirect_uri: 'https://myapp.com/callback',
      response_type: 'code',
      scope: 'openid email profile',
      code_challenge: codeChallenge,
      code_challenge_method: 'S256',
      state: state,
      nonce: nonce  // OIDCならnonceも追加
    };
    ```

    ### ライブラリを使う（推奨）

    実際の実装では、認証ライブラリを使用することを強く推奨します：

    **Next.js:**
    ```javascript
    // NextAuth.js が自動的に PKCE + state を設定
    import CognitoProvider from "next-auth/providers/cognito";

    CognitoProvider({
      clientId: process.env.COGNITO_CLIENT_ID,
      clientSecret: process.env.COGNITO_CLIENT_SECRET,
      issuer: process.env.COGNITO_ISSUER,
      checks: ["pkce", "state"]  // 自動設定
    })
    ```

    **React SPA:**
    ```javascript
    // @auth0/auth0-react が自動的に処理
    import { Auth0Provider } from "@auth0/auth0-react";

    <Auth0Provider
      domain="your-domain.auth0.com"
      clientId="xxx"
      // PKCE + state + nonce を自動的に処理
      authorizationParams={{
        redirect_uri: window.location.origin
      }}
    >
    ```

    **重要:** 自前実装は複雑でミスが起きやすいため、信頼できるライブラリの使用を推奨します。

---

## セキュリティの基本

??? success "Cookie のセキュリティ設定"

    ```javascript
    Set-Cookie: session_id=xxx;
      HttpOnly;         // JavaScriptからアクセス不可(XSS対策)
      Secure;           // HTTPS通信のみ
      SameSite=Strict;  // CSRF対策
      Max-Age=86400     // 有効期限
    ```

??? warning "トークン保管場所"

    | 場所 | セキュリティ | 推奨 |
    |------|------------|-----|
    | **LocalStorage** | ❌ XSSで盗まれる | 禁止 |
    | **HttpOnly Cookie** | ✅ JavaScript不可 | 推奨 |
    | **メモリ(React State)** | ✅ XSS耐性高 | 推奨 |

---

## 構成パターン別の認証方式

??? success "パターン1: モノリシック構成（統合型）"

    **セッションベース認証が適している**

    **使用する認証フロー:** Authorization Code + PKCE

    **実際の認証フロー（外部IdP使用）:**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ サーバー(Next.js等)
        participant IdP as 🔐 認証プロバイダー<br/>(Cognito/Auth0)
        participant Redis as 💾 セッションストア

        Browser->>Server: 1. ログインボタン押下
        Server->>IdP: 2. 認証リダイレクト
        IdP->>Browser: 3. ログイン画面表示
        Browser->>IdP: 4. ID/パスワード入力
        IdP->>Server: 5. 認証コード返却
        Server->>IdP: 6. トークン取得
        Server->>Redis: 7. セッション保存
        Server->>Browser: 8. Cookie(session_id)発行
    ```

    **ポイント:**

    - パスワード管理は**IdPが担当**（自前で管理しない）
    - サーバーは認証後の**セッション管理のみ**を行う
    - ライブラリ例: `next-auth`、`devise`（Rails）、`django-allauth`

    **構成の特徴:**

    - 📦 **1つのサーバー** で全て処理（フロント+バック統合）
    - 🏗️ サーバーが **HTMLを生成** して返す（SSR）
    - 🔐 **Cookieベースのセッション管理**
    - ✅ **即座なセッション無効化が可能**

    **技術例:** Ruby on Rails、Django、Laravel、Next.js（SSRモード）、Spring Boot + Thymeleaf

    **適用例:** 社内システム、管理画面、ブログ、ECサイト（SSR）

??? success "パターン2: 分離型構成（SPA + API）"

    **トークンベース認証が適している**

    **使用する認証フロー:** Authorization Code + PKCE

    **実際の認証フロー（外部IdP使用）:**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ(SPA)
        participant IdP as 🔐 認証プロバイダー<br/>(Cognito/Auth0)
        participant API as 🔌 APIサーバー

        Browser->>IdP: 1. ログインリクエスト
        IdP->>Browser: 2. ID/パスワード入力
        IdP->>Browser: 3. トークン発行(JWT)
        Browser->>Browser: 4. トークンをメモリ保存
        Browser->>API: 5. API呼び出し + Token
        API->>IdP: 6. トークン検証
        API->>Browser: 7. データ返却
    ```

    **ポイント:**

    - パスワード管理は**IdPが担当**（自前で管理しない）
    - サーバーは**トークン検証のみ**（状態を持たない）
    - ライブラリ例: `@auth0/auth0-react`、`aws-amplify`、`@supabase/auth-helpers`

    **構成の特徴:**

    - 🔀 **2つのサーバー** で役割分担（静的配信 + API）
    - 📄 **静的ファイルサーバー**: HTML/CSS/JavaScriptを配信（CDN）
    - 🔌 **APIサーバー**: データ処理・DB操作（JSON返却）
    - 🎫 **トークンベースの認証**（Authorization Header）

    **技術例:** React + Vercel/Netlify、Vue.js + Cloudflare Pages、Angular、バックエンドAPI（Spring Boot、FastAPI、Node.js等）

    **適用例:** モダンWebアプリ、SaaS製品、ダッシュボード、モバイルアプリバックエンド

??? tip "パターン3: ハイブリッド構成（BFFパターン）"

    **セッションとトークンの両方を併用可能**

    **使用する認証フロー:** Authorization Code + PKCE

    **実際の認証フロー（外部IdP使用）:**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant BFF as ⚡ Next.js<br/>(BFF)
        participant IdP as 🔐 認証プロバイダー<br/>(Cognito/Auth0)
        participant API as 🔌 バックエンドAPI<br/>(Spring Boot等)
        participant DB as 🗄️ データベース

        Note over Browser,BFF: 初回アクセス（SSR）
        Browser->>BFF: 1. ページリクエスト
        BFF->>IdP: 2. 認証リダイレクト
        IdP->>Browser: 3. ログイン画面表示
        Browser->>IdP: 4. ID/パスワード入力
        IdP->>BFF: 5. 認証コード返却
        BFF->>IdP: 6. トークン取得
        BFF->>BFF: 7. セッション保存
        BFF->>API: 8. データ取得API呼び出し
        API->>DB: 9. DB検索
        DB->>API: 10. データ返却
        API->>BFF: 11. JSON返却
        BFF->>BFF: 12. HTMLを生成（SSR）
        BFF->>Browser: 13. 完成したHTML + Cookie返却

        Note over Browser,DB: 以降のページ遷移（SPA）
        Browser->>BFF: 14. API Route呼び出し
        BFF->>API: 15. バックエンドAPIへ中継
        API->>DB: 16. DB検索
        DB->>API: 17. データ返却
        API->>BFF: 18. JSON返却
        BFF->>Browser: 19. JSON返却
        Browser->>Browser: 20. JavaScriptで画面更新
    ```

    **ポイント:**

    - パスワード管理は**IdPが担当**（自前で管理しない）
    - BFF層で**セッション管理**と**バックエンドAPIへの中継**を行う
    - ライブラリ例: `next-auth`、`@supabase/auth-helpers-nextjs`

    **BFFパターンの特徴:**

    - **初回表示はSSR**: 高速・SEO対策
    - **以降はSPA動作**: 部分更新・高速遷移
    - **BFF層でセッション管理**: サーバーサイドの認証状態を管理
    - **バックエンドAPIとの通信**: トークンまたはAPI Routesで中継

    **構成の特徴:**

    - 🎯 **Next.jsがBFF層として機能**（認証・セッション管理）
    - 🔌 **バックエンドAPIは別サーバー**（マイクロサービス対応）
    - ⚡ **SSRとSPAの良いとこ取り**
    - 🔐 **BFFでセッション管理** + **バックエンドへのAPI呼び出し**

    **技術例:** Next.js + API Routes、Nuxt.js + Server Routes、バックエンド（Spring Boot、FastAPI、Go等）

    **推奨ライブラリ:**

    - `next-auth` / `NextAuth.js`: Next.js向けの認証ライブラリ
    - `@supabase/auth-helpers-nextjs`: Supabase認証をNext.jsで使用

    **適用例:** ECサイト、ポータルサイト、エンタープライズWebアプリ

??? warning "重要: 認証方式とパスワード管理は別の概念"
    **セッションベース認証 ≠ 自前でパスワード管理**

    セッションベースでも、実際の認証は外部の認証プロバイダー（AWS Cognito、Auth0等）に委譲するのが **現代の標準** です。

    - ❌ **非推奨**: 自前でパスワードをハッシュ化してDBに保存
    - ✅ **推奨**: 外部IdPで認証 → セッションIDをCookieで管理

---

## 用語集

??? abstract "用語一覧"

    | 用語 | 説明 |
    |------|------|
    | **スコープ** | アクセス権限の範囲(例: `read:profile`) |
    | **認可コード** | トークン取得のための一時的なコード |
    | **PKCE** | 認可コード横取り攻撃を防ぐ仕組み |
    | **CSRF** | クロスサイトリクエストフォージェリ |
    | **XSS** | クロスサイトスクリプティング |

---

## 次のステップ

基礎を理解したら、実装に進みましょう:

- **[実装ガイド](implementation.md)** - Next.js + Spring Boot + Cognito での具体的な実装方法

---

**最終更新:** 2025年10月
**対象読者:** 認証機能を実装する開発者(初級〜中級)
