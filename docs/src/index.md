# 設計ドキュメント

このサイトでは、プロジェクトの設計に関するドキュメントを提供しています。

## 📚 ドキュメント一覧

### 認証・認可

Webアプリケーションにおける認証・認可に関する包括的なドキュメント群です。

#### 基礎から学ぶ

##### 1. **[概要](auth-basics.md)**

- 認証と認可の違い
- セッションとトークンの基礎
- 登場人物と役割(OAuth/OIDC)

##### 2. **[OAuth 2.0 / OpenID Connect](oauth-oidc-basics.md)**

- プロトコルの詳細
- フローの種類と選択基準
- トークンの種類と使い分け
- セキュリティ機能(PKCE、state、nonce)

##### 3. **[セキュリティ対策](security-practices.md)**

- 典型的な脅威と対策
- トークン管理のベストプラクティス
- 2025年の最新動向

#### 実装

##### 1. **[実装ガイド](implementation-guide.md)**

- Next.js + Spring Boot + AWS Cognito
- BFFパターンによる認証実装
- 通信フローの詳細
- コード例とチェックリスト

#### リファレンス

##### 1. **[包括的リファレンス](authentication-guide.md)**

- 各認証方式の詳細解説
- アーキテクチャパターン
- 最新セキュリティ技術(PAR、DPoP、mTLS等)
- レガシーシステムのモダナイゼーション

---

## 💡 学習の進め方

### 初学者向け

1. [概要](auth-basics.md) で基本概念を理解
2. [OAuth 2.0 / OIDC](oauth-oidc-basics.md) でプロトコルを学習
3. [セキュリティ対策](security-practices.md) で脅威と対策を把握
4. [実装ガイド](implementation-guide.md) で実装方法を習得

### 実装担当者向け

- [実装ガイド](implementation-guide.md) を中心に参照
- 必要に応じて [包括的リファレンス](authentication-guide.md) で詳細を確認
- [セキュリティ対策](security-practices.md) でチェックリストを活用

### アーキテクト向け

- [包括的リファレンス](authentication-guide.md) でアーキテクチャパターンを検討
- [OAuth 2.0 / OIDC](oauth-oidc-basics.md) で標準仕様を確認
- [セキュリティ対策](security-practices.md) で最新動向をキャッチアップ

---

## ドキュメントについて

このドキュメントは MkDocs を使用して生成されています。

### ローカルでの閲覧方法

詳しくは、リポジトリの README を参照してください。
