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

??? tip "なぜJWT（トークン）の無効化は困難なのか？"

    **結論: JWTは署名検証のみで認証するため、発行後のトークンを取り消せません**

    ### 仕組みの違い

    | 項目 | セッション | JWT |
    |------|----------|-----|
    | **検証方法** | サーバーがストアを確認 | サーバーが署名を確認 |
    | **無効化** | ストアから削除 | ❌ 削除する場所がない |
    | **ステートレス** | ❌ (Stateful) | ✅ (Stateless) |

    ### AWS Cognito の場合

    **Cognitoの`GlobalSignOut`API:**

    ```bash
    aws cognito-idp admin-user-global-sign-out \
      --user-pool-id us-east-1_XXXXXXXXX \
      --username user@example.com
    ```

    **実際に無効化されるもの:**

    - ✅ **リフレッシュトークン**: 即座に無効化（新しいアクセストークンを取得できなくなる）
    - ❌ **アクセストークン**: 有効期限まで使える（既に発行済みのトークンは無効化できない）
    - ❌ **IDトークン**: 有効期限まで使える（既に発行済みのトークンは無効化できない）

    **つまり:**

    ```mermaid
    sequenceDiagram
        participant User as ユーザー
        participant API as APIサーバー
        participant Cognito as Cognito

        Note over User,Cognito: ログイン時
        User->>Cognito: 認証
        Cognito->>User: アクセストークン（有効期限: 1時間）<br/>リフレッシュトークン

        Note over Cognito: 管理者がGlobalSignOut実行
        Cognito->>Cognito: リフレッシュトークンを無効化

        Note over User,API: GlobalSignOut後も
        User->>API: Bearer: アクセストークン（既に発行済み）
        API->>API: JWT署名検証 → ✅ 有効
        API->>User: レスポンス（有効期限まで使える！）

        Note over User,Cognito: アクセストークン期限切れ後
        User->>Cognito: リフレッシュトークンで更新
        Cognito->>User: ❌ エラー（リフレッシュトークンは無効化済み）
    ```

    ### 対策: トークン有効期限を短く設定

    Cognitoなどの認証プロバイダーでは、トークン有効期限を短く設定することで影響を最小化できます：

    - アクセストークン有効期限: 5〜15分
    - IDトークン有効期限: 5〜15分
    - リフレッシュトークン有効期限: 30日

    → **GlobalSignOut実行後、最大5〜15分で完全に無効化**

    ### まとめ

    | 要件 | 推奨方法 |
    |------|---------|
    | **一般的なWebアプリ** | 短い有効期限（5〜15分）+ リフレッシュトークン |
    | **即座の無効化が必須** | セッションベース認証 |
    | **Cognito使用** | アクセストークン有効期限を5〜15分に設定 |

    **JWTの無効化が困難な理由は、ステートレス設計という本質的な特性によるものです。リフレッシュトークンは無効化できますが、既に発行済みのアクセストークンは有効期限まで使えてしまいます。これを理解した上で、適切な有効期限設定で対応するのがベストプラクティスです。**

