# zenn

## 概要

このディレクトリでは、[Zenn](https://zenn.dev/newbee1958)に公開する記事やブックを管理しています。main ブランチにマージされた内容が自動的に Zenn に公開されます。

## ディレクトリ構造

```
.
├── articles/    # 記事を管理するディレクトリ
├── books/       # 本を管理するディレクトリ
└── images/      # 画像を管理するディレクトリ
```

## セットアップ手順

1.  zenn ディレクトリに移動する

    ```bash
    cd zenn
    ```

2.  必要なパッケージをインストールする

    ```bash
    npm ci
    ```

## 記事の作成方法

1.  新しい記事を作成する

    ```bash
    npx zenn new:article --slug 記事のスラッグ

    npx zenn new:article --slug 記事のスラッグ --title タイトル --type idea --emoji ✨
    ```

    > **スラッグ（slug）について**
    >
    > - 記事の URL に使用される一意の識別子です（`https://zenn.dev/ユーザー名/articles/スラッグ`）。
    > - 12〜50文字の半角英数字（`a-z`、`0-9`）とハイフン（`-`）が使用可能です。
    > - `slug` オプションを省略した場合は、ランダムな14文字の文字列が自動で生成されます。

2.  記事をプレビューする

    ```bash
    npx zenn preview
    ```

3. publishedの設定を変更

`published: true`に設定することで、記事が公開状態に設定される。

4.  変更をコミットしてプッシュする

    ```bash
    git add .
    git commit -m "記事を追加"
    git push origin main
    ```

5.  main ブランチにマージされると自動的に Zenn に公開されます

## CLI をアップデートする

Zenn CLI の表示が zenn.dev と異なるときや CLI 利用時に更新通知が表示されたときは下記のコマンドでアップデートを行ってください。

```
$ npm install zenn-cli@latest
```

## 日時を指定して記事を公開する（公開予約する）

公開時間を指定して記事を公開するには、Front Matter にて `published` を `true` にした上で、 `published_at` を指定します。`published_at` のフォーマットは、 `YYYY-MM-DD` または `YYYY-MM-DD hh:mm` です。日付だけを指定した場合、時刻は `00:00` となります。

```
published: true # trueを指定する
published_at: 2050-06-12 09:03 # 未来の日時を指定する
```

この状態で、GitHub リポジトリへプッシュすると、zenn.dev 上で記事が公開予約状態となり、公開予約時刻が過ぎると自動的に記事が公開されます。

> ⚠️ `published_at` のタイムゾーンは JST（日本時間）です。

## 参考資料

### Zenn と GitHub の連携

[Zenn と Github を連携する方法](https://zenn.dev/eguchi244_dev/articles/github-zenn-linkage-20230501)

Zenn と github を連携して、GitHub リポジトリで Zenn のコンテンツを管理する方法についての解説記事です。

### Zenn 公式ドキュメント

[GitHub リポジトリで Zenn のコンテンツを管理する](https://zenn.dev/zenn/articles/connect-to-github)

Zenn と GitHub の連携方法についての公式ドキュメントです。

[Zenn CLI をインストールする](https://zenn.dev/zenn/articles/install-zenn-cli)

Zenn CLI のインストール方法についての公式ドキュメントです。

[Zenn CLI の使い方](https://zenn.dev/zenn/articles/zenn-cli-guide)

Zenn CLI の基本的な使い方についての公式ドキュメントです。
