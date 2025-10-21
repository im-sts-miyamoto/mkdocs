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

## Webアーキテクチャの進化

??? info "時代背景と技術の変遷"

    Webアプリケーションのアーキテクチャは、技術的な課題と時代のニーズに応じて進化してきました。

    ??? success "第1世代: モノリシック全盛期（2000年代前半〜）"

        **時代背景:**

        - 💻 Web 2.0の到来（ブログ、SNSの普及）
        - 🚀 スタートアップブーム（素早くサービスを立ち上げる必要）
        - 👨‍💻 小規模チームでの開発が主流

        **主流の構成:**

        ```
        ブラウザ ←→ モノリシックサーバー（Rails/Django/PHP） ←→ DB
        ```

        **採用された理由:**

        - ✅ **開発速度が速い**: 1つのフレームワークで全て完結
        - ✅ **学習コストが低い**: サーバーサイド言語のみで開発可能
        - ✅ **デプロイが簡単**: 1つのアプリをデプロイするだけ
        - ✅ **SEO対策が容易**: サーバーが完成したHTMLを返す

        **代表的な技術:**

        - **Ruby on Rails** (2004年) - Twitter, GitHub, Airbnb初期
        - **Django** (2005年) - Instagram, Pinterest初期
        - **Laravel** (2011年)

        **当時の課題:**

        - ❌ JavaScriptはあくまで「補助的な機能」
        - ❌ リッチなUIは難しい（jQuery全盛期）
        - ❌ ページ遷移時に全体リロード（UX低下）

    ??? success "第2世代: SPA革命（2010年代前半〜）"

        **時代背景:**

        - 📱 スマートフォンの普及（iPhone 2007年、Android 2008年）
        - ⚡ ユーザーが「アプリのような体験」を期待
        - 🌐 JavaScriptエンジンの高速化（Chrome V8登場 2008年）
        - 📦 npm登場（2010年）でJavaScriptエコシステム拡大

        **主流の構成:**

        ```
        ブラウザ（SPA: React/Angular） ←→ RESTful API ←→ DB
                  ↑
             静的ファイルはCDN
        ```

        **SPAが選ばれた理由:**

        - ✅ **ネイティブアプリのようなUX**: ページ遷移が高速・滑らか
        - ✅ **フロント・バック分離**: 並行開発可能
        - ✅ **モバイルアプリとAPI共有**: iOSアプリもAndroidアプリも同じAPIを使用
        - ✅ **CDNで高速配信**: 静的ファイルは世界中のCDNから配信
        - ✅ **フロントエンド専門エンジニア**: 役割分担が明確化

        **代表的な技術:**

        - **Backbone.js** (2010年) - 初期のSPAフレームワーク
        - **AngularJS** (2010年) - Google製、エンタープライズで人気
        - **React** (2013年) - Facebook製、現在の標準
        - **Vue.js** (2014年) - 学習が容易、日本で人気

        **実例:**

        - **Gmail** (2004年) - AJAX技術でSPA的な体験を先駆的に実現
        - **Facebook** (2013年〜) - React導入でSPA化
        - **Netflix** (2015年) - React採用

        **新たな課題:**

        - ❌ **初回表示が遅い**: JavaScriptのダウンロード・実行待ち
        - ❌ **SEO対策が困難**: 検索エンジンがJavaScriptを実行できない（当時）
        - ❌ **状態管理の複雑化**: Redux等の学習コスト
        - ❌ **開発の複雑化**: ビルドツール、トランスパイラ等が必要

    ??? success "第3世代: ハイブリッド時代（2016年〜現在）"

        **時代背景:**

        - 🔍 **SEOの重要性再認識**: Googleがビジネスの生命線
        - ⚡ **Core Web Vitals**: Googleがページ速度を検索順位に影響
        - 🌍 **グローバル展開**: 低速回線地域への対応
        - 🏢 **大規模化**: モノリシックでは限界、マイクロサービス化

        **主流の構成（BFFパターン）:**

        ```
        ブラウザ ←→ Next.js/Nuxt.js（BFF） ←→ バックエンドAPI（マイクロサービス） ←→ DB
                  初回SSR + 以降SPA               Spring Boot/FastAPI等
        ```

        **Next.js/Nuxt.jsが選ばれた理由:**

        - ✅ **初回SSR + 以降SPA**: SPAの良さを保ちつつSEO対策
        - ✅ **フロント・バック分離維持**: BFFでAPI中継
        - ✅ **開発者体験（DX）向上**: ファイルベースルーティング、自動最適化
        - ✅ **段階的な移行が可能**: ページ単位でSSR/CSR/SSGを選択
        - ✅ **エッジコンピューティング**: Vercel/Netlifyで世界中に配信

        **代表的な技術:**

        - **Next.js** (2016年) - React製、Vercel社
        - **Nuxt.js** (2016年) - Vue製
        - **Remix** (2020年) - React製、Web標準重視
        - **SvelteKit** (2020年) - Svelte製

        **実例:**

        - **TikTok** - Next.js採用
        - **Notion** - Next.js採用
        - **Hulu** - Next.js採用
        - **楽天市場** - Nuxt.js採用

        **現在のトレンド（2024年〜）:**

        - 🔥 **React Server Components**: Next.js App Router
        - ⚡ **Streaming SSR**: HTMLを段階的に送信
        - 🎯 **Islands Architecture**: Astro等、必要な部分だけJS
        - 🚀 **エッジファースト**: Cloudflare Workers, Deno Deploy

