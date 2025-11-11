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

??? question "🔐 検討1: ブラウザとBFF間の認証情報の受け渡し方式"

    !!! tip "検討ポイント"

        **① ブラウザとBFF間で認証トークンをどう安全に受け渡すか？**

        - Cookie（HttpOnly）によるセッション管理 vs Authorization ヘッダー
        - XSS攻撃からのトークン保護方法

        **② BFFでセッションをDB管理する必要性があるか？**

        - ステートレス（JWE形式） vs ステートフル（DB永続化セッション）
        - 即時無効化の要件とインフラ複雑度のトレードオフ

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


    !!! abstract "選択肢の概要"

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


    !!! abstract "比較表"

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

    !!! success "推奨する選択肢: ③ セッションストア方式"

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

??? question "🔑 検討2: 認可方式と権限データの持ち方"

    !!! tip "検討ポイント"

        **権限情報をJWTに含めるか、外部サービス(AVP)で管理するか？**

        - JWT詳細権限方式: トークンに権限を埋め込み（サイズ増大リスク）
        - AVP方式: トークンは最小限、認可判定は外部化（柔軟性・即時反映）
        - トークンサイズ制限（8KB）とパフォーマンスのバランス

    !!! abstract "選択肢の概要"

        **① JWT詳細権限方式**

        - **設計方針**: JWTに細かい権限条件を含め、認可チェック時はJWTの情報だけで判断
        - **JWT内容**: `userId`, `groups`, `permissions`（アクション・リソース単位の詳細権限）
        - **トークンサイズ**: 3〜8KB（8KB制限超過リスク）
        - **権限判定**: JWT内の`permissions`配列をチェック（例: `permissions.includes('customers:read')`）
        - **DB参照**: トークン生成時のみ（Pre Token Generation Lambda で権限を取得してJWTに埋め込み）

        **② JWT最小限 + AVP方式**

        - **設計方針**: JWTには最低限のグループ情報のみ、アクション・リソースの概念はAVP側で制御
        - **JWT内容**: `userId`, `email`, `groups`（Cognito標準クレーム）
        - **トークンサイズ**: ~1KB
        - **権限判定**: AVP（Amazon Verified Permissions）がCedarポリシーで評価
        - **DB参照**: 不要（JWTとCedarポリシーのみで判定）

        **③ JWT最小限 + DBアクセスを伴うアプリケーションでのチェック方式**

        - **設計方針**: JWTには最低限のグループ情報のみ、詳細権限はDBから取得してアプリケーションコードで判定
        - **JWT内容**: `userId`, `email`, `groups`（Cognito標準クレーム）
        - **トークンサイズ**: ~1KB
        - **権限判定**: 認可チェック時にDBから権限データを取得し、アプリケーションコードで判定
        - **DB参照**: 認可チェック時（毎回またはキャッシュ）

    !!! abstract "比較表"

        | 比較項目 | ① JWT詳細権限 | ② JWT最小限 + AVP | ③ JWT最小限 + DB権限チェック | 最適 |
        |---------|-------------|-----------------|------------------------|------|
        | **JWT サイズ** | ❌ 3〜8KB<br>（8KB制限超過リスク） | ✅ ~1KB | ✅ ~1KB | ②③ |
        | **トークン発行速度** | ❌ Lambda + DB参照で遅延 | ✅ 高速（DB参照不要） | ✅ 高速（DB参照不要） | ②③ |
        | **認可チェック時のDB参照** | ✅ 不要<br>（JWT内で完結） | ✅ 不要<br>（Cedarポリシーのみ） | ❌ 毎回必要<br>（キャッシュで軽減可） | ①② |
        | **権限の柔軟性** | 🟡 JWT埋め込み内容次第<br>変更にLambda修正必要 | ✅ Cedar ポリシーで<br>複雑な要件に対応 | 🟡 コード実装次第<br>複雑化すると保守困難 | ② |
        | **権限変更の即時反映** | ❌ JWT再発行まで反映されない | ✅ ポリシー変更は即座に反映<br>（グループ変更はJWT再発行必要） | 🟡 DB更新後に反映<br>（キャッシュTTL次第） | ② |
        | **監査ログ** | 🟡 自前実装が必要 | ✅ CloudWatch Logs に<br>全判定詳細を自動記録 | ❌ 自前実装が必要<br>漏れのリスク | ② |
        | **保守性** | ❌ Lambda + コードに<br>権限ロジック散在 | ✅ ポリシー集中管理<br>コードから分離 | ❌ 権限ロジックが<br>コード全体に散在 | ② |
        | **責務の分離** | 🟡 Lambda に権限取得ロジック | ✅ 認可エンジンとして<br>明確に分離 | ❌ 認可ロジックが<br>ビジネスロジックと混在 | ② |
        | **複雑な権限要件への対応** | ❌ JWT再構築・デプロイ必要 | ✅ ポリシー追加のみ<br>デプロイ不要 | ❌ コード修正・デプロイ必要<br>影響範囲が不明確 | ② |
        | **トークン盗難リスク** | ❌ 詳細権限が露出 | ✅ 被害最小（基本情報のみ） | ✅ 被害最小（基本情報のみ） | ②③ |
        | **実装コスト** | 🟡 Lambda実装が必要 | 🟡 AVP設定 + SDK実装 | 🟡 DB設計 + 権限テーブル<br>+ 判定ロジック実装 | - |
        | **運用コスト** | ✅ 追加コストなし | 🟡 AVP + Redis<br>（約 $6〜13/月） | ✅ 追加コストなし<br>（既存DBで対応） | ①③ |
        | **パフォーマンス** | ✅ JWT検証のみで高速 | 🟢 Redis キャッシュで高速<br>（初回のみAVP呼び出し） | 🟡 DB参照で遅延<br>（キャッシュで軽減可） | ① |
        | **テスタビリティ** | 🟡 JWT生成が必要 | ✅ ポリシーを独立して<br>テスト可能 | ❌ コードと一体で<br>テスト、モック必要 | ② |
        | **ポリシー管理** | ❌ Lambda コードに埋め込み | ✅ Cedar で宣言的に管理<br>バージョニング可能 | ❌ コード内に散在<br>一覧性なし | ② |

    !!! abstract "推奨する選択肢: ② JWT最小限 + AVP方式"

        **選択理由**:

        - ✅ **JWT サイズ最小化**: ~1KBに抑え、8KB制限問題を完全回避
        - ✅ **柔軟性**: Cedar ポリシーで複雑な権限要件（部署別、プロジェクト別、リソース属性など）に対応可能
        - ✅ **ポリシー即時反映**: Cedarポリシー変更が即座に反映される（デプロイ不要）
        - ✅ **監査**: CloudWatch Logs に全ての認可判定の詳細ログを自動記録
        - ✅ **保守性**: 権限ロジックがCedarポリシーに集中、アプリケーションコードから分離
        - ✅ **セキュリティ**: トークン盗難時の被害を最小化、トークン発行も高速
        - 🟡 **運用コスト**: AVP + Redis で月額 $6〜13（200 DAU × 200 API コール/日の場合）

        **① を採用しない理由**:

        - ❌ **JWT サイズ超過リスク**: 詳細権限を埋め込むと8KB制限を超える可能性が高い
        - ❌ **トークン発行遅延**: Pre Token Generation Lambda で DB から権限を取得する必要がある
        - ❌ **柔軟性の欠如**: 権限ロジックをJWTに埋め込むと、変更にLambda修正とデプロイが必要
        - ❌ **セキュリティリスク**: 詳細権限情報がJWTに含まれ、盗難時の被害が大きい
        - ❌ **権限変更反映**: JWT再発行まで反映されない（②③のグループ変更と同じ制約だが、詳細権限全体が対象）

        **③ を採用しない理由**:

        - ❌ **認可チェック時のDB参照**: 毎回のAPIコールでDB参照が発生し、レイテンシ増加（キャッシュで軽減可能だが複雑化）
        - ❌ **保守性**: 権限判定ロジックがアプリケーションコード全体に散在し、変更時の影響範囲が不明確
        - ❌ **責務の混在**: 認可ロジックとビジネスロジックが同じコード内に存在し、複雑化しやすい
        - ❌ **複雑な権限要件への対応**: 新しい権限条件追加時にコード修正とデプロイが必要、影響範囲の特定が困難
        - ❌ **監査ログ不足**: 認可判定の詳細な記録を自前で実装する必要があり、記録漏れのリスク
        - ❌ **テスト困難**: DBモックが必要で、権限パターン網羅的なテストが困難
        - ❌ **ポリシー管理**: 権限ルールがコード内に散在し、全体像の把握や一元管理が不可能
        - ℹ️ **②との比較**: 同じDB参照が必要なら、専用の認可エンジン（AVP）に集約した方が保守性・監査性で圧倒的に優位

    !!! warning "グループ変更の反映タイミング"
        ユーザーのグループ変更（例: `guest` → `sales`）は以下の手順で反映されます：

        1. 管理者が Cognito でグループ変更
        2. Cognito Post Confirmation トリガーで Redis キャッシュを削除
        3. **ユーザーが再ログインして新しいJWTを取得**
        4. 新しいJWTの`groups`属性で認可判定が行われる

        **重要**: Cedarポリシーの変更（誰がどのリソースにアクセスできるかのルール）は即座に反映されますが、JWTに含まれる`groups`などのユーザー属性の変更はJWT再発行（再ログイン）まで反映されません。これはJWTの仕様上の制約です。

    ??? example "② 推奨方式の実装詳細"

        **JWT の内容**:

        - `userId`, `email`, `groups`（Cognito標準クレーム）
        - トークンサイズ: ~1KB

        **権限判定フロー**:

        1. BFF (Next.js) が AVP に権限チェックをリクエスト（IsAuthorizedWithToken API）
        2. AVP が Cedar ポリシーを評価（キャッシュヒット時はRedisから取得）
        3. 許可/拒否を返却、CloudWatch Logs に詳細を記録

        **具体例: グループベースの判定**

        ```typescript
        // BFF (Next.js API Route)
        export async function GET(req: Request) {
          const session = await getServerSession();
          const token = session.accessToken;

          // AVP で認可チェック
          const result = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: {
              actionType: "Action",
              actionId: "listCustomers"
            },
            resource: {
              entityType: "APIEndpoint",
              entityId: "/api/customers"
            }
          });

          if (result.decision !== 'ALLOW') {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 認可OK、データ取得
          const customers = await db.getCustomers();
          return Response.json(customers);
        }
        ```

        **Cedar ポリシー例**:

        ```cedar
        // 営業グループは顧客一覧にアクセス可能
        permit(
          principal,
          action == Action::"listCustomers",
          resource == APIEndpoint::"/api/customers"
        ) when {
          principal.groups.contains("sales") ||
          principal.groups.contains("sales_manager")
        };

        // 管理者は全APIにアクセス可能
        permit(
          principal,
          action,
          resource
        ) when {
          principal.groups.contains("admin")
        };
        ```

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
        例: avp:user123:listCustomers:/api/customers
        ```

        ??? example "コスト試算（具体例）"

            **料金情報**:

            - [Amazon Verified Permissions 料金](https://aws.amazon.com/verified-permissions/pricing/)
              - Single Authorization Request: $0.000005/リクエスト
              - **2025年6月12日に大幅値下げ** (旧価格の約1/30): [AWS、Amazon Verified Permissionsの料金を大幅値下げ](https://dev.classmethod.jp/articles/verified-permissions-reduces-price/)
            - [Amazon ElastiCache 料金](https://aws.amazon.com/elasticache/pricing/)
              - 参考: cache.t4g.micro (US East) 約 $0.016/時間 = 約 $12/月

            **前提: 200 DAU、各ユーザー1日200 APIコール（全てのAPIコールで認可チェック）、想定キャッシュヒット率80%**

            **キャッシュなし（毎回 AVP 呼び出し）**

            ```
            AVP コール数/月 = 200 ユーザー × 200 リクエスト/日 × 30 日 = 1,200,000 リクエスト
            AVP コスト = 1,200,000 × $0.000005 = $6.00/月
            Redis コスト = $0/月
            合計 = $6.00/月
            ```

            **Redis キャッシュあり（キャッシュヒット率80%）**

            ```
            AVP コール数/月 = 1,200,000 × 20% = 240,000 リクエスト
            AVP コスト = 240,000 × $0.000005 = $1.20/月
            Redis コスト (cache.t4g.micro) = 約 $12/月
            合計 = 約 $13.20/月
            ```

            !!! info "コスト vs パフォーマンスのトレードオフ"
                **コスト面:**

                - AVP 単体: $6.00/月 → Redis 併用: $13.20/月（約2.2倍）
                - 200 DAU × 200 API コール/日の規模では、Redis の固定費（$12/月）が AVP コスト削減額（$4.80/月）を上回る

                **パフォーマンス面:**

                - Redis キャッシュにより認可チェックのレスポンス時間を大幅に短縮
                - AVP API 呼び出しを80%削減することで、API レート制限への余裕が生まれる
                - ユーザー体験の向上（画面遷移・操作の高速化）

                **推奨:**

                - 初期段階ではキャッシュなしで運用し、コストを最小化
                - ユーザー数増加やパフォーマンス要件に応じて Redis キャッシュを導入
                - 1,000 DAU 以上では Redis 導入によるコスト削減効果が顕著になる

        **Cognito グループ管理**:

        - 初回登録: 初期グループ（例: `guest`）を自動付与
        - 権限付与: 管理者が Cognito コンソール/API で設定
        - JWT 反映: Cognito が自動的に `cognito:groups` クレームに含める

    ??? example "① JWT詳細権限方式の実装例（参考）"

        **Pre Token Generation Lambda（トークン発行時）**

        ```typescript
        export const handler = async (event) => {
          const userId = event.request.userAttributes.sub;

          // DBから詳細権限を取得
          const permissions = await db.getUserPermissions(userId);
          // 例: ['customers:read', 'customers:write', 'orders:read', ...]

          // JWTに埋め込み（サイズ超過リスク）
          event.response = {
            claimsOverrideDetails: {
              claimsToAddOrOverride: {
                permissions: JSON.stringify(permissions) // 大きくなる可能性
              }
            }
          };

          return event;
        };
        ```

        **BFF (Next.js API Route)**

        ```typescript
        export async function GET(req: Request) {
          const session = await getServerSession();
          const permissions = JSON.parse(session.user.permissions);

          // JWTの権限情報で判定
          if (!permissions.includes('customers:read')) {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          const customers = await db.getCustomers();
          return Response.json(customers);
        }
        ```

        **課題**:

        - トークンサイズが大きくなり、8KB制限を超えるリスク
        - 権限変更がJWT再発行（再ログイン）まで反映されない
        - Lambda実装とDB参照によるトークン発行の遅延
        - 詳細権限情報が盗難時に露出

    ??? example "③ JWT最小限 + DB権限チェック方式の実装例（参考）"

        **権限テーブル設計**

        ```sql
        CREATE TABLE user_permissions (
          user_id UUID NOT NULL,
          resource VARCHAR(100) NOT NULL,
          action VARCHAR(50) NOT NULL,
          granted_at TIMESTAMP DEFAULT NOW(),
          PRIMARY KEY (user_id, resource, action)
        );

        -- 例: ユーザー123は顧客データの読み取りが可能
        INSERT INTO user_permissions (user_id, resource, action)
        VALUES ('user-123', 'customers', 'read');
        ```

        **BFF (Next.js API Route)**

        ```typescript
        export async function GET(req: Request) {
          const session = await getServerSession();
          const userId = session.user.id;

          // DBから権限を取得（毎回またはキャッシュ）
          const hasPermission = await db.query(
            'SELECT 1 FROM user_permissions WHERE user_id = $1 AND resource = $2 AND action = $3',
            [userId, 'customers', 'read']
          );

          if (!hasPermission) {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 認可OK、データ取得
          const customers = await db.getCustomers();
          return Response.json(customers);
        }
        ```

        **キャッシュを使った改善版**

        ```typescript
        // 権限チェック関数（キャッシュ付き）
        async function checkPermission(userId: string, resource: string, action: string): Promise<boolean> {
          const cacheKey = `perm:${userId}:${resource}:${action}`;

          // キャッシュチェック
          const cached = await redis.get(cacheKey);
          if (cached !== null) {
            return cached === 'true';
          }

          // DBから取得
          const result = await db.query(
            'SELECT 1 FROM user_permissions WHERE user_id = $1 AND resource = $2 AND action = $3',
            [userId, resource, action]
          );

          const hasPermission = result.rowCount > 0;

          // キャッシュに保存（5分）
          await redis.setex(cacheKey, 300, hasPermission ? 'true' : 'false');

          return hasPermission;
        }

        export async function GET(req: Request) {
          const session = await getServerSession();

          if (!await checkPermission(session.user.id, 'customers', 'read')) {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          const customers = await db.getCustomers();
          return Response.json(customers);
        }
        ```

        **課題**:

        - **権限ロジックの散在**: 各エンドポイントで`checkPermission`を呼び出す必要があり、実装漏れのリスク
        - **DB負荷**: キャッシュがない場合、毎回DBクエリが発生（AVPも外部呼び出しだが、専用の高速エンジン）
        - **複雑な条件への対応困難**: 「自分の部署の顧客のみ」などの条件は別途実装が必要
        - **監査ログ**: `checkPermission`内で個別に実装する必要があり、記録漏れや形式不統一のリスク
        - **保守性**: 権限テーブル設計・キャッシュ戦略・判定ロジックをすべて自前で管理
        - **テスト**: DBモック・キャッシュモックが必要で、テストコードが複雑化
        - **②（AVP）との比較**: 同じ外部アクセス（AVP vs DB）なら、Cedar という宣言的ポリシー言語で管理でき、監査ログが自動記録されるAVPの方が優位

??? question "🎯 検討3: リソース属性による認可制御の実装方針"

    !!! tip "検討ポイント"

        **ユーザーごとに参照可能なデータが異なる場合の実装方式は？**

        - ユースケース例: 「営業部のユーザーは自分の部署の顧客のみ表示」「ユーザーは自分が作成した注文のみ表示」
        - AVP事前チェック方式: リソース取得後にAVPで個別判定（セキュリティ重視）
        - DB条件付き取得方式: WHERE句で絞り込み（パフォーマンス重視）
        - Cedar Query方式: 複数リソースの一括判定（大量データ処理）

    !!! abstract "選択肢の概要"

        **① AVP事前チェック方式（個別リソース取得→AVP判定）**

        - リソースをDBから取得
        - リソース属性をAVPに渡して認可チェック
        - 許可された場合のみレスポンスを返却

        **② DB条件付き取得方式（WHERE句で制限）**

        - JWTのユーザー属性をWHERE句に組み込んでDB取得時に絞り込み
        - AVPは使用せず、SQLレベルで制限
        - フィルタ条件はアプリケーションコードで実装

        **③ Cedar Query方式（複数リソースの一括判定）**

        - AVP の BatchIsAuthorized API を使用
        - 複数リソースに対して一括で認可チェック
        - フィルタ済みリソースのみ返却

    !!! abstract "比較表"

        | 比較項目 | ① AVP事前チェック | ② DB条件付き取得 | ③ Cedar Query | 最適 |
        |---------|----------------|------------|---------------|------|
        | **実装の容易さ** | 🟡 中程度<br>個別チェック実装 | ✅ 簡単<br>WHERE句追加 | ❌ 複雑<br>一括判定の実装 | ② |
        | **パフォーマンス** | ✅ 良好<br>必要なデータのみ取得 | ✅ 最良<br>DBで絞り込み | 🟡 中程度<br>API呼び出し増 | ② |
        | **セキュリティ** | ✅ 高い<br>AVPで統一的に管理 | 🟡 中程度<br>実装ミスのリスク | ✅ 高い<br>AVPで統一的に管理 | ①③ |
        | **権限の柔軟性** | ✅ 高い<br>Cedarで柔軟に定義 | 🟡 限定的<br>SQLで実装 | ✅ 高い<br>Cedarで柔軟に定義 | ①③ |
        | **保守性** | ✅ 高い<br>ポリシー集中管理 | 🟡 中程度<br>SQLに散在 | ✅ 高い<br>ポリシー集中管理 | ①③ |
        | **監査ログ** | ✅ 詳細<br>AVPログに記録 | ❌ なし<br>自前実装が必要 | ✅ 詳細<br>AVPログに記録 | ①③ |
        | **大量データ** | 🟡 要注意<br>N+1問題のリスク | ✅ 適している<br>DBで効率的に処理 | ❌ 不適<br>API呼び出し多数 | ② |

    !!! abstract "推奨する選択肢: ② DB条件付き取得方式（初期）→ ① AVP事前チェック方式（将来）"

        **選択理由**:

        **初期フェーズ（②を採用）**:
        - ✅ **実装速度**: WHERE句追加のみで迅速に実装可能
        - ✅ **パフォーマンス**: DBレベルで絞り込みが最も効率的
        - ✅ **シンプル**: 既存のDB設計・SQLノウハウを活用
        - 🟡 **権限変更時**: コード修正が必要（デプロイ必要）

        **将来フェーズ（①に移行）**:
        - ✅ **権限の柔軟性**: Cedarポリシーで複雑な条件も対応可能
        - ✅ **集中管理**: 権限ロジックがAVPに集約、保守性向上
        - ✅ **監査**: すべての認可判定がCloudWatch Logsに記録
        - 🟡 **段階的移行**: 重要エンドポイントから順次移行

        **③ を採用しない理由**:

        - ❌ **API呼び出し増**: リソース数に比例してAVP APIコールが増加（コスト・レイテンシ）
        - ❌ **実装複雑度**: BatchIsAuthorized の結果とリソースをマッピングする必要がある
        - ℹ️ 少数の重要リソースに対する詳細チェックには有効

    ??? example "推奨方式の実装詳細"

        ### フェーズ1: DB条件付き取得方式（初期実装）

        **実装例: 自分の部署の顧客のみ取得**

        ```typescript
        // BFF (Next.js API Route)
        export async function GET(req: Request) {
          const session = await getServerSession();
          const token = session.accessToken;

          // 1. エンドポイントレベルのAVPチェック（グループベース）
          const canAccess = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: { actionType: "Action", actionId: "listCustomers" },
            resource: { entityType: "APIEndpoint", entityId: "/api/customers" }
          });

          if (canAccess.decision !== 'ALLOW') {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 2. JWTから部署情報を取得
          const decoded = jwt.decode(token);
          const userDepartment = decoded['custom:department'];

          // 3. DBクエリで部署による絞り込み（ここがポイント）
          const customers = await db.query(
            'SELECT * FROM customers WHERE department = $1',
            [userDepartment]
          );

          return Response.json(customers);
        }
        ```

        **Cedar ポリシー（エンドポイントレベル）**:

        ```cedar
        // 営業グループは顧客一覧APIにアクセス可能
        permit(
          principal,
          action == Action::"listCustomers",
          resource
        ) when {
          principal.groups.contains("sales") ||
          principal.groups.contains("sales_manager")
        };
        ```

        **実装例: 自分が作成した注文のみ取得**

        ```typescript
        // 注文詳細取得
        export async function GET(req: Request, { params }: { params: { id: string } }) {
          const session = await getServerSession();
          const token = session.accessToken;
          const decoded = jwt.decode(token);
          const userId = decoded.sub;

          // エンドポイントレベルのチェック
          const canAccess = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: { actionType: "Action", actionId: "viewOrder" },
            resource: { entityType: "APIEndpoint", entityId: "/api/orders" }
          });

          if (canAccess.decision !== 'ALLOW') {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 所有者チェックをSQL WHERE句で実施
          const order = await db.query(
            'SELECT * FROM orders WHERE id = $1 AND created_by = $2',
            [params.id, userId]
          );

          if (!order) {
            return Response.json({ error: 'Not Found' }, { status: 404 });
          }

          return Response.json(order);
        }
        ```

        **メリット**:

        - WHERE句で効率的に絞り込み、不要なデータを取得しない
        - 実装がシンプルで既存の開発手法を活用できる
        - パフォーマンスが良好

        **デメリット**:

        - 権限ロジックがアプリケーションコードに散在
        - 権限変更時にコード修正・デプロイが必要
        - 監査ログは自前で実装する必要がある

        ---

        ### フェーズ2: AVP事前チェック方式（将来の移行先）

        **実装例: リソース属性を使った詳細チェック**

        ```typescript
        // 顧客削除（所有者 or 管理者のみ）
        export async function DELETE(req: Request, { params }: { params: { id: string } }) {
          const session = await getServerSession();
          const token = session.accessToken;

          // 1. リソースをDBから取得
          const customer = await db.getCustomer(params.id);

          if (!customer) {
            return Response.json({ error: 'Not Found' }, { status: 404 });
          }

          // 2. AVPでリソース属性を含めた詳細チェック
          const result = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: {
              actionType: "Action",
              actionId: "deleteCustomer"
            },
            resource: {
              entityType: "Customer",
              entityId: customer.id,
              attributes: {
                createdBy: customer.createdBy,
                department: customer.department,
                classification: customer.classification
              }
            }
          });

          if (result.decision !== 'ALLOW') {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 3. 削除実行
          await db.deleteCustomer(customer.id);
          return Response.json({ success: true });
        }
        ```

        **Cedar ポリシー（リソース属性チェック）**:

        ```cedar
        // 自分が作成した顧客のみ削除可能
        permit(
          principal,
          action == Action::"deleteCustomer",
          resource
        ) when {
          resource.createdBy == principal.userId
        };

        // 管理者は全ての顧客を削除可能
        permit(
          principal,
          action == Action::"deleteCustomer",
          resource
        ) when {
          principal.groups.contains("admin")
        };

        // 同じ部署の顧客のみ削除可能（営業マネージャー）
        permit(
          principal,
          action == Action::"deleteCustomer",
          resource
        ) when {
          principal.groups.contains("sales_manager") &&
          principal.department == resource.department
        };

        // 機密顧客は削除不可（例外規定）
        forbid(
          principal,
          action == Action::"deleteCustomer",
          resource
        ) when {
          resource.classification == "confidential" &&
          !principal.groups.contains("super_admin")
        };
        ```

        **メリット**:

        - 権限ロジックがCedarポリシーに集約され、保守性が向上
        - ポリシー更新のみで権限変更可能（デプロイ不要）
        - すべての判定がCloudWatch Logsに記録され、監査が容易

        **デメリット**:

        - リソース取得後のAVPチェックのため、若干のオーバーヘッド
        - 一覧取得APIでは全件取得後フィルタが必要（パフォーマンス懸念）

        ---

        ### 一覧取得APIでのハイブリッド実装

        大量データの一覧取得では、③と①を組み合わせる：

        ```typescript
        // 顧客一覧取得（ハイブリッド方式）
        export async function GET(req: Request) {
          const session = await getServerSession();
          const token = session.accessToken;
          const decoded = jwt.decode(token);

          // エンドポイントレベルチェック
          const canAccess = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: { actionType: "Action", actionId: "listCustomers" },
            resource: { entityType: "APIEndpoint", entityId: "/api/customers" }
          });

          if (canAccess.decision !== 'ALLOW') {
            return Response.json({ error: 'Forbidden' }, { status: 403 });
          }

          // 一覧取得は引き続きDB条件付き取得（パフォーマンス優先）
          let query = 'SELECT * FROM customers WHERE 1=1';
          const params = [];

          // 管理者以外は部署でフィルタ
          if (!decoded['cognito:groups']?.includes('admin')) {
            query += ' AND department = $1';
            params.push(decoded['custom:department']);
          }

          const customers = await db.query(query, params);
          return Response.json(customers);
        }
        ```

        **設計方針**:

        | 操作種別 | 推奨方式 | 理由 |
        |---------|---------|------|
        | **一覧取得** | ② DB条件付き取得 | 大量データを効率的に処理 |
        | **個別取得** | ② DB条件付き取得<br>（WHERE句で所有者チェック） | シンプルで高速 |
        | **更新・削除** | ① AVP事前チェック | 監査ログが重要、ポリシーで柔軟に制御 |
        | **承認・重要操作** | ① AVP事前チェック | 複雑な条件、厳密な監査が必要 |

    ??? info "段階的移行戦略"
        **初期（フェーズ1）**:

        - すべてのエンドポイントで② DB条件付き取得方式を採用
        - エンドポイントレベルのAVPチェックでグループベース制御
        - 迅速にリリース、基本的な権限制御を実現

        **中期（フェーズ2）**:

        - 削除・承認などの重要操作を① AVP事前チェックに移行
        - リソース属性を使った詳細なCedarポリシーを作成
        - 監査ログの整備

        **長期（フェーズ3）**:

        - 一覧取得以外のエンドポイントを順次①に移行
        - 権限ロジックをCedarポリシーに集約
        - アプリケーションコードから権限ロジックを分離

    ??? warning "N+1問題への注意"
        一覧取得APIで各リソースに対してAVPチェックを行うと、N+1問題が発生します：

        ```typescript
        // ❌ これはやらない（N+1問題）
        const customers = await db.getAllCustomers();
        const filtered = [];

        for (const customer of customers) {
          const allowed = await avp.isAuthorizedWithToken({
            identityToken: token,
            action: { actionType: "Action", actionId: "viewCustomer" },
            resource: {
              entityType: "Customer",
              entityId: customer.id,
              attributes: { department: customer.department }
            }
          });

          if (allowed.decision === 'ALLOW') {
            filtered.push(customer);
          }
        }
        ```

        **対策**:

        - 一覧取得は③ DB条件付き取得方式を継続使用
        - または、BatchIsAuthorized APIを使用（ただしコスト増）

??? question "🛡️ 検討4: 認証・認可チェックの実施箇所"

    !!! tip "検討ポイント"

        **BFFとAPIのどちらで認証・認可チェックを実施するか？**

        - BFF集中方式: BFFのみで検証、APIは内部通信として信頼
        - 多層防御方式: BFFとAPI両方で独立検証（セキュリティ強化）
        - 責務分離とパフォーマンスのトレードオフ

    !!! abstract "選択肢の概要"

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

    !!! abstract "比較表"

        | 比較項目 | ① BFF 集中 | ② 多層防御 | ③ Gateway 集中 | 最適 |
        |---------|-----------|-----------|---------------|------|
        | **早期リジェクション** | ✅ BFF で拒否 | ✅ BFF で拒否 | 🟡 Gateway で拒否<br>（BFF経由後） | ①② |
        | **多層防御** | ❌ API が無防備<br>（BFF信頼前提） | ✅ BFF・API で<br>二重チェック | ✅ Gateway・API で<br>二重チェック可能 | ②③ |
        | **実装の容易さ** | ✅ BFF のみ実装 | 🟡 BFF・API 両方に実装 | ❌ Gateway + Lambda<br>追加実装が必要 | ① |
        | **API 単体利用** | ❌ 不可<br>（BFF 必須） | ✅ 可能<br>（モバイルアプリ等） | ✅ 可能<br>（Gateway 経由） | ②③ |
        | **ページレベル制御** | ✅ BFF で SSR 制御 | ✅ BFF で SSR 制御 | ✅ BFF で SSR 制御 | ①②③ |
        | **運用コスト** | ✅ 最小限 | 🟡 やや増加 | ❌ Gateway 管理が<br>追加で必要 | ① |
        | **責務の明確さ** | 🟡 BFF に集中 | ✅ 明確に分離 | 🟡 Gateway が集中 | ② |

    !!! abstract "推奨する選択肢: ② 多層防御方式（BFF・API 両方でチェック）"

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

??? question "🔗 検討5: SSO 実装方針"

    !!! tip "検討ポイント"

        **複数サービス間でシームレスなログイン体験をどう実現するか？**

        - Cookie共有方式: 同一ドメイン配下でセッション共有（ドメイン制約あり）
        - サイレント認証方式: IdPセッション活用のリダイレクトSSO（ドメイン自由）
        - 各方式のドメイン構成要件と実装複雑度

    !!! abstract "選択肢の概要"

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

    !!! abstract "比較表"

        | 比較項目 | ① Cookie共有 | ② サイレント認証 | ③ トークンGW | 最適 |
        |---------|-------------|----------------|-------------|------|
        | **IdP セッション活用** | 🟡 初回のみ | ✅ 毎回活用 | 🟡 初回のみ | ② |
        | **ドメイン制約** | ❌ サブドメイン必須<br>（`.example.com`） | ✅ 制約なし | ✅ 制約なし | ②③ |
        | **実装の容易さ** | 🟡 Cookie設定の調整 | ✅ NextAuth.js標準<br>（`prompt=none`追加のみ） | ❌ GW実装が必要 | ② |
        | **サービス間遷移** | ✅ 即座（0秒） | 🟡 サイレント認証経由<br>（リダイレクト発生） | 🟡 トークン引き回し<br>（リダイレクト発生） | ① |
        | **セキュリティ** | 🟡 Cookie範囲が広い | ✅ ホスト限定Cookie | ✅ トークン分離 | ②③ |
        | **スケーラビリティ** | 🟡 サブドメイン制約 | ✅ サービス追加容易 | 🟡 GW が SPOF | ② |
        | **運用コスト** | ✅ 最小限 | ✅ 最小限 | ❌ GW運用が必要 | ①② |

    !!! abstract "推奨する選択肢: ✅ ② サイレント認証方式（`prompt=none`）"

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
    | **検討1: ブラウザ⇔BFF認証情報管理** | HttpOnly Cookie + サーバー側セッション | XSS対策、トークン非露出、NextAuth.js標準 |
    | **検討2: 認可方式** | AVP による実行時権限チェック | JWT サイズ最小化、ポリシー即時反映、柔軟性、監査ログ |
    | **検討3: リソース属性制御** | DB条件付き取得（初期）→ AVP事前チェック（将来） | パフォーマンス優先、段階的にAVPへ移行 |
    | **検討4: 認証・認可実施箇所** | BFF・API 多層防御 | 早期リジェクション、多層防御、柔軟性 |
    | **検討5: SSO 実装** | サイレント認証（prompt=none） | IdP セッション活用、ドメイン制約なし、実装軽微 |

??? success "この構成の利点"

    ✅ **セキュリティ**

    - JWT サイズ最小化（`userId` + `groups` のみ）で詳細権限は AVP で管理
    - BFF と API の多層防御、トークンをブラウザに露出させない設計
    - 権限変更・ログアウト時のセッション即時無効化
    - CloudWatch Logs による監査証跡

    ✅ **パフォーマンス・コスト**

    - Redis キャッシュによる認可チェックの高速化
    - AVP の従量課金は非常に安価（200 DAU × 200 API コール/日で約 $6/月）
    - 水平スケールアウト対応

    ✅ **開発・運用効率**

    - NextAuth.js 活用による実装の簡潔さ
    - BFF での早期リジェクション、責務の明確な分離
    - トークン更新・CSRF 対策・セッション管理の自動化

    ✅ **拡張性**

    - 新規サービス追加が容易（同じ認証フローを適用）
    - Cedar ポリシー更新のみで権限要件変更に対応
