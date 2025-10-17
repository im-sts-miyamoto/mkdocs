# 認証・認可の基礎知識

> **📚 認証・認可 > 概要**
> 本ドキュメントでは、認証機能を理解するための基礎知識をまとめています。認証と認可の違い、セッションとトークンの比較、OAuth/OIDCにおける登場人物など、認証機能を学ぶ上での土台となる概念を解説します。

## 関連ドキュメント

- 次のステップ: **[OAuth 2.0 / OpenID Connect](oauth-oidc-basics.md)** - プロトコルの詳細
- 実装: **[実装ガイド](implementation-guide.md)** - Next.js + Spring Boot + Cognito
- セキュリティ: **[セキュリティ対策](security-practices.md)** - 脅威と対策

---

## 目次

1. [認証と認可の違い](#認証と認可の違い)
2. [認証の3要素](#認証の3要素)
3. [セッションとトークンの違い](#セッションとトークンの違い)
4. [主要な登場人物](#主要な登場人物oauthoidc)

---

## 認証と認可の違い

### 認証(Authentication)とは

**「あなたは誰ですか?」**という問いに答えるプロセス。

- 目的: 本人確認
- 例: ログイン、パスワード入力
- 英語: Authentication (AuthN)

### 認可(Authorization)とは

**「あなたは何ができますか?」**という問いに答えるプロセス。

- 目的: 権限確認
- 例: 管理者画面へのアクセス、ファイルの編集権限
- 英語: Authorization (AuthZ)

### 比較表

| 項目 | 認証(Authentication) | 認可(Authorization) |
|------|---------------------|-------------------|
| 問い | 「あなたは誰?」 | 「あなたは何ができる?」 |
| 目的 | 身元確認 | 権限確認 |
| タイミング | アクセス時(最初) | リソース操作時(都度) |
| 例 | ログイン、パスワード入力 | 管理者画面アクセス、データ編集 |
| 失敗時 | 401 Unauthorized | 403 Forbidden |

### 実際の流れ

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリケーション
    participant Resource as リソース

    User->>App: 1. ログイン(認証)
    App->>User: 認証成功
    User->>App: 2. 管理画面にアクセス
    App->>App: 3. 権限チェック(認可)
    App->>Resource: 4. リソース取得
    Resource->>User: データ表示
```

---

## 認証の3要素

多要素認証(MFA)を理解するための基礎。

### 1. 知識要素(Something You Know)

#### 「あなたが知っていること」

- パスワード
- PINコード
- 秘密の質問

**メリット**: 実装が簡単
**デメリット**: 忘れる、漏えいのリスク

### 2. 所持要素(Something You Have)

#### 「あなたが持っているもの」

- スマートフォン
- セキュリティキー
- ワンタイムパスワード(OTP)トークン
- ICカード

**メリット**: 物理的な盗難が必要
**デメリット**: 紛失、破損のリスク

### 3. 生体要素(Something You Are)

#### 「あなた自身」

- 指紋
- 顔認証
- 虹彩認証
- 声紋

**メリット**: 本人以外使用困難
**デメリット**: 偽装の可能性、プライバシー懸念

### 多要素認証(MFA)

**2つ以上の要素を組み合わせる**ことでセキュリティを強化。

#### 良い組み合わせ例

- ✅ パスワード(知識) + スマホアプリのOTP(所持)
- ✅ パスワード(知識) + 指紋(生体)
- ✅ ICカード(所持) + PIN(知識)

#### 悪い組み合わせ例

- ❌ パスワード + 秘密の質問 (両方とも知識要素)
- ❌ 指紋 + 顔認証 (両方とも生体要素)

---

## セッションとトークンの違い

### セッションベース認証

```mermaid
graph LR
    Browser[ブラウザ] -->|Cookie: session_id| Server[サーバー]
    Server -->|セッション情報取得| SessionStore[(セッションストア)]
```

#### セッションの特徴

- サーバー側でセッション情報を保持
- クライアントにはセッションIDのみ送信
- サーバーに状態を持つ(Stateful)

#### セッションのメリット

- ✅ サーバー側で即座に無効化可能
- ✅ セッション情報をサーバーで完全管理
- ✅ CSRF対策が可能

#### セッションのデメリット

- ❌ サーバーにストレージが必要
- ❌ 水平スケーリング時にセッション共有が必要
- ❌ モバイルアプリでのCookie管理が複雑

---

### トークンベース認証(JWT)

```mermaid
graph LR
    Browser[ブラウザ] -->|Authorization: Bearer xxx| Server[サーバー]
    Server -->|トークン検証| Server
```

#### トークンの特徴

- 署名付きトークンをクライアント側で保持
- サーバーは状態を持たない(Stateless)
- トークン自体にユーザー情報を含む

#### JWTの構造

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOiIxMjM0NSIsIm5hbWUiOiJKb2huIERvZSIsImV4cCI6MTYxNjIzOTAyMn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Header.Payload.Signature
```

- **Header**: アルゴリズム情報
- **Payload**: ユーザー情報、有効期限
- **Signature**: 改ざん検知用の署名

#### トークンのメリット

- ✅ ステートレス(サーバー側で状態不要)
- ✅ 水平スケーリングが容易
- ✅ マイクロサービス間で共有可能
- ✅ モバイルアプリと相性が良い

#### トークンのデメリット

- ❌ トークンの即座な無効化が困難
- ❌ トークンサイズが大きい
- ❌ XSS攻撃でトークンが盗まれるリスク

---

### セッションとトークンの比較

| 項目 | セッション | トークン(JWT) |
|------|----------|--------------|
| 状態 | Stateful | Stateless |
| 保存場所(サーバー) | 必要 | 不要 |
| 保存場所(クライアント) | Cookie(ID のみ) | LocalStorage/Cookie(全情報) |
| 無効化 | 即座に可能 | 困難(有効期限まで有効) |
| スケーリング | セッション共有必要 | 容易 |
| サイズ | 小(IDのみ) | 大(全情報) |
| 適用シーン | モノリス、管理画面 | SPA、マイクロサービス |

---

## 主要な登場人物(OAuth/OIDC)

### 4つの役割

```mermaid
graph TB
    User[👤 ユーザー<br/>リソースオーナー]
    Client[💻 クライアント<br/>フロントエンドアプリ]
    AuthServer[🔐 認可サーバー/IdP<br/>AWS Cognito等]
    ResourceServer[🗄️ リソースサーバー<br/>APIサーバー]

    User -->|1. ログイン| AuthServer
    AuthServer -->|2. トークン発行| Client
    Client -->|3. トークン付きリクエスト| ResourceServer
    ResourceServer -->|4. データ返却| Client
```

#### 1. ユーザー(リソースオーナー)

- エンドユーザー、実際にログインする人
- リソース(データ)の所有者
- 例: あなた

#### 2. クライアント

- ユーザーが使用するアプリケーション
- 認証・認可を要求する側
- 例: Next.js フロントエンド、React SPA、モバイルアプリ

#### 3. 認可サーバー / IdP (Identity Provider)

- 認証を実施し、トークンを発行
- ユーザー情報を管理
- 例: AWS Cognito, Auth0, Google, Microsoft Azure AD

#### 4. リソースサーバー(API)

- 保護されたリソースを提供
- トークンを検証してデータを返却
- 例: Spring Boot API, Node.js API

---

### 実際のシステムでの対応

**Next.js + Spring Boot + AWS Cognito の場合:**

| 役割 | 対応するコンポーネント |
|------|---------------------|
| ユーザー | ブラウザを操作する人 |
| クライアント | Next.js アプリケーション |
| 認可サーバー/IdP | **AWS Cognito** |
| リソースサーバー | **Spring Boot API** |

---

## 用語集

### OAuth 2.0関連

| 用語 | 説明 |
|------|------|
| **スコープ(Scope)** | アクセス権限の範囲(例: `read:profile`, `write:posts`) |
| **認可コード(Authorization Code)** | トークン取得のための一時的なコード |
| **アクセストークン** | API呼び出しに使用する短命のトークン |
| **リフレッシュトークン** | アクセストークン再発行用の長命トークン |
| **IDトークン(OIDC)** | ユーザー情報を含むトークン |

### セキュリティ関連

| 用語 | 説明 |
|------|------|
| **PKCE** | Proof Key for Code Exchange、認可コード横取り攻撃を防ぐ |
| **CSRF** | Cross-Site Request Forgery、クロスサイトリクエストフォージェリ |
| **XSS** | Cross-Site Scripting、クロスサイトスクリプティング |
| **state** | CSRF対策用のランダム文字列 |
| **nonce** | リプレイアタック対策用のランダム文字列 |

### Cookie属性

| 属性 | 説明 |
|------|------|
| **HttpOnly** | JavaScriptからアクセス不可(XSS対策) |
| **Secure** | HTTPS通信でのみ送信 |
| **SameSite** | クロスサイトリクエストでの送信制御(CSRF対策) |

---

## 次のステップ

基礎知識を理解したら、以下のドキュメントに進んでください:

1. **[OAuth 2.0 / OpenID Connect](oauth-oidc-basics.md)** - 認証プロトコルの詳細
2. **[セキュリティ対策](security-practices.md)** - 脅威と対策
3. **[実装ガイド](implementation-guide.md)** - 実際の実装方法

---

**最終更新**: 2025年10月15日
**対象読者**: 認証機能の実装を学ぶ開発者(初級〜中級)
