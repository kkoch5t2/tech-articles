# 技術記事の管理リポジトリ

Zennの記事と本、Qiitaの記事をMarkdownで管理します。

## 初期セットアップ

新しく環境を構築する場合（別のPCでクローンした時など）は、最初に依存パッケージをインストールしてください。

```bash
npm ci
```

## Zennの使い方

Zenn CLIを使用して、記事の作成やプレビューを行います。コマンドはすべてこのディレクトリ（`package.json` がある場所）で実行してください。

### ローカルプレビューの起動

```bash
npx zenn preview
```
実行後、ブラウザで `http://localhost:8000` にアクセスすると、執筆中の記事や本をリアルタイムでプレビューできます。

### 新規記事の作成

```bash
npx zenn new:article
```
`articles` ディレクトリに新しいMarkdownファイルが生成されます。

### 新規本の作成

```bash
npx zenn new:book
```
`books` ディレクトリに新しい本のディレクトリが生成されます。

## Qiitaの使い方

Qiitaの原稿は `qiita/public/` に置きます。Qiita CLI 1.13.0 には Node.js 22.22.1 以上が必要です。

### 初回のみ認証する

Qiitaの[アクセストークン発行画面](https://qiita.com/settings/tokens/new)で `read_qiita` と `write_qiita` を選んで発行し、以下のコマンドの入力欄に貼り付けます。トークンはCLIの認証設定に保存してください。

```bash
npm run qiita -- login
```

既存の公開記事を取り込む場合は `npm run qiita -- pull` を実行します。

### 記事を書く

```bash
npm run qiita -- new article-name
npm run qiita -- preview
```

`qiita/public/article-name.md` が生成されます。執筆中は先頭の `ignorePublish` を `true` にします。このGitHubリポジトリは公開されているため、公開前に伏せておきたい原稿や画像は手元で管理してください。

### 記事を公開・更新する

対象の原稿にタイトルとタグを入れ、`ignorePublish: false` にしてから、ファイル名を指定して投稿します。

```bash
npm run qiita -- publish article-name
```

初回投稿でファイルに書き込まれる `id` と `updated_at` をコミットしてください。以後は同じ `id` のファイルを編集して投稿すると、既存記事が更新されます。URLを開いて本文と画像の表示を確認してください。

## 参考リンク
* [📘 Zenn CLI ガイド (公式)](https://zenn.dev/zenn/articles/zenn-cli-guide)
* [Qiita CLI 公式ドキュメント](https://github.com/increments/qiita-cli)