??? question "なぜ今も複数のアーキテクチャが共存するのか"

    **答え: プロジェクトの性質によって最適解が異なる**

    ### **モノリシックが今も選ばれる理由:**

    - ✅ スタートアップ初期（MVP開発）
    - ✅ 社内システム（SEO不要、開発速度重視）
    - ✅ 小規模チーム（フロント専門家がいない）
    - ✅ 既存システムの保守

    **事例:** Basecamp（Ruby on Rails）、GitHub（Ruby on Rails）

    ---

    ### **SPA + APIが選ばれる理由:**

    - ✅ リッチなUI/UX重視（管理画面、ダッシュボード）
    - ✅ モバイルアプリも同時開発
    - ✅ フロント・バックを別チームで開発
    - ✅ SEOが不要（ログイン後の画面）

    **事例:** Figma, Slack, Trello

    ---

    ### **Next.js（BFF）が選ばれる理由:**

    - ✅ ECサイト（SEO + リッチUI）
    - ✅ メディアサイト（SEO + 高速表示）
    - ✅ グローバル展開（エッジ配信）
    - ✅ 大規模アプリ（マイクロサービス化）

    **事例:** Nike.com, Twitch, Netflix（一部）

    ---

    ### **選択の判断基準:**

    | 要件 | おすすめ構成 |
    |------|------------|
    | **開発速度最優先** | モノリシック（Rails/Django） |
    | **SEO最重要** | モノリシック or Next.js（SSR） |
    | **リッチなUI** | SPA + API or Next.js |
    | **モバイルアプリ連携** | SPA + API or Next.js（BFF） |
    | **グローバル展開** | Next.js（エッジ配信） |
    | **大規模・複雑** | Next.js + マイクロサービス |

---

## レンダリング方式（HTML生成のタイミング）

Webアプリケーションでは、HTMLをどこで・いつ生成するかによって、いくつかの方式があります。

