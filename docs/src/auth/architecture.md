# アーキテクチャ検討

??? info "📌 前提条件"

    ### システム構成

    - **フロントエンド**: Next.js (BFF パターン)
    - **バックエンド**: Spring Boot (API サーバ)
    - **サービス数**: 初期3サービス（今後も増加見込み）
    - **SSO要件**: 1回のログインで全サービスにアクセス可能

    ### 認証基盤

    - **社内IdP**: HENNGE One（社内既存認証基盤）
    - **アプリ側トークン発行**: Amazon Cognito User Pool（HENNGE One を外部 IdP としてフェデレーション）
    - **連携方式**: OIDC （または SAML）

    ### 権限管理の要件

    - グループベースの基本権限（Cognito User Pool の `groups` 属性で管理）
    - 複雑な権限要件にも対応可能な設計（将来拡張性）
    - 権限変更の即時反映が必要
    - 監査ログの記録が必須

---

## 🤔 検討ポイントと選択肢

前提条件のもとで、以下の項目について最適な選択肢を検討します。

??? note "検討1: 認可方式と権限データの持ち方"

    **課題**: 誰がどのリソースにアクセスできるかの権限チェックの仕組みをどのような構成で実現するか

    ### 選択肢の概要

    **① JWT埋め込み方式**

    - JWT に個別の詳細権限（リソースごとのアクセス権など）を埋め込む
    - Cognito Pre Token Generation Lambda で DB から権限を取得してトークンに追加
    - 権限判定はトークンの検証のみで完結

    **② OIDC Groups方式**

    - Cognito の `groups` 属性のみで権限管理
    - シンプルなグループベースのアクセス制御（例: admin, user, guest）
    - 複雑な権限要件には対応困難

    **③ AVP実行時チェック方式**

    - JWT には最小限の情報（`userId` + `groups`）のみ
    - Amazon Verified Permissions (AVP) が実行時に Cedar ポリシーで権限を評価
    - Redis キャッシュ + イベント駆動の即時無効化で高速化

    ### 比較表

    | 比較項目 | ① JWT埋め込み | ② OIDC Groups | ③ AVP実行時チェック | 最適 |
    |---------|--------------|--------------|-------------------|------|
    | **JWT サイズ** | ❌ 大きくなりやすい<br>（8KB超過リスク） | ✅ 小さい<br>（~1KB） | ✅ 小さい<br>（~1KB） | ②③ |
    | **権限の柔軟性** | 🟡 埋め込み内容次第 | ❌ グループのみ<br>複雑な要件に非対応 | ✅ Cedar ポリシーで<br>複雑な要件に対応 | ③ |
    | **即時反映** | ❌ JWT再発行まで<br>反映されない | ❌ JWT再発行まで<br>反映されない | ✅ ポリシー更新で<br>即座に反映 | ③ |
    | **監査ログ** | 🟡 トークン検証ログのみ | 🟡 トークン検証ログのみ | ✅ CloudWatch Logs に<br>判定詳細を記録 | ③ |
    | **パフォーマンス** | ✅ トークン検証のみ | ✅ トークン検証のみ | 🟢 Redis キャッシュで<br>高速化 | ①②③ |
    | **実装コスト** | 🟡 Lambda実装が必要 | ✅ 最小限 | 🟡 AVP設定が必要 | ② |

    **推奨する選択肢**: ✅ **③ AVP実行時チェック方式**

    **選択理由**:

    - ✅ **JWT サイズ最小化**: `userId` + `groups` のみで~1KB、トークンサイズ問題を回避
    - ✅ **柔軟性**: Cedar ポリシーで複雑な権限要件（部署別、プロジェクト別など）に対応可能
    - ✅ **即時反映**: ポリシー更新で権限変更が反映される（JWT 再発行不要）
    - ✅ **監査**: CloudWatch Logs に認可判定の詳細ログを自動記録
    - ✅ **パフォーマンス**: Redis キャッシュで高速な判定が可能

    **① を採用しない理由**:

    - ❌ **JWT サイズ超過リスク**: 詳細権限を埋め込むと8KB制限を超える可能性
    - ❌ **即時反映不可**: 権限変更がJWT再発行まで反映されない（セキュリティリスク）
    - ❌ **Lambda パフォーマンス**: DB取得処理でトークン発行が遅延する可能性

    **② を採用しない理由**:

    - ❌ **柔軟性不足**: グループベースのみで、複雑な権限要件（リソース単位の制御など）に非対応
    - ❌ **即時反映不可**: グループ変更がJWT再発行まで反映されない
    - ❌ **監査ログ限定的**: 詳細な認可判定の記録が困難

    ??? example "③ 推奨方式の実装詳細"

        **JWT の内容**:

        - `userId`, `email`, `groups`（Cognito標準クレーム）
        - トークンサイズ: ~1KB

        **権限判定フロー**:

        1. BFF (Next.js) が AVP に権限チェックをリクエスト
        2. AVP が Cedar ポリシーを評価（キャッシュヒット時はRedisから取得）
        3. 許可/拒否を返却、CloudWatch Logs に詳細を記録

        **キャッシュ戦略**:

        **主戦略: イベント駆動無効化**

        - ポリシー更新時: EventBridge → Lambda → Redis キャッシュ削除
        - グループ変更時: Cognito Post Confirmation → Lambda → Redis キャッシュ削除
        - イベント駆動で自動的にキャッシュを無効化

        **副戦略: TTL フォールバック**

        - キャッシュ TTL: 5分
        - イベント欠損時の安全ネット
        - TTL期限切れ後に新しい権限が反映

        **キャッシュキー設計**:

        ```
        avp:{userId}:{action}:{resource}
        例: avp:user123:viewCustomer:customer456
        ```

        ??? example "コスト試算（具体例）"

            **前提: 1,000 DAU、各ユーザー1日30リクエスト、想定キャッシュヒット率80%**

            **キャッシュなし（毎回 AVP 呼び出し）**

            ```
            AVP コール数/月 = 1,000 × 30 × 30 = 900,000 コール
            AVP コスト = 900,000 / 1,000 × $0.15 = $135/月
            Redis コスト = $0/月
            合計 = $135/月
            ```

            **Redis キャッシュあり（キャッシュヒット率80%）**

            ```
            AVP コール数/月 = 900,000 × 20% = 180,000 コール
            AVP コスト = 180,000 / 1,000 × $0.15 = $27/月
            Redis コスト (cache.t4g.micro) = $12/月
            合計 = $39/月

            → $96/月の削減（71% コストダウン）
            ```

            !!! success "結論: キャッシュ戦略はコスト削減に大きく貢献"
                - AVP API の従量課金コストを80%削減
                - Redis の固定費を考慮しても、トータルで70%以上のコスト削減
                - ユーザー数・リクエスト数が増えるほど削減効果は大きくなる

        **Cognito グループ管理**:

        - 初回登録: 初期グループ（例: `guest`）を自動付与
        - 権限付与: 管理者が Cognito コンソール/API で設定
        - JWT 反映: Cognito が自動的に `cognito:groups` クレームに含める

    !!! note "Pre Token Generation Lambda について"
        Cognito で JWT の内容を追加・変換する場合は **Pre Token Generation Lambda トリガー** を実装できます。本構成ではトークンサイズを最小限に保つため基本的な `userId` と `groups` の付与にとどめ、グループ名の変換やカスタムクレーム付与が必要な場合のみ Lambda を導入します。Lambda を挟む際はコールドスタートや外部アクセスによる遅延を避けるため、処理内容は軽量に保ち、外部 DB 参照などは行わない方針とします。

