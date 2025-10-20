# 認証・認可の基礎知識

> **🎯 このドキュメントの目的**
> 認証・認可の基本概念から、OAuth 2.0/OpenID Connectの仕組みまで、体系的に理解できます。実装に必要な本質的な知識に絞り、簡潔に解説します。

## 📚 関連ドキュメント

- 次のステップ: **[実装ガイド](implementation.md)** - Next.js + Spring Boot + Cognito での実装
- 高度な内容: **[リファレンス](reference.md)** - 最新セキュリティ技術とアーキテクチャパターン

---

## 認証と認可

??? note "認証(Authentication) - 「誰か」を確認"

    **「あなたは誰ですか?」** という問いに答えるプロセス。

    - 例: ログイン、パスワード入力
    - HTTP ステータス: `401 Unauthorized`

??? note "認可(Authorization) - 「何ができるか」を確認"

    **「あなたは何ができますか?」** という問いに答えるプロセス。

    - 例: 管理者画面へのアクセス、ファイルの編集権限
    - HTTP ステータス: `403 Forbidden`

??? note "実際の流れ"

    ### 認証(Authentication)の流れ

    ```mermaid
    sequenceDiagram
        participant User as ユーザー
        participant App as アプリ

        User->>App: ログイン(ID/パスワード)
        alt 認証成功
            App->>User: ✅ 認証成功
        else 認証失敗
            App->>User: ❌ 401 Unauthorized<br/>(ID/パスワードが間違っている)
        end
    ```

    ### 認可(Authorization)の流れ

    ```mermaid
    sequenceDiagram
        participant User as ユーザー
        participant App as アプリ

        User->>App: リソースにアクセス
        App->>App: 権限チェック(認可)
        alt 権限あり
            App->>User: ✅ データ表示
        else 権限なし
            App->>User: ❌ 403 Forbidden<br/>(権限がない)
        end
    ```

---

## セッション vs トークン

??? note "セッションベース認証"

    ```mermaid
    graph LR
        Browser[ブラウザ] -->|Cookie: session_id| Server[サーバー]
        Server -->|セッション取得| Store[(セッションストア)]
    ```

    **特徴:**

    - サーバー側でセッション情報を保持(Stateful)
    - クライアントにはセッションIDのみ送信

    **メリット:**

    - ✅ サーバーで即座に無効化可能
    - ✅ セッション情報を完全管理

    **デメリット:**

    - ❌ ストレージが必要
    - ❌ 水平スケーリング時に共有が必要

    ### セッションの保存場所

    Webサーバーのセッション管理は、保存場所によって特性が異なります。

    ??? example "1. メモリ(デフォルト)"

        多くのフレームワークのデフォルト設定です。

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

    ??? example "2. Redis(本番推奨)"

        最も一般的な本番環境の構成です。

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

        - ✅ 高速
        - ✅ 水平スケーリング対応
        - ✅ 永続化可能

        **適用:** 本番環境、複数サーバー構成

    ??? example "3. その他のストレージ"

        - **データベース**: PostgreSQL、MongoDB等
        - **Memcached**: 高速キャッシュ
        - **DynamoDB**: AWS環境

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

??? note "トークンベース認証(JWT)"

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

??? note "比較表"

    | 項目 | セッション | トークン(JWT) |
    |------|----------|--------------|
    | 状態 | Stateful | Stateless |
    | 保存場所(サーバー) | 必要 | 不要 |
    | 無効化 | 即座に可能 | 困難 |
    | スケーリング | 共有必要 | 容易 |
    | 適用 | モノリス | SPA、マイクロサービス |

---

## OAuth 2.0 と OpenID Connect

??? note "OAuth 2.0 - 認可のプロトコル"

    **目的:** サードパーティアプリに限定的なアクセス権を与える

    **解決する問題:**

    ```
    ❌ 悪い例: パスワードを共有
    ユーザー → 写真アプリにGoogleのパスワードを渡す
             → アプリが全データにアクセス可能

    ✅ 良い例: OAuth 2.0
    ユーザー → Googleで認証
             → 「写真の閲覧のみ」の権限を発行
             → アプリは限定的なアクセスのみ
    ```

??? note "OpenID Connect(OIDC) - 認証のプロトコル"

    **目的:** OAuth 2.0 に認証機能を追加

    | 項目 | OAuth 2.0 | OpenID Connect |
    |------|----------|----------------|
    | 目的 | **認可** | **認証** |
    | 主なトークン | アクセストークン | **IDトークン** + アクセストークン |
    | 取得情報 | アクセス権 | **ユーザー情報** |

