# OAuth 2.0 / OpenID Connect の基礎

> **🔐 認証・認可 > OAuth 2.0 / OIDC**
> 本ドキュメントでは、OAuth 2.0とOpenID Connectの基礎知識を解説します。プロトコルの目的と違い、各種フローの選択基準、トークンの種類、重要なセキュリティ機能について詳しく説明します。

## 前提知識

- **[概要](auth-basics.md)** - 認証・認可の基礎を先に読むことを推奨

## 関連ドキュメント

- 次のステップ: **[セキュリティ対策](security-practices.md)** - 脅威と対策
- 実装: **[実装ガイド](implementation-guide.md)** - Next.js + Spring Boot + Cognito
- 詳細: **[包括的リファレンス](authentication-guide.md)** - 最新セキュリティ技術

---

## 目次

1. [OAuth 2.0とは](#oauth-20とは)
2. [OpenID Connect(OIDC)とは](#openid-connectoidcとは)
3. [フローの種類と選択](#フローの種類と選択)
4. [トークンの種類](#トークンの種類)
5. [重要なセキュリティ機能](#重要なセキュリティ機能)

---

## OAuth 2.0とは

### 概要

**OAuth 2.0は「認可」のための標準プロトコル**です(認証ではありません)。

- 策定: 2012年(RFC 6749)
- 最新: OAuth 2.1(ドラフト、2025年標準化予定)
- 目的: サードパーティアプリに限定的なアクセス権を与える

### 解決する問題

**悪い例: パスワードを共有**

```
ユーザー → 写真共有アプリにGoogleのパスワードを渡す
         → アプリがGoogleの全データにアクセス可能
```

**良い例: OAuth 2.0**

```
ユーザー → Googleで認証
         → 「写真の閲覧のみ」の権限(トークン)を発行
         → アプリは限定的なアクセスのみ可能
```

### 基本的な流れ

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Client as クライアントアプリ
    participant AuthServer as 認可サーバー
    participant API as リソースサーバー(API)

    User->>Client: 1. ログインボタンをクリック
    Client->>AuthServer: 2. 認可リクエスト(redirect)
    AuthServer->>User: 3. ログイン画面表示
    User->>AuthServer: 4. 認証情報入力
    AuthServer->>User: 5. 権限許可画面表示
    User->>AuthServer: 6. 許可
    AuthServer->>Client: 7. 認可コード発行(redirect)
    Client->>AuthServer: 8. 認可コード + クライアント認証情報
    AuthServer->>Client: 9. アクセストークン発行
    Client->>API: 10. アクセストークン付きリクエスト
    API->>Client: 11. データ返却
```

---

## OpenID Connect(OIDC)とは

### 概要

**OpenID Connectは「認証」のためのプロトコル**で、OAuth 2.0の上に構築されています。

- 策定: 2014年
- 目的: OAuth 2.0に認証機能を追加
- 提供: ユーザー情報(ID、名前、メールアドレス等)

### OAuth 2.0との違い

| 項目 | OAuth 2.0 | OpenID Connect |
|------|----------|----------------|
| 目的 | **認可**(権限付与) | **認証**(本人確認) |
| 主なトークン | アクセストークン | **IDトークン** + アクセストークン |
| 取得情報 | リソースへのアクセス権 | **ユーザー情報** |
| 用途 | API アクセス | ログイン、ユーザー情報取得 |

### IDトークンの中身(JWT形式)

```json
{
  "iss": "https://cognito-idp.ap-northeast-1.amazonaws.com/ap-northeast-1_xxxxx",
  "sub": "12345678-1234-1234-1234-123456789012",
  "aud": "your-client-id",
  "exp": 1616239022,
  "iat": 1616235422,
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "cognito:groups": ["Admin", "Users"]
}
```

- **iss** (issuer): 発行者
- **sub** (subject): ユーザーID
- **aud** (audience): 想定受信者(クライアントID)
- **exp** (expiration): 有効期限
- **iat** (issued at): 発行時刻

---

## フローの種類と選択

### OAuth 2.0 / OIDC の主要フロー

#### 1. Authorization Code Flow(認可コードフロー) ✅ 推奨

**最も安全なフロー。すべての状況で推奨。**

```mermaid
sequenceDiagram
    participant Browser
    participant Client
    participant AuthServer

    Browser->>AuthServer: 1. 認可リクエスト(redirect)
    AuthServer->>Browser: 2. 認可コード(redirect)
    Browser->>Client: 3. 認可コードを渡す
    Client->>AuthServer: 4. 認可コード + クライアントシークレット
    AuthServer->>Client: 5. トークン発行
```

**特徴**:

- ✅ トークンがブラウザに露出しない
- ✅ クライアントシークレットでサーバー認証
- ✅ 最もセキュア

**適用**: サーバーサイドアプリ(Next.js SSR等)

---

#### 2. Authorization Code + PKCE ✅ 2025年標準

**SPAやモバイルアプリで必須。OAuth 2.1の標準フロー。**

```mermaid
sequenceDiagram
    participant SPA
    participant AuthServer

    SPA->>SPA: code_verifier生成
    SPA->>SPA: code_challenge = hash(code_verifier)
    SPA->>AuthServer: 1. 認可リクエスト(code_challenge)
    AuthServer->>SPA: 2. 認可コード
    SPA->>AuthServer: 3. 認可コード + code_verifier
    AuthServer->>AuthServer: 4. code_verifier検証
    AuthServer->>SPA: 5. トークン発行
```

**PKCE(Proof Key for Code Exchange)の仕組み**:

1. ランダムな`code_verifier`を生成
2. `code_challenge = SHA256(code_verifier)`を計算
3. 認可リクエストに`code_challenge`を含める
4. トークン取得時に`code_verifier`を送信
5. サーバーが検証: `SHA256(code_verifier) == code_challenge`

**特徴**:

- ✅ クライアントシークレット不要
- ✅ 認可コード横取り攻撃を防止
- ✅ パブリッククライアント(SPA/Mobile)でも安全

**適用**: SPA、モバイルアプリ、すべての環境

---

#### 3. Implicit Flow ❌ 廃止

**⚠️ OAuth 2.1で廃止。使用禁止。**

```mermaid
sequenceDiagram
    participant SPA
    participant AuthServer

    SPA->>AuthServer: 1. 認可リクエスト
    AuthServer->>SPA: 2. トークンを直接返却(URLフラグメント)
```

**問題点**:

- ❌ トークンがブラウザ履歴に残る
- ❌ JavaScriptで取得(XSS脆弱)
- ❌ リフレッシュトークンが使えない

**代替**: Authorization Code + PKCE

---

#### 4. Client Credentials Flow

**サーバー間通信(M2M)専用。**

```mermaid
sequenceDiagram
    participant ServiceA
    participant AuthServer
    participant ServiceB

    ServiceA->>AuthServer: 1. クライアント認証情報
    AuthServer->>ServiceA: 2. アクセストークン
    ServiceA->>ServiceB: 3. トークン付きリクエスト
```

**特徴**:

- ユーザーが介在しない
- サーバー同士の通信

**適用**: バッチ処理、マイクロサービス間通信

---

#### 5. Device Authorization Grant(デバイスコードフロー)

**入力が困難なデバイス用。**

```mermaid
sequenceDiagram
    participant TV
    participant AuthServer
    participant Browser

    TV->>AuthServer: 1. デバイスコード要求
    AuthServer->>TV: 2. デバイスコード + URL
    TV->>TV: 3. QRコード/URLを画面表示
    Browser->>AuthServer: 4. URLにアクセスしてログイン
    TV->>AuthServer: 5. ポーリング(トークン取得)
```

**適用**: スマートTV、IoTデバイス、CLI

---

### フロー選択チャート(2025年版)

```mermaid
graph TD
    Start[認証フローを選ぶ] --> Q1{ユーザーがいる?}
    Q1 -->|いいえ| M2M[Client Credentials]
    Q1 -->|はい| Q2{デバイスの種類は?}
    Q2 -->|Webブラウザ| Q3{実装方式は?}
    Q3 -->|サーバーサイド| Code[Authorization Code]
    Q3 -->|SPA| PKCE[Authorization Code + PKCE]
    Q2 -->|モバイルアプリ| PKCE
    Q2 -->|TV/IoT| Device[Device Authorization Grant]
```

---

## トークンの種類

### 1. アクセストークン(Access Token)

**用途**: APIアクセスの認可

```http
GET /api/users/profile HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...
```

**特徴**:

- 短命(15分〜1時間)
- スコープ(権限)を含む
- JWT または Opaque Token

**検証項目**:

- ✅ 署名の正当性
- ✅ 有効期限(exp)
- ✅ 発行者(iss)
- ✅ 対象者(aud)
- ✅ スコープ

---

### 2. リフレッシュトークン(Refresh Token)

**用途**: アクセストークンの再発行

```http
POST /oauth2/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=xyz123...&
client_id=your-client-id
```

**特徴**:

- 長命(7〜30日)
- 1回限り使用(ローテーション推奨)
- 不透明トークン

**セキュリティ対策**:

- ✅ HttpOnly Cookie に保存
- ✅ 使用後は新しいトークンと交換
- ✅ 再利用検知で全トークン無効化

---

### 3. IDトークン(ID Token) - OIDC専用

**用途**: ユーザー情報の取得、クライアント側での認証確認

```javascript
// IDトークンからユーザー情報を取得
const decoded = jwt.decode(idToken);
console.log(decoded.email);  // "user@example.com"
console.log(decoded.name);   // "John Doe"
```

**特徴**:

- JWT形式(必須)
- ユーザー情報を含む
- API認証には**使用しない**(アクセストークンを使用)

**検証項目**:

- ✅ 署名の正当性
- ✅ 有効期限(exp)
- ✅ 発行者(iss)
- ✅ 対象者(aud) - クライアントID
- ✅ nonce(リプレイ攻撃対策)

---

### トークンの使い分け

| トークン | 用途 | 送信先 | 有効期限 | 形式 |
|---------|------|--------|---------|------|
| アクセストークン | API呼び出し | リソースサーバー | 短い(15分〜1時間) | JWT/Opaque |
| リフレッシュトークン | トークン再発行 | 認可サーバー | 長い(7〜30日) | Opaque |
| IDトークン | ユーザー情報取得 | クライアント | 短い(1時間) | JWT |

---

## 重要なセキュリティ機能

### 1. PKCE(Proof Key for Code Exchange)

**認可コード横取り攻撃を防ぐ。**

#### 攻撃シナリオ(PKCEなし)

```
1. 正規アプリが認可コードを受け取る
2. 悪意あるアプリがURLから認可コードを盗む
3. 悪意あるアプリがトークンを取得
```

#### PKCE による防止

```
1. 認可リクエスト時: code_challenge を送信
2. トークン取得時: code_verifier を送信
3. サーバーが検証: hash(code_verifier) == code_challenge
4. 一致しない場合はトークン発行拒否
```

**必須環境**: SPA、モバイルアプリ、OAuth 2.1準拠の全環境

---

### 2. state パラメータ

**CSRF攻撃を防ぐ。**

#### 攻撃シナリオ(stateなし)

```
1. 攻撃者が自分のアカウントで認可フローを開始
2. 認可コードを含むコールバックURLを被害者に送信
3. 被害者がクリック → 攻撃者のアカウントにログイン
4. 被害者の操作が攻撃者のアカウントに記録される
```

#### state による防止

```javascript
// 認可リクエスト前
const state = generateRandomString();
sessionStorage.setItem('oauth_state', state);
location.href = `${authUrl}?state=${state}&...`;

// コールバック時
const returnedState = new URLSearchParams(location.search).get('state');
const savedState = sessionStorage.getItem('oauth_state');

if (returnedState !== savedState) {
  throw new Error('State mismatch - possible CSRF attack');
}
```

**必須環境**: すべての OAuth/OIDC 実装

---

### 3. nonce パラメータ(OIDC)

**IDトークンのリプレイアタック を防ぐ。**

```javascript
// 認可リクエスト前
const nonce = generateRandomString();
sessionStorage.setItem('oidc_nonce', nonce);
location.href = `${authUrl}?nonce=${nonce}&...`;

// IDトークン検証時
const decoded = jwt.decode(idToken);
const savedNonce = sessionStorage.getItem('oidc_nonce');

if (decoded.nonce !== savedNonce) {
  throw new Error('Nonce mismatch - possible replay attack');
}
```

**必須環境**: OpenID Connect 使用時

---

### 4. リダイレクトURIの厳格な検証

**オープンリダイレクタ攻撃を防ぐ。**

#### 悪い例

```javascript
// ❌ ワイルドカード許可
allowed_redirect_uris: ["https://example.com/*"]

// 攻撃者が悪用
https://example.com/evil-redirect?target=https://attacker.com
```

#### 良い例

```javascript
// ✅ 完全一致のみ
allowed_redirect_uris: [
  "https://example.com/callback",
  "http://localhost:3000/api/auth/callback/cognito"
]
```

**ルール**:

- ✅ 完全一致のみ許可
- ❌ ワイルドカード禁止
- ❌ 部分一致禁止

---

### 5. スコープの最小権限

**必要最小限の権限のみ要求。**

#### 悪い例

```javascript
// ❌ 過剰な権限
scope: "openid profile email admin:all"
```

#### 良い例

```javascript
// ✅ 必要最小限
scope: "openid email"  // ログインとメールアドレスのみ
```

---

## OAuth 2.0 vs OAuth 2.1

### 主な変更点(2025年)

| 項目 | OAuth 2.0 | OAuth 2.1 |
|------|----------|----------|
| 推奨フロー | 複数のフロー | **Authorization Code + PKCE** |
| Implicit Flow | 許可 | **廃止** |
| ROPC | 許可 | **廃止** |
| PKCE | オプション | **必須** |
| リダイレクトURI | 部分一致も可 | **完全一致必須** |
| state | 推奨 | **必須** |
| リフレッシュトークン | 再利用可能 | **ローテーション推奨** |

---

## まとめ

### 覚えておくべき重要ポイント

1. **OAuth 2.0 = 認可**、**OIDC = 認証**
2. **2025年の標準フロー = Authorization Code + PKCE**
3. **Implicit Flow は使用禁止**
4. **トークンは3種類**: アクセス、リフレッシュ、ID
5. **必須セキュリティ**: PKCE、state、nonce、リダイレクトURI厳格検証

### 次のステップ

基礎知識を理解したら、以下のドキュメントに進んでください:

1. **[セキュリティ対策](security-practices.md)** - 脅威と対策の詳細
2. **[実装ガイド](implementation-guide.md)** - Next.js + Spring Boot + Cognito の実装

---

**参考資料**:

- [OAuth 2.0 RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)
- [OAuth 2.1 Draft](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-10)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [PKCE RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)

**最終更新**: 2025年10月15日
**対象読者**: 認証機能の実装を学ぶ開発者(初級〜中級)
