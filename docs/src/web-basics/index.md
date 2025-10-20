# Webアプリケーションの基礎

> **🎯 このドキュメントの目的**
> Webアプリケーションの仕組みを、ブラウザとサーバーの通信から理解します。「サーバーサイド」「クライアントサイド」といった用語の意味と、アーキテクチャの違いを学べます。

---

## ブラウザとサーバーの基本的な通信

### Webページが表示される仕組み

??? example "1. アドレスバーにURLを入力"

    ブラウザのアドレスバーに `https://example.com` と入力してEnterを押すと、以下の流れでWebページが表示されます。

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ Webサーバー

        Browser->>Server: 1. HTTPリクエスト<br/>GET https://example.com
        Server->>Browser: 2. レスポンス<br/>HTML + CSS + JavaScript
        Browser->>Browser: 3. HTML解析・画面描画
        Browser->>Browser: 4. CSS適用（スタイル）
        Browser->>Browser: 5. JavaScript実行（動作）
    ```

    **サーバーが返すもの:**

    - **HTML**: ページの構造（見出し、段落、ボタン等）
    - **CSS**: ページの見た目（色、レイアウト、フォント等）
    - **JavaScript**: ページの動作（クリック時の処理、アニメーション等）

    **ブラウザがやること:**

    - HTMLを解析してページ構造を理解
    - CSSを適用してデザインを適用
    - JavaScriptを実行して動的な動作を追加

??? example "2. 検索ボタンを押す"

    Webページで検索ボタンを押すと、サーバーにリクエストが送られ、結果が返ってきます。

    ```mermaid
    sequenceDiagram
        participant User as 👤 ユーザー
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ Webサーバー

        User->>Browser: 1. 検索ワード入力<br/>「猫」
        User->>Browser: 2. 検索ボタン押下
        Browser->>Server: 3. HTTPリクエスト<br/>GET /search?q=猫
        Server->>Server: 4. データベース検索
        Server->>Browser: 5. レスポンス<br/>検索結果データ
        Browser->>Browser: 6. 画面に検索結果を表示
    ```

    **リクエストの種類:**

    | メソッド | 用途 | 例 |
    |---------|------|-----|
    | **GET** | データ取得 | 検索、ページ表示 |
    | **POST** | データ送信 | ログイン、投稿 |
    | **PUT** | データ更新 | プロフィール編集 |
    | **DELETE** | データ削除 | 投稿削除 |

---

## クライアントサイドとサーバーサイド

??? note "クライアントサイド（フロントエンド）"

    **ブラウザ上で実行される処理**

    ```mermaid
    graph LR
        Browser[🌐 ブラウザ<br/>クライアントサイド] -->|実行| HTML[📄 HTML<br/>構造]
        Browser -->|適用| CSS[🎨 CSS<br/>見た目]
        Browser -->|実行| JS[⚙️ JavaScript<br/>動作]
    ```

    **主な役割:**

    - ✅ ユーザーに画面を表示
    - ✅ ユーザーの操作を受け付ける
    - ✅ 動的な画面更新（ボタンクリック、アニメーション等）
    - ✅ サーバーへのリクエスト送信

    **実行場所:** ユーザーのPC・スマホ上のブラウザ

    **技術例:**

    - HTML, CSS, JavaScript
    - React, Vue.js, Angular（フレームワーク）

??? note "サーバーサイド（バックエンド）"

    **サーバー上で実行される処理**

    ```mermaid
    graph LR
        Server[🖥️ Webサーバー<br/>サーバーサイド] -->|処理| Logic[⚙️ ビジネスロジック<br/>検索・計算]
        Server -->|アクセス| DB[(🗄️ データベース<br/>データ保存)]
        Server -->|生成| Response[📦 レスポンス<br/>HTML/JSON]
    ```

    **主な役割:**

    - ✅ ビジネスロジック実行（検索、計算、判定等）
    - ✅ データベースアクセス
    - ✅ 認証・認可処理
    - ✅ HTMLやデータ（JSON）を生成してブラウザに返す

    **実行場所:** インターネット上のサーバー

    **技術例:**

    - Node.js, Python, Java, Ruby, PHP等
    - フレームワーク: Express, Django, Spring Boot, Rails等

---

## Webアプリケーションの構成パターン

### パターン1: モノリシック構成（統合型）

??? success "フロントとバックが同じサーバー"

    **HTMLを生成するサーバーとAPIサーバーが同一**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ 統合Webサーバー<br/>(フロント+バック)
        participant DB as 🗄️ データベース

        Browser->>Server: 1. ページリクエスト<br/>GET /search?q=猫
        Server->>DB: 2. データ検索
        DB->>Server: 3. 検索結果
        Server->>Server: 4. HTMLを生成<br/>(サーバーサイドレンダリング)
        Server->>Browser: 5. 完成したHTML返却
        Browser->>Browser: 6. 画面表示
    ```

    **特徴:**

    - 📦 **1つのサーバー** で全て処理
    - 🏗️ サーバーが **HTMLを生成** して返す（SSR: Server-Side Rendering）
    - 🔄 ページ遷移時に **全体を再読み込み**

    **呼び方:**

    - **モノリシックアーキテクチャ**
    - **サーバーサイドレンダリング（SSR）**
    - **従来型Webアプリケーション**

    **技術例:**

    **従来型（典型的なモノリシック）:**

    - **Ruby on Rails**
    - **Django**（Python）
    - **Laravel**（PHP）
    - **Spring Boot + Thymeleaf**（Java）
    - **ASP.NET MVC**（.NET）

    **モダンフレームワーク（SSRモード使用時）:**

    - **Next.js**（React）- SSRモード
    - **Nuxt.js**（Vue）- SSRモード
    - **Remix**（React）

    !!! note "モダンフレームワークのSSRモード"
        Next.js等のモダンフレームワークも、純粋なSSRモード（`getServerSideProps`のみ使用）で構築すれば、パターン1に該当します。ただし、実際にはAPI Routesやクライアントサイドの機能を併用することが多いため、 **パターン3（ハイブリッド）として使われることがほとんど** です。

    **メリット:**

    - ✅ 構成がシンプル
    - ✅ SEO対策が容易（HTMLが完成している）
    - ✅ 初回表示が速い

    **デメリット:**

    - ❌ ページ遷移時に全体再読み込み
    - ❌ サーバー負荷が高い（毎回HTML生成）