??? note "SSR（Server-Side Rendering）"

    ### サーバーでHTMLを生成

    **サーバーがリクエストごとにHTMLを生成してブラウザに返す**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ Webサーバー
        participant DB as 🗄️ データベース

        Browser->>Server: 1. ページリクエスト
        Server->>DB: 2. データ取得
        DB->>Server: 3. データ返却
        Server->>Server: 4. HTMLを生成
        Server->>Browser: 5. 完成したHTML返却
        Browser->>Browser: 6. 画面表示
    ```

    **特徴:**

    - 🏗️ **サーバーがHTMLを生成**
    - ⚡ **初回表示が速い**（HTMLが既に完成している）
    - ✅ **SEO対策が容易**（検索エンジンがHTMLを読み取れる）
    - 🔄 **リクエストごとに最新データ**

    **技術例:**

    - **従来型:** Ruby on Rails, Django, Laravel, Spring Boot + Thymeleaf
    - **モダン:** Next.js (`getServerSideProps`), Nuxt.js (`asyncData`), Remix

    **メリット:**

    - ✅ 初回表示が高速
    - ✅ SEO対策が容易
    - ✅ JavaScriptが無効でも動作

    **デメリット:**

    - ❌ サーバー負荷が高い（毎回HTML生成）
    - ❌ ページ遷移時に画面全体を再読み込み（従来型の場合）

??? note "CSR（Client-Side Rendering）"

    ### ブラウザでHTMLを生成

    **ブラウザがJavaScriptを使ってHTMLを生成・更新**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant CDN as 📦 静的サーバー
        participant API as 🔌 APIサーバー
        participant DB as 🗄️ データベース

        Browser->>CDN: 1. ページリクエスト
        CDN->>Browser: 2. 空のHTML + JavaScript
        Browser->>Browser: 3. JavaScriptアプリ起動
        Browser->>API: 4. データ取得API呼び出し
        API->>DB: 5. データ取得
        DB->>API: 6. データ返却
        API->>Browser: 7. JSON返却
        Browser->>Browser: 8. JavaScriptでHTML生成・表示
    ```

    **特徴:**

    - ⚙️ **ブラウザがJavaScriptでHTMLを生成**
    - 🐌 **初回表示が遅い**（JavaScript読み込み・実行が必要）
    - ⚡ **ページ遷移が高速**（部分更新のみ）
    - 🔄 **動的でインタラクティブ**

    **技術例:**

    - **React** 単体（Create React App, Vite）
    - **Vue.js** 単体
    - **Angular**

    **メリット:**

    - ✅ ページ遷移が高速（部分更新）
    - ✅ リッチなUI・UX
    - ✅ サーバー負荷が低い（静的ファイル配信のみ）

    **デメリット:**

    - ❌ 初回表示が遅い
    - ❌ SEO対策に工夫が必要
    - ❌ JavaScriptが必須

??? abstract "その他のレンダリング方式（SSG / ISR）"

    **SSG（Static Site Generation）: ビルド時に生成**

    - ビルド時に全ページのHTMLを事前生成
    - 変更がない限り同じHTMLを配信
    - **用途:** ブログ、ドキュメント、ランディングページ
    - **技術例:** Next.js (`getStaticProps`), Gatsby, Hugo

    **ISR（Incremental Static Regeneration）: 段階的再生成**

    - SSGとSSRの中間
    - 一定期間キャッシュし、期限切れ時に再生成
    - **用途:** 頻繁に更新されるコンテンツ（ECサイト等）
    - **技術例:** Next.js (`revalidate`), Nuxt.js (Hybrid Rendering)

---

## サーバー構成（アーキテクチャパターン）

Webアプリケーションのサーバー構成には、主に3つのパターンがあります。

