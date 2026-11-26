# 技術記事の管理

Zennの記事と本、Qiitaの記事をMarkdownで管理するリポジトリです。

## 原稿と公開先

| 公開先 | ファイルの場所 | 公開方法 |
| --- | --- | --- |
| Zenn | 記事は `zenn/articles/`、本は `zenn/books/`、画像は `zenn/images/` | `main` から公開用ブランチ `zenn-deploy` に自動同期し、Zennに反映 |
| Qiita | 記事は `qiita/public/` | Qiita CLIで対象の記事を指定して投稿 |

このGitHubリポジトリは公開されています。ZennやQiitaで下書きにしていても、GitHubへプッシュしたファイルは閲覧できます。公開前に伏せたい原稿や画像は手元に置いてください。

## 初回セットアップ

Node.js 22.22.1以上を用意して、リポジトリをクローンします。以降のコマンドも `package.json` があるリポジトリのルートで実行します。

```bash
git clone https://github.com/kkoch5t2/tech-articles.git
cd tech-articles
npm ci
```

### Qiitaを使う場合

Qiitaにログインし、`read_qiita` と `write_qiita` を選んでアクセストークンを発行します。[Qiitaのアクセストークン発行画面](https://qiita.com/settings/tokens/new)

次のコマンドを実行し、トークンをCLIの入力欄に貼り付けます。トークンはGitにコミットしないでください。

```bash
npm run qiita -- login
```

## Zennの記事を書く・公開する

1. `npm run zenn -- new:article` で原稿を作ります。`zenn/articles/` にMarkdownファイルが生成されます。下書きの間はファイル先頭の `published` を `false` にします。
2. `npm run zenn -- preview` を実行し、`http://localhost:8000` で本文、画像、リンクを確認します。
3. 公開するときは `published: true` にして、`main` に反映します。GitHub Actionsが `zenn/` の中身を `zenn-deploy` の直下へ同期します。反映結果は[GitHub Actionsの実行履歴](https://github.com/kkoch5t2/tech-articles/actions/workflows/sync-zenn.yml)と[Zennのデプロイ履歴](https://zenn.dev/dashboard/deploys)で確認できます。

公開済みの記事を直すときは、同じファイル名（`slug`）の原稿を編集して反映します。本を作るときは `npm run zenn -- new:book` を実行します。

画像は `zenn/images/` に置き、記事内では `/images/ファイル名` で参照します。記事の編集は作業ブランチで行い、公開用の `zenn-deploy` はGitHub Actionsで更新します。

## Qiitaの記事を書く・公開する

### 原稿を作ってプレビューする

```bash
npm run qiita -- new article-name
npm run qiita -- preview
```

`article-name` は付けたいファイル名に置き換えます。原稿は `qiita/public/article-name.md` に作られます。原稿を作ったら、執筆中はファイル先頭の `ignorePublish` を `true` にします。プレビューは `http://localhost:8888` で確認できます。

既存の公開記事を取り込むときは `npm run qiita -- pull` を実行します。`preview` の起動時にもQiitaの記事が同期されるので、取り込まれたファイルはコミット前に差分を確認してください。

### 1本ずつ投稿・更新する

タイトルとタグを設定し、`ignorePublish: false`、`private: false` にして、対象のファイル名を拡張子なしで指定します。

```bash
npm run qiita -- publish article-name
```

投稿後に書き込まれた `id` と `updated_at` を含めて原稿をコミットします。次回は `id` を残した同じ原稿を編集し、同じコマンドで更新します。公開ページを開き、本文と画像の表示を確認します。

### GitHub Actionsから1本ずつ投稿・更新する

1. Qiitaで `read_qiita` と `write_qiita` を選んで発行したアクセストークンを、[Actionsのリポジトリシークレット](https://github.com/kkoch5t2/tech-articles/settings/secrets/actions)に `QIITA_TOKEN` という名前で登録します。
2. `main` に原稿を反映し、[Qiitaの記事を投稿・更新](https://github.com/kkoch5t2/tech-articles/actions/workflows/publish-qiita.yml)を開きます。
3. `Run workflow` を選び、ブランチを `main` にします。`article` に `qiita/public/` 内の記事ファイル名を拡張子なしで入力して実行します。

実行履歴で公開先を確認できます。投稿や更新でQiita CLIが書き込んだ `id` と `updated_at` は `main` に反映されます。

## 公式ドキュメント

- [Zenn CLIで記事・本を管理する方法](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [ZennとGitHubリポジトリを連携する方法](https://zenn.dev/zenn/articles/connect-to-github)
- [Qiita CLIの使い方](https://github.com/increments/qiita-cli)