### パターン2: 分離型構成（SPA + API）

??? success "フロントとバックが別のサーバー"

    **静的HTMLを配信するサーバーとAPIサーバーが別**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ<br/>(SPA)
        participant CDN as 📦 静的ファイルサーバー<br/>(CDN/S3)
        participant API as 🔌 APIサーバー<br/>(バックエンド)
        participant DB as 🗄️ データベース

        Note over Browser,CDN: 初回アクセス
        Browser->>CDN: 1. ページリクエスト
        CDN->>Browser: 2. 静的ファイル<br/>(HTML+CSS+JS)
        Browser->>Browser: 3. JavaScriptアプリ起動

        Note over Browser,DB: データ取得
        Browser->>API: 4. APIリクエスト<br/>GET /api/search?q=猫
        API->>DB: 5. データ検索
        DB->>API: 6. 検索結果
        API->>Browser: 7. JSON返却
        Browser->>Browser: 8. JavaScriptで画面更新
    ```

    **特徴:**

    - 🔀 **2つのサーバー** で役割分担
    - 📄 **静的ファイルサーバー**: HTML/CSS/JavaScriptを配信（CDN）
    - 🔌 **APIサーバー**: データ処理・DB操作（JSON返却）
    - ⚡ ページ遷移時も **部分的な更新のみ**（高速）

    **呼び方:**

    - **SPA（Single Page Application）**
    - **フロントエンド・バックエンド分離**
    - **API駆動アーキテクチャ**
    - **JAMstack**（静的サイト + API）

    **技術例:**

    - **フロントエンド（静的配信）:**
        - **React** + Vercel/Netlify
        - **Vue.js** + Cloudflare Pages
        - **Angular** + AWS S3 + CloudFront

    - **バックエンド（API）:**
        - **Node.js** (Express, Fastify)
        - **Python** (FastAPI, Django REST Framework)
        - **Java** (Spring Boot)
        - **Go** (Gin, Echo)

    **メリット:**

    - ✅ フロントとバックを独立開発可能
    - ✅ ページ遷移が高速（部分更新のみ）
    - ✅ 静的ファイルはCDNで高速配信
    - ✅ 複数クライアント対応（Web、Mobile、デスクトップ）

    **デメリット:**

    - ❌ 構成が複雑
    - ❌ 初回表示が遅い（JavaScriptの読み込み・実行が必要）
    - ❌ SEO対策に工夫が必要

### パターン3: ハイブリッド構成（BFFパターン）

??? success "Next.js/Nuxt.jsをBFFとして活用"

    **SSRとSPAの良いとこ取り + バックエンドAPI分離**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant BFF as ⚡ Next.js<br/>(BFF)
        participant API as 🔌 バックエンドAPI<br/>(Spring Boot等)
        participant DB as 🗄️ データベース

        Note over Browser,BFF: 初回アクセス（SSR）
        Browser->>BFF: 1. ページリクエスト
        BFF->>API: 2. データ取得API呼び出し
        API->>DB: 3. DB検索
        DB->>API: 4. データ返却
        API->>BFF: 5. JSON返却
        BFF->>BFF: 6. HTMLを生成（SSR）
        BFF->>Browser: 7. 完成したHTML返却

        Note over Browser,DB: 以降のページ遷移（SPA）
        Browser->>BFF: 8. API Route呼び出し<br/>/api/search
        BFF->>API: 9. バックエンドAPIへ中継
        API->>DB: 10. DB検索
        DB->>API: 11. データ返却
        API->>BFF: 12. JSON返却
        BFF->>Browser: 13. JSON返却
        Browser->>Browser: 14. JavaScriptで画面更新
    ```

    **特徴:**

    - 🎯 **初回表示はSSR** （高速・SEO対策）
    - ⚡ **以降はSPA動作** （部分更新・高速遷移）
    - 🔀 **Next.jsがBFF（Backend For Frontend）として機能**
    - 🔌 **バックエンドAPIは別サーバー** （マイクロサービス対応）
    - 🔐 **BFFで認証・セッション管理**

    **BFF（Backend For Frontend）の役割:**

    - ✅ サーバーサイドレンダリング（SSR）
    - ✅ API Routesでバックエンドへの中継
    - ✅ 認証・セッション管理
    - ✅ バックエンドAPIの集約・変換
    - ✅ クライアント向けの最適化

    **技術例:**

    - **フロント+BFF:**
        - **Next.js**（React）+ API Routes
        - **Nuxt.js**（Vue）+ Server Routes
        - **SvelteKit**（Svelte）+ Server Endpoints

    - **バックエンドAPI:**
        - **Spring Boot**（Java）
        - **FastAPI**（Python）
        - **Express/Fastify**（Node.js）
        - **Go**（Gin, Echo）

    **メリット:**

    - ✅ 初回表示が速い（SSR）
    - ✅ ページ遷移が高速（SPA）
    - ✅ フロントとバックの分離（独立開発可能）
    - ✅ バックエンドAPIを複数マイクロサービス化可能
    - ✅ BFFでフロント特化の処理を実装

    **デメリット:**

    - ⚠️ 構成が複雑（3層構造）
    - ⚠️ BFFとバックエンドの役割分担が必要