??? question "セキュリティ: どちらが安全？"

    **結論: 適切に実装すれば両方とも安全です**

    どちらもリクエストごとに認証情報を送信するため、この点でのリスクは同じです。

    ### セキュリティリスク比較

    | リスク | セッション | トークン(JWT) | 対策 |
    |-------|----------|--------------|-----|
    | **ネットワーク盗聴** | ⚠️ 同じ | ⚠️ 同じ | ✅ HTTPS必須 |
    | **盗まれた場合の無効化** | ✅ 即座に可能 | ❌ 困難 | 短い有効期限 |
    | **XSS攻撃** | ✅ HttpOnly Cookieで防御 | ❌ LocalStorageは危険 | HttpOnly Cookie or メモリ保存 |
    | **CSRF攻撃** | ⚠️ 対策必要 | ✅ Authorization Headerは安全 | SameSite属性 |

    ### トークンベース認証の注意点

    **危険な実装:** LocalStorageに保存 - XSSで盗まれる

    **安全な実装:**
    - メモリ保存（React Stateなど）
    - HttpOnly Cookieで保存

    ### 安全な実装のチェックリスト

    どちらの方式でも以下を守る:

    - ✅ **HTTPS必須** - 通信を暗号化
    - ✅ **HttpOnly Cookie** - JavaScriptからアクセス不可
    - ✅ **短い有効期限** - 盗まれても影響を最小化（15分〜1時間）
    - ✅ **Secure属性** - HTTPS通信のみ
    - ✅ **SameSite属性** - CSRF対策

    **重要:** リクエストに認証情報が含まれること自体は問題ではありません。セッションIDもトークンも同様に含まれます。問題は **保存方法** と **有効期限管理** です。

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

    **OAuth 2.0:** パスワードを共有せずに、他のアプリに限定的なアクセス権を与える **認可プロトコル**

    **OpenID Connect (OIDC):** OAuth 2.0に **認証機能** を追加したプロトコル（ユーザー情報の取得が可能）

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

        **仕組み:**

        1. クライアントがランダムな `code_verifier` を生成
        2. `code_verifier` をSHA256でハッシュ化して `code_challenge` を作成
        3. 認可リクエスト時に `code_challenge` を送信
        4. トークンリクエスト時に `code_verifier` を送信
        5. サーバーが `code_verifier` をハッシュ化して `code_challenge` と一致するか検証

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

        **仕組み:**

        1. クライアントがランダムな `state` を生成してセッションに保存
        2. 認可リクエスト時に `state` を送信
        3. コールバック時に返却された `state` がセッションの値と一致するか検証

        **CSRF攻撃のシナリオ:**

        1. 攻撃者が自分のアカウントで認可リクエストを開始
        2. 被害者に認可コードを含むURLをクリックさせる
        3. `state` がないと、被害者のセッションに攻撃者のアカウントが紐付けられる
        4. `state` があれば、セッションの値と一致しないため防げる ✅

    ??? example "3. nonce パラメータの実装（OIDC）"

        **目的:** IDトークンのリプレイ攻撃を防ぐ

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

        **仕組み:**

        1. クライアントがランダムな `nonce` を生成してセッションに保存
        2. 認可リクエスト時に `nonce` を送信
        3. IDトークン内の `nonce` がセッションの値と一致するか検証

    ### セキュリティパラメータの組み合わせ

    **最小限の実装（必須）:**

    - PKCE（`code_challenge` + `code_verifier`）
    - state（CSRF対策）

    **推奨実装（OIDC使用時）:**

    - PKCE（`code_challenge` + `code_verifier`）
    - state（CSRF対策）
    - nonce（IDトークンのリプレイ攻撃防止）

    ### ライブラリを使う（推奨）

    実際の実装では、認証ライブラリを使用することを強く推奨します。Auth.js (NextAuth.js)、Auth0 SDK、AWS Amplifyなどの認証ライブラリは、これらのセキュリティパラメータを自動的に処理します。

    **重要:** 自前実装は複雑でミスが起きやすいため、信頼できるライブラリの使用を推奨します。

---

## セキュリティの基本

??? success "Cookie のセキュリティ設定"

    Cookieには以下のセキュリティ属性を設定します：

    - **HttpOnly**: JavaScriptからアクセス不可（XSS対策）
    - **Secure**: HTTPS通信のみ
    - **SameSite=Strict/Lax**: CSRF対策
    - **Max-Age/Expires**: 有効期限

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

    - パスワード管理は **IdPが担当** （自前で管理しない）
    - サーバーは認証後の **セッション管理のみ** を行う
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

    - パスワード管理は **IdPが担当** （自前で管理しない）
    - サーバーは **トークン検証のみ** （状態を持たない）
    - ライブラリ例: `@auth0/auth0-react`、`aws-amplify`、`@supabase/auth-helpers`

    **構成の特徴:**

    - 🔀 **2つのサーバー** で役割分担（静的配信 + API）
    - 📄 **静的ファイルサーバー**: HTML/CSS/JavaScriptを配信（CDN）
    - 🔌 **APIサーバー**: データ処理・DB操作（JSON返却）
    - 🎫 **トークンベースの認証**（Authorization Header）

    **技術例:** React + Vercel/Netlify、Vue.js + Cloudflare Pages、Angular、バックエンドAPI（Spring Boot、FastAPI、Node.js等）

    **適用例:** モダンWebアプリ、SaaS製品、ダッシュボード、モバイルアプリバックエンド

