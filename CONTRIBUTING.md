# コントリビューションガイド

このプロジェクトへの貢献を検討いただきありがとうございます。このドキュメントでは、開発環境のセットアップ方法やデプロイ方法について説明します。

## 開発環境のセットアップ

### セットアップ（Dev Container）

#### 必要な環境

- **Docker**
- **Visual Studio Code**
- **Dev Containers 拡張機能**

> **Note**: Windows PCの場合は、WSL2にUbuntu環境をセットアップし、その上でDockerをインストールして使用してください。

#### 起動手順

1. リポジトリをクローン

   ```bash
   git clone git@github.com:im-sts-miyamoto/mkdocs.git
   cd mkdocs
   ```

2. VS Code で開く

   ```bash
   code .
   ```

3. コマンドパレット（`Ctrl+Shift+P` または `Cmd+Shift+P`）を開き、
   `Dev Containers: Reopen in Container` を選択

4. コンテナ起動後、ターミナルで以下を実行

   ```bash
   cd docs
   mkdocs serve -a 0.0.0.0:8000
   ```

5. ブラウザで `http://localhost:8000/` にアクセス

### ローカル環境での起動（Dev Container不使用）

> **Note**: この方法では、Python 3.x、pip、および必要なシステムパッケージが事前にインストールされている必要があります。これらのセットアップは各自で行ってください。

```bash
cd docs
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

ブラウザで `http://127.0.0.1:8000/` にアクセスしてください。

## GitHub Pagesへのデプロイ

このプロジェクトはGitHub Actionsを使って自動的にGitHub Pagesにデプロイされます。

### 公開URL

- **ドキュメントサイト**: <https://im-sts-miyamoto.github.io/mkdocs/>

### デプロイ方法

1. `main` ブランチにプッシュすると自動的にデプロイされます
2. 手動でデプロイする場合:
   - GitHubリポジトリの「Actions」タブを開く
   - 「Deploy MkDocs to GitHub Pages」ワークフローを選択
   - 「Run workflow」ボタンをクリック

### 初回セットアップ（リポジトリ管理者のみ）

1. GitHubリポジトリの「Settings」→「Pages」を開く
2. 「Source」で「Deploy from a branch」を選択
3. 「Branch」で `gh-pages` ブランチと `/ (root)` を選択
4. 「Save」をクリック

※ 初回のGitHub Actionsワークフロー実行後、`gh-pages` ブランチが自動作成されます。

## プロジェクト構成

```text
/workspaces/work/
├── .devcontainer/          # Dev Container設定
├── .github/                # GitHub Actions ワークフロー
├── docs/                   # ドキュメントルート
│   ├── mkdocs.yml          # MkDocs設定ファイル
│   ├── requirements.txt    # Python依存パッケージ
│   ├── PRESENTATION_GUIDE.md  # プレゼンテーションガイド
│   └── src/                # ドキュメントソース
│       ├── index.md        # トップページ
│       ├── auth-basics.md  # 認証の基礎
│       ├── oauth-oidc-basics.md  # OAuth/OIDC基礎
│       ├── authentication-guide.md  # 認証ガイド
│       ├── implementation-guide.md  # 実装ガイド
│       └── security-practices.md   # セキュリティプラクティス
├── site/                   # ビルド済み静的サイト（自動生成）
├── CONTRIBUTING.md         # このファイル
└── README.md               # プロジェクト概要
```

## ドキュメントの追加・編集

1. `docs/src/` ディレクトリに Markdown ファイルを追加または編集
2. `docs/mkdocs.yml` でナビゲーション構造を更新（必要に応じて）
3. ローカルで `mkdocs serve` を実行して変更を確認
4. 変更をコミットして `main` ブランチにプッシュ

## 質問やフィードバック

質問やフィードバックがある場合は、GitHubのIssuesで報告してください。