??? note "検討2: ブラウザとBFF間の認証情報の受け渡し方式"

    **課題**: ブラウザとNext.js BFF間で認証状態をどう管理し、トークンをどう保持するか？

    !!! info "BFF 構成における前提"
        本構成では **BFF (Backend for Frontend) パターン** を採用しているため、ブラウザ⇔BFF間の認証情報の受け渡しは **Cookie** を使用します。

        **Cookie を採用する理由**:

        - ✅ ブラウザとBFFは同一オリジン（同じドメイン）なのでCookieが自然
        - ✅ HttpOnly 属性によりJavaScriptからアクセス不可（XSS対策）
        - ✅ Next.js の SSR/SSG 機能との親和性が高い
        - ✅ NextAuth.js の機能を活用できる

        **Authorizationヘッダーを採用しない理由**:

        - ❌ トークンをlocalStorage等に保存する必要があり、XSS脆弱性のリスク
        - ❌ SSR時にトークンを取得できない
        - ℹ️ 注: Authorization ヘッダーは、モバイルアプリ⇔API や BFF⇔バックエンドAPI 間では有効

    Cookieを前提とした場合、次の議論は **Cookieに何を格納するか** です。

    ### 選択肢の概要

    **① ステートレスJWT方式**

    - Cookie に Cognito の JWT（Access Token、Refresh Token）をそのまま平文で格納
    - サーバー側のストレージ不要でステートレス
    - NextAuth.js を使わない独自実装が必要

    **② 暗号化JWT方式（NextAuth.js JWT戦略）**

    - Cookie に暗号化した JWT を格納（NextAuth.js `strategy: "jwt"` で自動対応）
    - サーバー側のストレージ不要でステートレス
    - NextAuth.js が暗号化・復号化を自動処理

    **③ セッションストア方式（NextAuth.js Database戦略）**

    - Cookie にはセッションIDのみを格納（数十バイト）
    - Cognito トークンはサーバー側のセッションストア（DynamoDB/Redis）に保存
    - データベースアダプターの設定が必要

    ### 比較表

    | 比較項目 | ① ステートレスJWT | ② 暗号化JWT | ③ セッションストア | 最適 |
    |---------|------------------|-------------|-------------------|------|
    | **実装の容易さ** | ❌ 自前実装が必要 | 🟢 NextAuth.js が<br>自動対応 | 🟡 DynamoDB/Redisの<br>準備が必要 | ② |
    | **Cookie サイズ** | ❌ 3〜5KB<br>（4KB制限超過リスク） | ❌ 3〜5KB<br>（暗号化でさらに増） | ✅ 数十バイト | ③ |
    | **セキュリティ（XSS）** | ❌ 平文トークンが<br>Cookie内に存在 | 🟡 暗号化されているが<br>Cookie内に存在 | ✅ トークンがブラウザに<br>一切露出しない | ③ |
    | **即時無効化** | ❌ 不可 | ❌ 不可 | ✅ サーバー側で可能 | ③ |
    | **Cognitoトークン更新** | ❌ 自前実装が必要 | 🟡 `jwt`コールバックで<br>ロジック実装が必要 | ✅ デフォルトで自動処理 | ③ |
    | **CSRF対策** | ❌ 自前実装が必要 | ✅ NextAuth.js が自動実装 | ✅ NextAuth.js が自動実装 | ②③ |
    | **ステートレス性** | ✅ ステートレス | ✅ ステートレス | ❌ セッションストア必要 | ①② |
    | **運用コスト** | ✅ ストレージ不要 | ✅ ストレージ不要 | 🟡 セッションストアの<br>管理が必要 | ①② |

    **推奨する選択肢**: ✅ **③ セッションストア方式**

    **選択理由**:

    - ✅ **セキュリティ最優先**: Cognitoトークンがブラウザに一切露出せず、XSS攻撃のリスクを最小化
    - ✅ **即時無効化**: ログアウトや権限変更時にサーバー側でセッションを無効化可能
    - ✅ **Cookie サイズ**: 4KB制限を気にする必要が一切ない
    - ✅ **実装の簡潔さ**: NextAuth.js のデフォルト動作でトークン更新・CSRF対策が自動処理
    - 🟡 セッションストアの準備が必要だが、DynamoDB（サーバーレス）で運用負荷を最小化

    **① を採用しない理由**:

    - ❌ **Cookie サイズ超過**: Cognito JWT（2〜3KB）+ Refresh Token で4KB制限を超える可能性が高い
    - ❌ **セキュリティリスク**: 平文トークンがCookieに格納される
    - ❌ **即時無効化不可**: ログアウト後もトークン有効期限まで使用可能
    - ❌ **実装コスト**: トークン更新・CSRF対策などを自前実装する必要がある

    **② を採用しない理由**:

    - ❌ **Cookie サイズ超過**: 暗号化オーバーヘッドで①よりさらに大きくなり、4KB制限超過の可能性が極めて高い
    - ❌ **即時無効化不可**: ログアウト後もトークン有効期限まで使用可能（セキュリティリスク）
    - 🟡 **トークン更新実装**: `jwt` コールバックで Cognito Refresh Token のロジックを実装する必要がある
    - ℹ️ ステートレスが要件の場合は選択肢になるが、本構成では③が適切

    ??? example "③ 推奨方式の実装詳細"

        **Cookie の内容**:

        - セッションIDのみ（数十バイト、例: `session_abc123xyz...`）
        - `HttpOnly`, `Secure`, `SameSite=Lax` 属性を設定

        **セッションストアの内容**:

        - Cognito ID Token, Access Token, Refresh Token
        - ユーザー情報（userId, email, groups）
        - セッション有効期限

        **自動処理される機能**:

        - Refresh Token による Access Token の自動更新
        - CSRF トークンの生成・検証
        - セッションの有効期限管理

        **インフラ構成**:

        - 初期: DynamoDB（サーバーレス、低コスト）
        - スケール時: Redis（ElastiCache）追加でパフォーマンス最適化

    ??? warning "セキュリティ上の考慮事項"

        **JWT の取り扱い**

        - ✅ JWT には `userId` + `groups` のみ格納（詳細権限は含めない）
        - ✅ トークン盗難時の被害を最小化
        - ✅ 権限変更時に JWT 再発行不要（AVP ポリシー更新のみ）

        **Cookie セキュリティ属性**

        - ✅ `HttpOnly`: JavaScript からアクセス不可（XSS 対策）
        - ✅ `Secure`: HTTPS 通信でのみ送信
        - ✅ `SameSite=Lax`: CSRF 攻撃を防止

        **トークン有効期限**

        - ID Token: 1時間（Cognito デフォルト）
        - Access Token: 1時間（Cognito デフォルト）
        - Refresh Token: 設定した有効期限（NextAuth.js が自動更新）
        - セッション: NextAuth.js が Refresh Token を使用して自動延長

        **セッション無効化**

        - ログアウト時: サーバー側でセッションを削除
        - 権限変更時: 必要に応じてセッションを無効化可能
        - 不正アクセス検知時: 管理者が特定ユーザーのセッションを強制終了可能