??? tip "パターン3: ハイブリッド構成（BFFパターン）"

    **Auth.js (NextAuth.js) は2つのセッションストラテジーをサポート**

    Auth.jsは **JWT Session** と **Database Session** の2つのセッション戦略を提供します。

    !!! info "参考資料"
        📘 **[Auth.js - Session Strategies](https://authjs.dev/concepts/session-strategies)** - 公式ドキュメント

    **使用する認証フロー:** Authorization Code + PKCE

    ### 📌 パターン3-A: JWT Session（推奨）

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
        BFF->>IdP: 6. トークン取得（アクセストークン、IDトークン等）
        BFF->>BFF: 7. JWTを暗号化してCookieに保存<br/>（セッションストア不要）
        BFF->>API: 8. データ取得API呼び出し
        API->>DB: 9. DB検索
        DB->>API: 10. データ返却
        API->>BFF: 11. JSON返却
        BFF->>BFF: 12. HTMLを生成（SSR）
        BFF->>Browser: 13. 完成したHTML + 暗号化Cookie返却

        Note over Browser,DB: 以降のページ遷移（SPA）
        Browser->>BFF: 14. API Route呼び出し（暗号化Cookie送信）
        BFF->>BFF: 15. Cookie内のJWTを復号化・検証
        BFF->>API: 16. バックエンドAPIへ中継
        API->>DB: 17. DB検索
        DB->>API: 18. データ返却
        API->>BFF: 19. JSON返却
        BFF->>Browser: 20. JSON返却
        Browser->>Browser: 21. JavaScriptで画面更新
    ```

    **ポイント:**

    - パスワード管理は **IdPが担当** （自前で管理しない）
    - BFF層で **暗号化されたJWTをHttpOnly Cookieで管理** （データベース不要）
    - JWTは暗号化（JWE）され、秘密鍵で保護される
    - Auth.js (NextAuth.js) が自動的にJWTの暗号化・復号化・検証を実行
    - ブラウザには暗号化されたCookieのみ送信（セキュア）
    - **バックエンドAPIへの中継** 時にアクセストークンを使用可能

    **JWT Sessionの特徴:**

    - ✅ **ステートレス**: データベース不要、スケーリングが容易
    - ✅ **パフォーマンス**: データベースへの問い合わせ不要で高速
    - ✅ **コスト削減**: セッションストレージのインフラ不要
    - ⚠️ **即座の無効化が困難**: トークン有効期限まで有効（短い有効期限で対応）
    - ⚠️ **サイズ制限**: Cookie制限（約4KB）、Auth.jsはチャンキング対応

    ### 📌 パターン3-B: Database Session

    **実際の認証フロー（外部IdP + セッションDB使用）:**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant BFF as ⚡ Next.js<br/>(BFF)
        participant SessionDB as 💾 セッションDB<br/>(Redis/PostgreSQL)
        participant IdP as 🔐 認証プロバイダー<br/>(Cognito/Auth0)
        participant API as 🔌 バックエンドAPI<br/>(Spring Boot等)
        participant DB as 🗄️ データベース

        Note over Browser,BFF: 初回アクセス（SSR）
        Browser->>BFF: 1. ページリクエスト
        BFF->>IdP: 2. 認証リダイレクト
        IdP->>Browser: 3. ログイン画面表示
        Browser->>IdP: 4. ID/パスワード入力
        IdP->>BFF: 5. 認証コード返却
        BFF->>IdP: 6. トークン取得（アクセストークン、IDトークン等）
        BFF->>SessionDB: 7. セッション情報を保存<br/>（トークン、ユーザー情報）
        BFF->>BFF: 8. セッションIDを生成
        BFF->>API: 9. データ取得API呼び出し
        API->>DB: 10. DB検索
        DB->>API: 11. データ返却
        API->>BFF: 12. JSON返却
        BFF->>BFF: 13. HTMLを生成（SSR）
        BFF->>Browser: 14. 完成したHTML + Cookie(session_id)返却

        Note over Browser,DB: 以降のページ遷移（SPA）
        Browser->>BFF: 15. API Route呼び出し（Cookie: session_id送信）
        BFF->>SessionDB: 16. セッションIDでセッション情報を取得
        SessionDB->>BFF: 17. セッション情報返却
        BFF->>API: 18. バックエンドAPIへ中継
        API->>DB: 19. DB検索
        DB->>API: 20. データ返却
        API->>BFF: 21. JSON返却
        BFF->>Browser: 22. JSON返却
        Browser->>Browser: 23. JavaScriptで画面更新
    ```

    **ポイント:**

    - パスワード管理は **IdPが担当** （自前で管理しない）
    - BFF層で **セッションIDをHttpOnly Cookieで管理** 、セッション情報はDBに保存
    - CookieにはセッションIDのみ保存、ユーザーデータはサーバー側で管理
    - **即座にセッション無効化が可能** （DBから削除するだけ）
    - Auth.js (NextAuth.js) が自動的にセッションのCRUD操作を実行
    - **バックエンドAPIへの中継** 時にセッション情報からトークンを取得可能

    **Database Sessionの特徴:**

    - ✅ **即座の無効化**: "すべてのデバイスでログアウト"などが実装可能
    - ✅ **セキュリティ**: サーバー側でセッション完全管理
    - ✅ **同時ログイン制限**: データベースで管理可能
    - ⚠️ **パフォーマンス**: リクエストごとにデータベース問い合わせが必要
    - ⚠️ **インフラコスト**: データベースアダプター・サービスの管理が必要
    - ⚠️ **スケーリング**: データベース接続を考慮する必要あり

    ### 📊 2つのセッション戦略の比較

    | 項目 | JWT Session | Database Session |
    |------|-------------|------------------|
    | **セッション保存先** | HttpOnly Cookie（暗号化JWT/JWE） | データベース（セッションIDのみCookie） |
    | **データベース要件** | ✅ 不要（ステートレス） | ❌ 必須 |
    | **即座の無効化** | ❌ 困難（有効期限まで有効） | ✅ 可能（DBから削除） |
    | **パフォーマンス** | ✅ 高速（DB問い合わせ不要） | ⚠️ DB問い合わせが必要 |
    | **水平スケーリング** | ✅ 容易（ステートレス） | ⚠️ DB接続を考慮 |
    | **実装コスト** | ✅ 低い | ⚠️ 高い（DB設定・管理必要） |
    | **運用コスト** | ✅ 低い | ⚠️ 高い（DB管理必要） |
    | **適用例** | 一般的なWebアプリ | 即座の無効化が必須 |

    **推奨の選び方:**

    - ✅ **JWT Session**: ほとんどのケース（Auth.jsのデフォルト）
    - ⚠️ **Database Session**: 即座のセッション無効化・同時ログイン制限が必須の場合のみ

    **BFFパターンの共通特徴:**

    - **初回表示はSSR**: 高速・SEO対策
    - **以降はSPA動作**: 部分更新・高速遷移
    - **BFF層で認証管理**: サーバーサイドで認証状態を管理
    - **バックエンドAPIとの通信**: API Routesで中継、アクセストークンを転送

    **構成の特徴:**

    - 🎯 **Next.jsがBFF層として機能**（認証管理）
    - 🔌 **バックエンドAPIは別サーバー**（マイクロサービス対応）
    - ⚡ **SSRとSPAの良いとこ取り**
    - 🔐 **BFFで認証管理** + **バックエンドへのAPI呼び出し**

    **技術例:** Next.js + Auth.js (NextAuth.js)、Nuxt.js + Auth.js、バックエンド（Spring Boot、FastAPI、Go等）

    **推奨ライブラリ:**

    - ✅ **`next-auth` (Auth.js)**: Next.js向けの認証ライブラリ（JWT/Database両対応）
    - `@auth/prisma-adapter`: Prisma ORM を使ったDB連携
    - `@supabase/auth-helpers-nextjs`: Supabase認証をNext.jsで使用

    **適用例:** ECサイト、ポータルサイト、エンタープライズWebアプリ

    ??? success "なぜ JWT Session が推奨されるのか？"

        **Database Session方式:**

        ```mermaid
        graph LR
            Browser[ブラウザ] -->|Cookie: session_id| NextJS[Next.js]
            NextJS -->|セッション取得| DB[(セッションDB<br/>PostgreSQL/MySQL等)]
            NextJS -->|API呼び出し| Backend[バックエンドAPI]
        ```

        - ❌ データベース必須（PostgreSQL、MySQL等）
        - ❌ リクエストごとにDB問い合わせが必要（遅延）
        - ❌ インフラが複雑化
        - ❌ セッションDBの管理コストがかかる
        - ❌ スケーリング時にDB接続を考慮
        - ✅ 即座のセッション無効化が可能

        **JWT Session方式:**

        ```mermaid
        graph LR
            Browser[ブラウザ] -->|暗号化Cookie<br/>(JWE)| NextJS[Next.js]
            NextJS -->|JWT復号化<br/>検証| NextJS
            NextJS -->|API呼び出し| Backend[バックエンドAPI]
        ```

        - ✅ データベース不要（ステートレス）
        - ✅ シンプルなアーキテクチャ
        - ✅ 高速なレスポンス（DB問い合わせ不要）
        - ✅ スケーリングが容易（ステートレス）
        - ✅ 低コスト（DB不要、運用負荷が低い）
        - ✅ 自動セッション更新とタブ間同期に対応
        - ✅ Auth.js がJWE暗号化・復号化を自動処理
        - ✅ HttpOnly Cookieで安全にやり取り
        - ❌ 有効期限前の無効化が困難

        **セキュリティ面:**

        - JWTは ** Auth.js の秘密鍵でJWE暗号化 ** されているため、ブラウザで読めない
        - Cookie属性: `HttpOnly`, `Secure`, `SameSite=Lax/Strict` で保護
        - トークンの有効期限管理も自動

        **結論: BFFパターンでセッションストアを使う理由はほとんどない**

        Auth.js の ** JWT Session ** を使えば、セッションストアなしでセキュアな認証が実現できます。これが ** 現代のNext.js認証のベストプラクティス ** です。ただし、即座のセッション無効化や同時ログイン制限が必要な場合は Database Session を検討してください。

    ??? info "BFFパターンでCookieを使うメリット・デメリット"

        **フロントエンド ↔ BFF間の認証情報やり取り方法の比較**

        ### 🍪 Cookie方式（推奨）

        **メリット:**

        - ✅ **自動送信**: ブラウザが自動的にCookieを送信（コード不要）
        - ✅ **HttpOnly属性**: JavaScriptからアクセス不可 → XSS攻撃を防御
        - ✅ **Secure属性**: HTTPS通信のみで送信 → 盗聴防止
        - ✅ **SameSite属性**: CSRF攻撃を防御（SameSite=Strict/Lax）
        - ✅ **有効期限管理**: Cookieの`Expires`/`Max-Age`で自動削除
        - ✅ **フレームワーク標準**: Auth.js (NextAuth.js) がデフォルトで対応
        - ✅ **SSR対応**: サーバーサイドでも認証状態を確認可能

        **仕組み:**

        - Auth.js (NextAuth.js) が自動的にCookieを設定（HttpOnly、Secure、SameSite属性付き）
        - フロントエンド側は何もする必要なし（自動送信）
        - API Route側は自動的にCookieから認証情報を取得

        **デメリット:**

        - ⚠️ **ドメイン制限**: Same-Origin Policy、サブドメイン設定が必要
        - ⚠️ **サイズ制限**: 4KB制限（通常は十分）
        - ⚠️ **CSRF対策必要**: SameSite属性で対応可能

        ### 🔗 Authorization Header方式

        **メリット:**

        - ✅ **ドメイン制限なし**: CORSで異なるドメイン間でも送信可能
        - ✅ **サイズ制限緩和**: リクエストヘッダーに制限なし
        - ✅ **明示的制御**: 開発者が送信タイミングを完全制御

        **デメリット:**

        - ❌ **XSS脆弱**: LocalStorageに保存すると盗まれるリスク
        - ❌ **手動実装**: 全APIコールで手動設定が必要
        - ❌ **SSR困難**: サーバーサイドでトークン取得が困難
        - ❌ **自動送信なし**: fetch/axiosで毎回ヘッダー設定が必要

        ### 📊 比較表

        | 項目 | Cookie | Authorization Header |
        |------|--------|---------------------|
        | **セキュリティ** | ✅ 高（HttpOnly） | ❌ 低（XSS脆弱） |
        | **実装コスト** | ✅ 低（自動） | ❌ 高（手動） |
        | **SSR対応** | ✅ 簡単 | ❌ 困難 |
        | **ドメイン間通信** | ❌ 制限あり | ✅ 制限なし |
        | **フレームワーク対応** | ✅ 標準 | ⚠️ カスタム実装 |

        ### 🎯 BFFパターンでの推奨構成

        ```mermaid
        graph LR
            Browser[🌐 ブラウザ]
            BFF[⚡ Next.js BFF]
            API[🔌 バックエンドAPI]

            Browser -->|🍪 Cookie<br/>（暗号化JWT）| BFF
            BFF -->|🔑 Bearer Token<br/>（アクセストークン）| API
        ```

        **推奨パターン:**

        1. **ブラウザ ↔ BFF**: Cookieで暗号化JWT（Auth.js 標準）
        2. **BFF ↔ バックエンドAPI**: Authorization Headerでアクセストークン

        **理由:**

        - ブラウザ側は**セキュリティ最優先** → Cookie（HttpOnly）
        - BFF側は**柔軟性重視** → Header（異なるドメインOK）

        **BFF層の役割:**

        - Cookieから認証情報を取得（Auth.js が自動処理）
        - バックエンドAPIにアクセストークンを転送
        - APIレスポンスをフロントエンドに返却

        **結論: BFFパターンではCookieが圧倒的に推奨**

        Auth.js (NextAuth.js) の標準実装に従い、フロントエンド・BFF間はCookieを使用するのがベストプラクティスです。セキュリティ、実装コスト、SSR対応すべての面で優れています。

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