??? success "パターン1: モノリシック構成"

    ### フロントとバックが同じサーバー（1サーバー）

    **フロントエンド（HTML生成）とバックエンド（ビジネスロジック・DB）が統合**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant Server as 🖥️ 統合Webサーバー<br/>(フロント+バック)
        participant DB as 🗄️ データベース

        Browser->>Server: 1. ページリクエスト
        Server->>DB: 2. データ検索
        DB->>Server: 3. 検索結果
        Server->>Server: 4. HTML生成
        Server->>Browser: 5. 完成したHTML返却
        Browser->>Browser: 6. 画面表示
    ```

    **特徴:**

    - 📦 **1つのサーバー** で全て処理
    - 🏗️ **レンダリング方式は自由**（SSR, CSR, SSG等を選択可能）
    - 🎯 **構成がシンプル**

    **呼び方:**

    - **モノリシックアーキテクチャ**
    - **統合型Webアプリケーション**
    - **従来型Webアプリケーション**

    **技術例:**

    - **Ruby on Rails**
    - **Django**（Python）
    - **Laravel**（PHP）
    - **Spring Boot + Thymeleaf**（Java）
    - **ASP.NET MVC**（.NET）
    - **Next.js**（単体使用・外部APIなし）

    !!! note "Next.js/Nuxt.jsの場合"
        Next.js等も、API RoutesでDB操作まで行い、外部のバックエンドAPIを使わない場合はパターン1です。レンダリング方式（SSR/CSR/SSG）はページ単位で選択できます。

    **メリット:**

    - ✅ 構成がシンプル
    - ✅ 開発・デプロイが容易
    - ✅ 小〜中規模アプリに最適

    **デメリット:**

    - ❌ フロント・バックが密結合
    - ❌ スケーリングが困難（全体を一緒にスケール）
    - ❌ モバイルアプリ等との連携が困難

??? success "パターン2: フロント・バックエンド分離構成"

    **フロントエンド（静的配信）とバックエンド（API）を完全分離**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant CDN as 📦 静的ファイルサーバー<br/>(CDN/S3)
        participant API as 🔌 APIサーバー<br/>(バックエンド)
        participant DB as 🗄️ データベース

        Note over Browser,CDN: 初回アクセス
        Browser->>CDN: 1. ページリクエスト
        CDN->>Browser: 2. 静的ファイル<br/>(HTML+CSS+JS)
        Browser->>Browser: 3. アプリ起動

        Note over Browser,DB: データ取得
        Browser->>API: 4. APIリクエスト
        API->>DB: 5. データ検索
        DB->>API: 6. 検索結果
        API->>Browser: 7. JSON返却
        Browser->>Browser: 8. 画面更新
    ```

    **特徴:**

    - 🔀 **2つのサーバー** で役割分担
    - 📄 **静的ファイルサーバー**: HTML/CSS/JavaScriptを配信（CDN）
    - 🔌 **APIサーバー**: データ処理・DB操作（JSON返却）
    - 🎨 **レンダリング方式は通常CSR**（SPAが一般的）

    **呼び方:**

    - **フロントエンド・バックエンド分離**
    - **API駆動アーキテクチャ**
    - **JAMstack**（静的サイト + API）

    **技術例:**

    - **フロント:** React, Vue.js, Angular（SPAフレームワーク）
    - **バック:** Spring Boot, FastAPI, Express.js, Django REST Framework

    **メリット:**

    - ✅ フロント・バックを独立開発
    - ✅ モバイルアプリでも同じAPIを利用
    - ✅ 静的ファイルはCDN配信で高速

    **デメリット:**

    - ❌ 初回表示が遅い（CSRのため）
    - ❌ SEO対策が難しい
    - ❌ 構成が複雑化