??? note "検討3: 認証・認可チェックの実施箇所"

    **課題**: ブラウザ → BFF → API の通信経路において、どのタイミングで認証・認可チェックを実施するか？

    ### 選択肢の概要

    **① BFF集中チェック方式（APIは無検証）**

    - ブラウザ→BFF: Cookie（セッションID）で認証 + AVP で認可チェック
    - BFF→API: 認証・認可チェックなしでリクエストを転送（内部通信として信頼）
    - すべてのセキュリティチェックを BFF に集中

    **② 多層防御方式（BFF・API両方でチェック）**

    - ブラウザ→BFF: Cookie（セッションID）で認証 + AVP で認可チェック
    - BFF→API: JWT（Access Token）をヘッダーに付与し、API側でJWT検証を実施
    - 各層で独立した検証を実施

    **③ API Gateway 集中制御方式**

    - ブラウザ→BFF: Cookie（セッションID）で認証 + AVP で認可チェック
    - BFF→API Gateway→API: API Gateway の Lambda Authorizer で JWT検証・認可チェック
    - Gateway で集中的にセキュリティ制御

    ### 比較表

    | 比較項目 | ① BFF 集中 | ② 多層防御 | ③ Gateway 集中 | 最適 |
    |---------|-----------|-----------|---------------|------|
    | **早期リジェクション** | ✅ BFF で拒否 | ✅ BFF で拒否 | 🟡 Gateway で拒否<br>（BFF経由後） | ①② |
    | **多層防御** | ❌ API が無防備<br>（BFF信頼前提） | ✅ BFF・API で<br>二重チェック | ✅ Gateway・API で<br>二重チェック可能 | ②③ |
    | **実装の容易さ** | ✅ BFF のみ実装 | 🟡 BFF・API 両方に実装 | ❌ Gateway + Lambda<br>追加実装が必要 | ① |
    | **API 単体利用** | ❌ 不可<br>（BFF 必須） | ✅ 可能<br>（モバイルアプリ等） | ✅ 可能<br>（Gateway 経由） | ②③ |
    | **ページレベル制御** | ✅ BFF で SSR 制御 | ✅ BFF で SSR 制御 | ✅ BFF で SSR 制御 | ①②③ |
    | **運用コスト** | ✅ 最小限 | 🟡 やや増加 | ❌ Gateway 管理が<br>追加で必要 | ① |
    | **責務の明確さ** | 🟡 BFF に集中 | ✅ 明確に分離 | 🟡 Gateway が集中 | ② |

    **推奨する選択肢**: ✅ **② 多層防御方式（BFF・API 両方でチェック）**

    **選択理由**:

    - ✅ **早期リジェクション**: BFF で認証・認可チェック、不正リクエストは API に到達させない
    - ✅ **多層防御**: API 側でも JWT 検証を実施し、BFF をバイパスした攻撃に対応
    - ✅ **柔軟性**: 将来的なモバイルアプリなど、BFF を経由しない API 直接アクセスに対応可能
    - ✅ **責務の明確化**: BFF（認証・認可・UI制御）と API（JWT検証・ビジネスロジック）の役割が明確
    - ✅ **実装のシンプルさ**: API Gateway 不要、BFF から直接 AVP SDK を呼び出し

    **① を採用しない理由**:

    - ❌ **セキュリティリスク**: BFF が侵害された場合や設定ミスがあった場合、API が無防備になる
    - ❌ **API 単体利用不可**: 将来的なモバイルアプリなどの要件に対応困難
    - ❌ **ゼロトラストに非対応**: BFF と API 間の内部通信を無条件に信頼する前提が必要

    **③ を採用しない理由**:

    - ❌ **BFF 構成で不要**: BFF 自体がゲートウェイの役割を果たしているため冗長
    - ❌ **実装・運用コスト**: Lambda Authorizer の実装・運用が追加で必要
    - ❌ **アーキテクチャの複雑化**: BFF と Gateway で役割が重複
    - ℹ️ BFF を使わない構成（SPA + API）の場合は有効な選択肢

    ??? example "② 推奨方式の実装詳細"

        **ブラウザ → BFF 間のチェック**:

        - **認証**: Cookie（セッションID）の有効性チェック
        - **セッション検証**: DynamoDB/Redis からセッション情報を取得
        - **認可**: AVP を呼び出して権限チェック（Redis キャッシュ活用）
        - **SSR 制御**: 権限に応じたページ生成・リダイレクト

        **BFF → API 間のチェック**:

        - **トークン付与**: BFF がセッションストアから Cognito Access Token を取得し、Authorization ヘッダーに設定
        - **JWT 検証**: API（Spring Boot）が JWT 署名・有効期限・Issuer を検証
        - **基本的な認可**: JWT の `groups` クレームで簡易チェック（詳細な認可は BFF で完了済み）

        **責務分担**:

        | 層 | 認証 | 認可 | その他の役割 |
        |---|-----|-----|------------|
        | **BFF (Next.js)** | ✅ セッション検証 | ✅ AVP で詳細チェック | UI制御、SSR、早期リジェクション、ページレベル制御 |
        | **API (Spring Boot)** | ✅ JWT 検証 | 🟡 Groups で簡易チェック | ビジネスロジック、データアクセス、多層防御 |

        **責務分担のポイント**:

        - **BFF**: 全ての認証・認可チェックを実施（ページアクセス時）
          - Cookie（セッションID）の検証
          - AVP による詳細な権限チェック
          - 不正リクエストは API に到達させない
          - 権限に応じた UI の出し分け

        - **API**: JWT 検証 + Groups 簡易チェック（多層防御として）
          - JWT の署名・有効期限・Issuer 検証
          - `groups` クレームによる基本的な権限確認
          - 詳細な認可ロジックは BFF に委譲
          - ビジネスロジックに専念

        **実装例**:

        ```typescript
        // BFF (Next.js Server Component)
        async function getProtectedData() {
          const session = await getServerSession(); // セッション検証

          // 認可チェック
          const allowed = await avp.isAuthorized({
            principal: session.user.id,
            action: "read",
            resource: "orders"
          });

          if (!allowed) {
            redirect('/unauthorized');
          }

          // API 呼び出し（Access Token 付与）
          const response = await fetch('https://api.example.com/orders', {
            headers: {
              'Authorization': `Bearer ${session.accessToken}`
            }
          });

          return response.json();
        }
        ```

        ```java
        // API (Spring Boot)
        @RestController
        @RequestMapping("/orders")
        public class OrderController {

          @GetMapping
          @PreAuthorize("hasAuthority('SCOPE_api/read')")
          public List<Order> getOrders(@AuthenticationPrincipal Jwt jwt) {
            // JWT は既に検証済み
            String userId = jwt.getSubject();

            // ビジネスロジック
            return orderService.findByUserId(userId);
          }
        }
        ```

    !!! note "モバイルアプリなど BFF 非経由アクセスへの対応"
        モバイルアプリなど BFF を経由しない直接 API アクセスがある場合は、該当する API エンドポイントで AVP による詳細な認可チェックも実装します（多層防御）。この場合、API 側でも BFF と同様の認可ロジックを持つことになります。

