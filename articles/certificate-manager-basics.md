---
title: "Certificate Managerの設定手順を解説【Google Cloud】"
emoji: "🐷"
type: "tech"
topics:
  - "gcp"
  - "googlecloud"
  - "sre"
  - "証明書"
published: true
published_at: "2024-12-08 22:18"
---

## Google CloudのSSL証明書について

Google Cloud で SSL 証明書を構成する方法は以下の２つです。

- Compute Engine SSL 証明書リソース
- Certificate Manager

Certificate Managerのコンソール上では、Compute Engine SSL 証明書リソースは「従来の証明書」に対応し、Certificate Managerは「証明書」に対応しています。

![](https://storage.googleapis.com/zenn-user-upload/724337449f09-20241114.png)

どちらの方法でも、「セルフマネージド SSL 証明書」と 「Google マネージド SSL 証明書」がサポートされています。

セルフマネージド SSL 証明書 とは、ユーザー自身で取得して、プロビジョニング・更新する証明書です。
マネージド SSL 証明書 は、Google Cloud が自動的に取得・管理・更新する証明書です。

本記事では、**Certificate Managerのマネージド SSL 証明書**を設定する手順を解説していきます。

## Certificate Managerの仕組みについて

Certificate Managerは以下のコンポーネントで構成されています。

- 証明書
- ドメインの承認
- 証明書マップエントリ
- 証明書マップ

![](https://storage.googleapis.com/zenn-user-upload/fdea03b9a5fe-20241114.png)

参考: [Certificate Manager の仕組み](https://cloud.google.com/certificate-manager/docs/how-it-works?hl=ja)

それぞれのコンポーネントについて簡単に解説します。

### 証明書

Certificate Manager は、以下のタイプの証明書をサポートしています。

- セルフマネージド SSL 証明書
- Google マネージド証明書

セルフマネージド SSL 証明書 は、ユーザー自身で取得して、プロビジョニング・更新する証明書です。

Google マネージド証明書は、Google Cloud がユーザーに代わり取得して管理する証明書で、`ロードバランサベース`または `DNS ベース`の承認を使用して、ドメインの所有権を確認することができます。

### ドメインの承認

Certificate Managerのドメインの承認（メインの所有者であることを認証）には、以下の２つの方法があります。

- ロードバランサベースの認証
- DNS ベースの認証

ロードバランサベースの認証の場合は、特別な手順やDNSの設定の必要はありませんが、ターゲットプロキシに証明書を関連付けてから証明書のステータスが ACTIVE になるまでに時間がかかります。

つまり、アプリケーションでダウンタイムが発生します。

一方 DNSベースの認証は、Google Cloud から指定された CNAME レコードを DNS に登録する手間はありますが、ターゲットプロキシに関連付ける前に証明書のステータスを ACTIVE にすることが可能です。

そのためダウンタイムは発生しません。

システムのマイグレーション等で、ダウンタイムを発生させたくない場合は「DNS ベースの認証」を利用することになるでしょう。

また、ロードバランサの承認を使用した Google マネージド証明書は、ワイルドカード ドメイン名をサポートしていません。

一方、DNSベースの認証はワイルドカード ドメイン名をサポートしています。

そのため、ワイルドカード証明書を利用したい場合も、DNSベースの認証を利用することになるでしょう。

本記事では、`DNS ベースの認証`を利用する手順をご紹介します。

### 証明書マップエントリ

ドメイン名と 証明書 の紐付けをするコンポーネントです。

www.example.com には証明書 A を、xxx.example.com には証明書 B を適応させるといったように、ドメインやサブドメインなどのドメイン名ごとに異なる証明書セットを定義できます。

### 証明書マップ

証明書マップは、特定の証明書を特定のホスト名に割り当てる 1 つ以上の証明書マップエントリを参照します。

クライアントが証明書マップで指定されたホスト名をリクエストすると、ロードバランサはそのホスト名にマッピングされた証明書を提供します。

## Certificate Managerの設定手順

次に、Certificate Managerで Google マネージド SSL 証明書を作成し、LBに設定する手順をご紹介します。

想定するドメインは`example.com`です。

また、ドメインの認証には 「DNS認証」 を利用します。

### 1. DNS 認証を作成する

以下のコマンドで DNS 認証を作成します。

```shell
gcloud certificate-manager dns-authorizations create example-app --domain="example.com" --project="sample-1234"
```

以下のコマンドで DNS 認証の CNAME レコードの内容を表示します。

```shell
gcloud certificate-manager dns-authorizations describe example-app --project="sample-1234"
```

以下のように DNS に設定する CNAME レコードが出力されます。（data: の値）

```
createTime: '2022-01-14T13:35:00.258409106Z'
dnsResourceRecord:
data: hogehoge-fugafuga.1.authorize.certificatemanager.goog.
name: _acme-challenge.myorg.example.com.
type: CNAME
domain: myorg.example.com
name: projects/myProject/locations/global/dnsAuthorizations/myAuthorization
updateTime: '2022-01-14T13:35:01.571086137Z'
```

### 2. DNS (e.g. Route53)に CNAME レコードを追加する

Route53 等の DNSサーバー の以下の example.com ゾーンのページにアクセスして、以下の形式のレコードを追加します。

| 証明書名                             | レコード名                                    | レコードタイプ | 値                |
| ------------------------------------ | --------------------------------------------- | -------------- | ----------------- |
| example-app                       | `_acme-challenge.example.com`        | CNAME          | 手順 1 の出力結果 |

以下の dig コマンドを実行して、値が返ってくること（CNAME が設定できていること）を確認します。

```shell
dig _acme-challenge.example.com CNAME
```

### 3. DNS 認証を参照する Google マネージド証明書を作成する

```shell
gcloud certificate-manager certificates create example-app \
  --project="sample-1234" \
  --domains=example.com \
  --dns-authorizations=example-app
```

### 4. 証明書が有効であることを確認する

証明書をロードバランサにデプロイする前に、証明書自体が有効であることを確認します。

```shell
gcloud certificate-manager certificates describe example-app --project="sample-1234"
```

上記のコマンドのステータスが`ACTIVE`であれば次の手順に進みます。

### 5. 証明書マップを作成する

証明書に関連付けられた証明書マップエントリを参照する証明書マップを作成します。

```shell
gcloud certificate-manager maps create example-app --project="sample-1234"
```

### 6. 証明書マップエントリを作成する

証明書マップエントリを作成し、証明書および証明書マップに関連付けます。

```shell
gcloud certificate-manager maps entries create example-app \
  --project="sample-1234" \
  --map="example-app" \
  --certificates="example-app" \
  --hostname="example.com"
```

### 7. 証明書マップエントリが有効であることを確認する

```shell
gcloud certificate-manager maps entries describe example-app \
  --project="sample-1234" \
  --map="example-app"
```

上記のコマンドのステータスが`ACTIVE`であれば次の手順に進みます。

### 8. 証明書マップを LB のターゲットプロキシに紐付ける

以下のコマンドを実行することで、ターゲットプロキシが参照する証明書が切り替わります。

証明書は既にACTIVEなので、紐付けた直後からSSL通信が可能となります。

```shell
gcloud compute target-https-proxies update lb-example-target-proxy \
  --project="sample-1234" \
  --certificate-map="example-app" \
  --global
```

## まとめ

Google Cloudの証明書は、以下のように分類できます。

- Compute Engine SSL 証明書リソース
    - セルフマネージド SSL 証明書
    - Google マネージド SSL 証明書
- Certificate Manager
    - セルフマネージド SSL 証明書
    - Google マネージド SSL 証明書
        - ロードバランサベースの認証
        - DNS ベースの認証

この中で今回は、**Certificate Managerで DNS ベースの認証 を用いて Google マネージド SSL 証明書を作成しLBに設定する手順**をご紹介しました。

Certificate Managerの設定をする上で少しでも参考になっていれば幸いです。

最後まで読んでいただきありがとうございました。

## 参考資料

- [SSL 証明書の概要](https://cloud.google.com/load-balancing/docs/ssl-certificates?hl=ja)
- [Certificate Manager の仕組み](https://cloud.google.com/certificate-manager/docs/how-it-works?hl=ja)
- [Certificate Manager デプロイの概要](https://cloud.google.com/certificate-manager/docs/deploy?hl=ja)