??? success "パターン3: BFF（Backends For Frontends）構成"

    **BFF（Backend For Frontend）を中間層として配置**

    BFF構成には **2つの実装方式** があります：

    ---

    #### **3-A: SSRハイブリッド型（Next.js等）**

    **Next.js/Nuxt.jsがBFF + SSR/SPAハイブリッド**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant BFF as ⚡ Next.js<br/>(BFF)
        participant API as 🔌 バックエンドAPI
        participant DB as 🗄️ DB

        Note over Browser,BFF: 初回（SSR）
        Browser->>BFF: 1. リクエスト
        BFF->>API: 2. API呼び出し
        API->>DB: 3. DB検索
        DB->>API: 4. 返却
        API->>BFF: 5. JSON
        BFF->>Browser: 6. HTML返却

        Note over Browser,DB: 以降（SPA）
        Browser->>BFF: 7. API Route
        BFF->>API: 8. 中継
        API->>BFF: 9. JSON
        BFF->>Browser: 10. JSON
    ```

    **特徴:**

    - 🎯 **初回SSR + 以降SPA**
    - 🔀 **BFFが認証・API中継・SSRを担当**
    - 🔌 **バックエンドAPIは別サーバー**

    **技術例:**

    - **Next.js** + Spring Boot
    - **Nuxt.js** + FastAPI
    - **SvelteKit** + Go API

    ---

    #### **3-B: 完全SPA + BFF型**

    **フロント（静的SPA） + BFF（API中継） + バックエンドAPI**

    ```mermaid
    sequenceDiagram
        participant Browser as 🌐 ブラウザ
        participant CDN as 📦 CDN
        participant BFF as 🔀 BFF
        participant API as 🔌 API
        participant DB as 🗄️ DB

        Browser->>CDN: 1. リクエスト
        CDN->>Browser: 2. 静的ファイル
        Browser->>Browser: 3. SPA起動
        Browser->>BFF: 4. API呼び出し
        BFF->>BFF: 5. 認証処理
        BFF->>API: 6. 中継
        API->>DB: 7. DB検索
        DB->>API: 8. 返却
        API->>BFF: 9. JSON
        BFF->>Browser: 10. JSON
    ```

    **特徴:**

    - 📄 **完全SPA（CSR）**
    - 🔀 **BFFはAPIゲートウェイ**
    - 🔐 **認証・API集約をBFFで実施**

    **技術例:**

    - **フロント:** React/Vue（Vite）
    - **BFF:** Node.js, Go, .NET
    - **バックエンド:** Spring Boot, FastAPI

    ---

    ### **BFF（Backend For Frontend）の役割:**

    - ✅ バックエンドAPIへの中継
    - ✅ 認証・セッション管理
    - ✅ 複数APIの集約・変換
    - ✅ フロント向けデータ最適化
    - ✅ CORS対策、レート制限
    - ✅ SSR（3-Aのみ）

    ### **メリット:**

    - ✅ フロント・バックの完全分離
    - ✅ マイクロサービス対応
    - ✅ BFFでフロント特化処理
    - ✅ 複数APIを1つに集約
    - ✅ 初回表示が速い（3-A）

    ### **デメリット:**

    - ❌ 構成が複雑（3層）
    - ❌ BFF層の運用・保守が必要
    - ❌ 役割分担の設計が重要

---

## サーバー構成の比較表

??? abstract "どの構成を選ぶべきか"

    | 項目 | パターン1<br/>モノリシック | パターン2<br/>フロント・バック分離 | パターン3<br/>BFF構成 |
    |------|----------------------|-------------------|----------------------|
    | **サーバー数** | 1つ | 2つ（静的+API） | 3つ（静的+BFF+API） |
    | **レンダリング** | SSR/CSR/SSG<br/>自由に選択 | 通常CSR | SSR/CSR<br/>選択可能 |
    | **フロント・バック** | 統合 | 完全分離 | 完全分離 |
    | **開発の複雑さ** | ✅ シンプル | ⚠️ やや複雑 | ❌ 複雑（3層） |
    | **スケーリング** | ❌ 困難 | ✅ 容易 | ✅ 容易 |
    | **マルチクライアント** | ❌ 困難 | ✅ 容易 | ✅ 容易 |
    | **認証管理** | アプリ内 | フロント or API | BFF |
    | **適用例** | 小〜中規模<br/>社内システム<br/>ブログ | モダンWebアプリ<br/>SaaS<br/>ダッシュボード | 大規模<br/>ECサイト<br/>エンタープライズ |

---

## レンダリング方式とサーバー構成の組み合わせ

??? abstract "実際のアプリケーション例"

    | 構成 | レンダリング | 技術例 | 用途 |
    |------|------------|--------|------|
    | **パターン1** | SSR | Rails, Django, Laravel | 社内システム、CMS |
    | **パターン1** | SSR+CSR | Next.js単体（API Routes） | 小規模Webアプリ |
    | **パターン2** | CSR | React + Spring Boot | モダンWebアプリ |
    | **パターン3-A** | SSR+SPA | Next.js + Spring Boot | ECサイト、メディア |
    | **パターン3-B** | CSR | React + BFF + Spring Boot | 管理画面、ダッシュボード |

    !!! tip "選択のポイント"
        - **小〜中規模 + 開発速度重視** → パターン1（モノリシック）
        - **フロント・バック独立開発 + モバイルアプリ連携** → パターン2（分離）
        - **大規模 + SEO重視 + マイクロサービス** → パターン3（BFF）

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