??? note "検討4: SSO 実装方針"

    **課題**: 複数サービス間でのログイン体験をどう実現するか？

    ### 選択肢の概要

    **① Cookie共有方式**

    - 親ドメインに共有 Cookie を設定（例: `.example.com`）
    - 全サブドメインでセッション共有、サービス間移動時のリダイレクトなし
    - サブドメイン構成が前提条件

    **② サイレント認証方式（`prompt=none`）**

    - IdP（HENNGE One）のセッションを活用したリダイレクトベース SSO
    - 各サービスでホスト限定 Cookie、サービス間移動時はサイレント認証経由
    - ドメイン構成に依存しない

    **③ カスタムトークンゲートウェイ方式**

    - 専用の認証ゲートウェイでトークンを一元管理
    - サービス間でトークンを引き回し、各サービスで検証
    - 完全な中央集権型アーキテクチャ

    ### 比較表

    | 比較項目 | ① Cookie共有 | ② サイレント認証 | ③ トークンGW | 最適 |
    |---------|-------------|----------------|-------------|------|
    | **IdP セッション活用** | 🟡 初回のみ | ✅ 毎回活用 | 🟡 初回のみ | ② |
    | **ドメイン制約** | ❌ サブドメイン必須<br>（`.example.com`） | ✅ 制約なし | ✅ 制約なし | ②③ |
    | **実装の容易さ** | 🟡 Cookie設定の調整 | ✅ NextAuth.js標準<br>（`prompt=none`追加のみ） | ❌ GW実装が必要 | ② |
    | **サービス間遷移** | ✅ 即座（0秒） | 🟡 サイレント認証経由<br>（リダイレクト発生） | 🟡 トークン引き回し<br>（リダイレクト発生） | ① |
    | **セキュリティ** | 🟡 Cookie範囲が広い | ✅ ホスト限定Cookie | ✅ トークン分離 | ②③ |
    | **スケーラビリティ** | 🟡 サブドメイン制約 | ✅ サービス追加容易 | 🟡 GW が SPOF | ② |
    | **運用コスト** | ✅ 最小限 | ✅ 最小限 | ❌ GW運用が必要 | ①② |

    **推奨する選択肢**: ✅ **② サイレント認証方式（`prompt=none`）**

    **選択理由**:

    - ✅ **IdP セッション活用**: HENNGE One のログイン状態を毎回再利用、IdP 側でのログアウトが次回アクセス時に全サービスに反映
    - ✅ **ドメイン制約なし**: サブドメイン構成に依存せず、将来的なドメイン変更にも柔軟に対応
    - ✅ **実装の軽さ**: NextAuth.js 標準フローに `prompt=none` パラメータを追加するのみ
    - ✅ **セキュリティ**: 各サービスがホスト限定 Cookie で独立、Cookie 漏洩時の影響範囲を最小化
    - ✅ **スケーラビリティ**: 新規サービス追加時も同じフローを適用、追加実装不要
    - 🟡 サービス間遷移にリダイレクトが発生するが、UX上の問題は限定的

    **① を採用しない理由**:

    - ❌ **ドメイン制約**: サブドメイン構成（`service-a.example.com`, `service-b.example.com`）が必須
    - ❌ **Cookie範囲**: 親ドメイン全体に Cookie が送信され、セキュリティリスクが高まる
    - ❌ **IdP ログアウト反映**: HENNGE One でログアウトしても、ブラウザ上の共有 Cookie が残る可能性
    - ℹ️ サブドメイン構成が確定しており、リダイレクトなしの遷移が必須要件の場合は選択肢になる

    **③ を採用しない理由**:

    - ❌ **実装・運用コスト**: 専用のトークンゲートウェイの実装・運用が必要
    - ❌ **SPOF（単一障害点）**: ゲートウェイがダウンすると全サービスが影響を受ける
    - ❌ **アーキテクチャの複雑化**: BFF + GW で責務が重複、メンテナンス性が低下
    - ℹ️ 既存の認証ゲートウェイがある環境では選択肢になる

    ??? example "② 推奨方式の実装詳細"

        **認証フロー**:

        - Cognito → HENNGE One へのサイレントリダイレクト
        - 各サービスでホスト限定 Cookie（`service-a.example.com`, `service-b.example.com` など）
        - SSO は IdP セッションに依存

        **ユーザー体験**:

        ```text
        【初回訪問】
        1. サービスA にアクセス → Cognito Hosted UI（HENNGE One へリダイレクト）
        2. ユーザーが HENNGE One でログイン（メールアドレス + パスワード）
        3. Cognito がトークンを発行しサービスA のページ表示

        【2回目以降（同一サービス）】
        4. サービスA にアクセス
        5. → Cognito に `prompt=none` でリダイレクト → 背後で HENNGE One セッションを再利用
        6. → ページ表示

        【サービス間移動】
        7. サービスA で作業中
        8. サービスB へのリンクをクリック
        9. → サイレント認証経由で Cognito/HENNGE One を往復しサービスB へ遷移
        ```

        **NextAuth.js 設定例**:

        ```typescript
        // pages/api/auth/[...nextauth].ts
        import NextAuth from "next-auth"
        import CognitoProvider from "next-auth/providers/cognito"

        export default NextAuth({
          providers: [
            CognitoProvider({
              clientId: process.env.COGNITO_CLIENT_ID,
              clientSecret: process.env.COGNITO_CLIENT_SECRET,
              issuer: process.env.COGNITO_ISSUER,
              authorization: {
                params: {
                  prompt: "none" // サイレント認証を有効化
                }
              }
            })
          ],
          session: {
            strategy: "database" // セッションストア方式（検討2の推奨方式）
          }
        })
        ```

        **ユーザー切り替え時の動作**:

        - アプリケーション単体のログアウト機能は提供せず、HENNGE One からサインアウトし、Cognito（Hosted UI）に再ログインしてもらう運用
        - HENNGE One / Cognito のセッションが破棄されると、次回サイトアクセス時に BFF が未認証と判断し新しいユーザーでサイレント認証に遷移
        - ブラウザに残る BFF セッション Cookie は次のサイレント認証完了時に新しいユーザー情報で上書き
        - IdP 側のログアウト完了（HENNGE One と Cognito 双方）を確認してからサイトに戻る運用をユーザーに案内

    !!! info "サイレント認証とは"
        **サイレント認証（Silent Authentication）** は、OIDCの `prompt=none` パラメータを使用して、ユーザーに認証画面を表示せずにバックグラウンドで自動的に認証を行う仕組みです。IdP（HENNGE One）側にセッションが存在する場合、ユーザー操作なしで新しいトークンを取得できます。

    !!! note "トークンフローの整理"
        NextAuth.js からは Amazon Cognito の OIDC エンドポイントを利用します。Cognito は HENNGE One を外部 IdP としてフェデレーションしているため、`prompt=none` で Cognito にアクセスすると、必要に応じて HENNGE One へサイレントにリダイレクトし、既存の社内セッションを再利用します。

