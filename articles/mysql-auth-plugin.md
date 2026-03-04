---
title: "認証プラグインを mysql_native_password にしたまま Cloud SQL の MySQL 8.4 を利用する方法"
emoji: "🐷"
type: "tech"
topics:
  - "mysql"
  - "インフラ"
  - "googlecloud"
  - "cloudsql"
published: true
published_at: "2024-12-11 07:13"
---

## コンテキスト

以下の状況を想定しています。

- Cloud SQL の MySQL 5.7 を利用している
- 諸事情により Cloud SQL の MySQL 8.4 に移行することになった
- 利用するアプリケーションの制約により caching_sha2_password が使えないため、mysql_native_password を利用する必要がある

## 解決方法

Cloud SQL for MySQL 8.4 以降では、caching_sha2_password プラグインがデフォルトの認証プラグインになります。

そのため、MySQL 8.4 でインスタンスを構築したあとに作成したユーザーは caching_sha2_password が使用されてしまいます。

しかし、既存の mysql_native_password を利用しているユーザーは、MySQL 8.4 にバージョンアップしてもそのまま利用することができます。

参考: [MySQL 8.4 認証プラグインのデフォルト](https://cloud.google.com/sql/docs/mysql/features?hl=ja)

つまり、以下の手順を踏めば、Cloud SQL の MySQL 8.4 を利用しつつ、mysql_native_password を利用することができるということです。

1. MySQL 5.7 で Cloud SQL インスタンスを構築
2. ユーザーを作成（mysql_native_password）
3. MySQL 8.4 にバージョンアップ

## バージョンアップの際の注意点

ただ、Cloud SQL のバージョンアップには以下の制約があります。

- 現在のメジャーバージョンの次のメジャーバージョンにのみアップグレードできる
- MySQL 8.4 に上げるには、MySQL 8.0.37 以上にしておく必要がある

参考: [Upgrade the database major version in-place](https://cloud.google.com/sql/docs/mysql/upgrade-major-db-version-inplace)

上記を踏まえると、以下の流れで Cloud SQL インスタンスを構築する必要があります。

1. MySQL 5.7 で Cloud SQL インスタンスを構築
2. ユーザーを作成（mysql_native_password）
3. MySQL 8.0.37 にバージョンアップ
4. MySQL 8.4 にバージョンアップ

段階的にバージョンアップする必要がある点に注意しましょう。

## おわりに

今回は、認証プラグインを mysql_native_password にしたまま Cloud SQL の MySQL 8.4 を利用する方法を解説しました。

ただ、できるのであれば caching_sha2_password を使えるようにするのが最優先なのは間違いありません。

どうしようもない場合の対応策として参考にしていただければと思います。
