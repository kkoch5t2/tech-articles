# Zenn コンテンツ管理リポジトリ

Zennの記事、本、画像を管理するリポジトリです。記事は articles/、本は books/、画像は images/ に置きます。

## 原稿を作成する

リポジトリのルートで次のコマンドを実行します。

    npm ci
    npx zenn new:article
    npx zenn preview

本を作成する場合は npx zenn new:book を実行します。プレビューは http://localhost:8000 で確認できます。

## 記事を公開・更新する

1. 下書きの間は記事ファイルの published を false にします。
2. 公開時に published を true にして main ブランチに反映します。ZennとのGitHub連携で自動反映されます。
3. [Zennのデプロイ履歴](https://zenn.dev/dashboard/deploys)で反映を確認します。

既存の記事を更新するときは、同じスラッグのMarkdownファイルを編集します。画像は /images/ 以下のパスで参照します。

- [Zenn CLI公式ガイド](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [ZennのGitHub連携](https://zenn.dev/zenn/articles/connect-to-github)