---

## 🎯 主要な設計決定のまとめ

??? success "採用した方式の一覧"

    | 検討項目 | 採用した方式 | 主な理由 |
    |---------|-------------|---------|
    | **認可方式** | AVP による実行時権限チェック | JWT サイズ最小化、即時反映、柔軟性、監査ログ |
    | **ブラウザ⇔BFF認証情報管理** | HttpOnly Cookie + サーバー側セッション | XSS対策、トークン非露出、NextAuth.js標準 |
    | **認証・認可実施箇所** | BFF・API 多層防御 | 早期リジェクション、多層防御、柔軟性 |
    | **SSO 実装** | サイレント認証（prompt=none） | IdP セッション活用、ドメイン制約なし、実装軽微 |

??? success "この構成の利点"

    ✅ **セキュリティ**

    - JWT サイズ最小化（`userId` + `groups` のみ）で詳細権限は AVP で管理
    - BFF と API の多層防御、トークンをブラウザに露出させない設計
    - 権限変更・ログアウト時のセッション即時無効化
    - CloudWatch Logs による監査証跡

    ✅ **パフォーマンス・コスト**

    - Redis キャッシュによる高速化と AVP API コスト削減
    - 水平スケールアウト対応

    ✅ **開発・運用効率**

    - NextAuth.js 活用による実装の簡潔さ
    - BFF での早期リジェクション、責務の明確な分離
    - トークン更新・CSRF 対策・セッション管理の自動化

    ✅ **拡張性**

    - 新規サービス追加が容易（同じ認証フローを適用）
    - Cedar ポリシー更新のみで権限要件変更に対応