??? note "登場人物"

    ```mermaid
    graph TB
        User[👤 ユーザー]
        Client[💻 クライアント]
        AuthServer[🔐 認可サーバー/IdP]
        API[🗄️ リソースサーバー]

        User -->|1. ログイン| AuthServer
        AuthServer -->|2. トークン発行| Client
        Client -->|3. トークン付き| API
        API -->|4. データ| Client
    ```

    | 役割 | 説明 | 例 |
    |------|------|-----|
    | **ユーザー** | リソースの所有者 | あなた |
    | **クライアント** | ユーザーが使用するアプリ | Next.js フロントエンド |
    | **認可サーバー/IdP** | 認証・トークン発行 | AWS Cognito, Auth0 |
    | **リソースサーバー** | 保護されたデータ提供 | Spring Boot API |

---

## 認証フロー

??? note "Authorization Code + PKCE フロー(推奨)"

    **2025年の標準。すべての環境で推奨されるフロー。**

    ```mermaid
    sequenceDiagram
        participant Client
        participant AuthServer

        Client->>Client: 1. code_verifier生成
        Client->>Client: 2. code_challenge計算
        Client->>AuthServer: 3. 認可リクエスト(code_challenge)
        AuthServer->>Client: 4. 認可コード
        Client->>AuthServer: 5. 認可コード + code_verifier
        AuthServer->>AuthServer: 6. PKCE検証
        AuthServer->>Client: 7. トークン発行
    ```

    **PKCE の仕組み:**

    1. ランダムな`code_verifier`を生成
    2. `code_challenge = SHA256(code_verifier)`を計算
    3. 認可リクエストに`code_challenge`を含める
    4. トークン取得時に`code_verifier`を送信
    5. サーバーが検証: `SHA256(code_verifier) == code_challenge`

    **メリット:**

    - ✅ クライアントシークレット不要
    - ✅ 認可コード横取り攻撃を防止
    - ✅ SPA/Mobile でも安全

??? note "フロー選択チャート"

    ```mermaid
    graph TD
        Start[認証フローを選ぶ] --> Q1{ユーザーがいる?}
        Q1 -->|いいえ| M2M[Client Credentials<br/>サーバー間通信]
        Q1 -->|はい| Q2{デバイスは?}
        Q2 -->|Web/Mobile| PKCE[Authorization Code + PKCE]
        Q2 -->|TV/IoT| Device[Device Authorization Grant]
    ```

---

## トークンの種類

??? note "アクセストークン"

    **用途:** API アクセスの認可

    ```http
    Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
    ```

    - **有効期限:** 短命(15分〜1時間)
    - **内容:** スコープ(権限)を含む

??? note "IDトークン(OIDC)"

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

??? note "リフレッシュトークン"

    **用途:** アクセストークンの再発行

    - **有効期限:** 長命(数日〜数ヶ月)
    - **保管:** 厳重に管理(HttpOnly Cookie推奨)

---

## セキュリティの基本

??? note "Cookie のセキュリティ設定"

    ```javascript
    Set-Cookie: session_id=xxx;
      HttpOnly;         // JavaScriptからアクセス不可(XSS対策)
      Secure;           // HTTPS通信のみ
      SameSite=Strict;  // CSRF対策
      Max-Age=86400     // 有効期限
    ```

??? note "トークン保管場所"

    | 場所 | セキュリティ | 推奨 |
    |------|------------|-----|
    | **LocalStorage** | ❌ XSSで盗まれる | 禁止 |
    | **HttpOnly Cookie** | ✅ JavaScript不可 | 推奨 |
    | **メモリ(React State)** | ✅ XSS耐性高 | 推奨 |

??? note "重要なパラメータ"

    | パラメータ | 目的 | 必須度 |
    |----------|------|--------|
    | **PKCE** | 認可コード横取り防止 | 🔴 必須 |
    | **state** | CSRF 対策 | 🔴 必須 |
    | **nonce** | IDトークンのリプレイ攻撃防止 | 🟡 推奨 |

---

## 用語集

??? note "用語一覧"

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
- **[リファレンス](reference.md)** - 高度なセキュリティ技術とアーキテクチャパターン

---

**最終更新:** 2025年10月
**対象読者:** 認証機能を実装する開発者(初級〜中級)