---

## 構成パターンの比較表

??? abstract "どの構成を選ぶべきか"

    | 項目 | モノリシック<br/>(統合型) | 分離型<br/>(SPA + API) | ハイブリッド<br/>(BFFパターン) |
    |------|----------------------|-------------------|----------------------|
    | **サーバー数** | 1つ | 2つ（静的+API） | 2つ（BFF+API） |
    | **HTML生成** | サーバー | ブラウザ(JS) | サーバー（初回のみ） |
    | **初回表示** | ⚡ 速い | 🐌 遅い | ⚡ 速い（SSR） |
    | **ページ遷移** | 🐌 遅い（全体再読み込み） | ⚡ 速い（部分更新） | ⚡ 速い（SPA） |
    | **SEO** | ✅ 良好 | ⚠️ 要工夫 | ✅ 良好（SSR） |
    | **開発の複雑さ** | ✅ シンプル | ❌ 複雑 | ❌ 複雑（3層） |
    | **バックエンド分離** | ❌ 統合 | ✅ 完全分離 | ✅ 完全分離 |
    | **認証管理** | サーバー | ブラウザ+API | BFF |
    | **適用例** | 社内システム<br/>管理画面<br/>ブログ | モダンWebアプリ<br/>SaaS<br/>ダッシュボード | ECサイト<br/>ポータルサイト<br/>エンタープライズ |

---

## 用語まとめ

??? abstract "用語一覧"

    | 用語 | 説明 |
    |------|------|
    | **クライアントサイド** | ブラウザ上で実行される処理（HTML/CSS/JavaScript） |
    | **サーバーサイド** | サーバー上で実行される処理（ビジネスロジック、DB操作） |
    | **フロントエンド** | クライアントサイドと同義（ユーザーが見る部分） |
    | **バックエンド** | サーバーサイドと同義（データ処理・ロジック） |
    | **BFF** | Backend For Frontend（フロント専用のバックエンド層） |
    | **SSR** | Server-Side Rendering（サーバーでHTML生成） |
    | **CSR** | Client-Side Rendering（ブラウザでHTML生成） |
    | **SPA** | Single Page Application（1つのHTMLで動的に切り替え） |
    | **モノリシック** | フロントとバックが統合された構成 |
    | **マイクロサービス** | 機能ごとにサービスを分割した構成 |
    | **API** | アプリケーション間でデータをやり取りする仕組み |
    | **JSON** | データ交換フォーマット（APIの返却データ等） |

---

**最終更新:** 2025年10月
**対象読者:** Web開発初心者、「サーバーサイド」「クライアントサイド」の違いを理解したい方
