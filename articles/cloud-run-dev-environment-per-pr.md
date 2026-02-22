---
title: "シンプル構成で実現する - Cloud Run によるプルリクエスト単位の検証環境構築🚀"
emoji: "🚀"
type: "tech"
topics:
  - "cloudrun"
  - "githubactions"
  - "googlecloud"
  - "cicd"
  - "開発生産性"
published: true
published_at: "2025-03-31 07:50"
---

この記事では Cloud Run を用いたプルリクエスト単位の検証環境をシンプルな構成で構築する方法を共有します。

同じようなことを実現したいと考えている方の参考になれば幸いです。

## 作成する構成

今回作成するシステムの構成は以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/f338ed305185-20250328.jpg)

以下の流れで処理が実行されます。

1. 開発者が Pull Request に dev ラベルを付与する
2. GitHub Actions のワークフローが起動して以下の処理を行う
   - A レコードの設定
   - タグ付きリビジョン(pr-1)のデプロイ
3. 名前解決：ユーザーが `pr-1.dev.example.com` にアクセスする
4. ワイルドカード証明書を用いた SSL 通信
5. URL マスクで対応するリビジョンにリクエストを転送する

これにより Pull Request 単位で独立した検証環境が自動的に構築され、開発者とユーザーは専用 URL で各環境にアクセスできるようになります。

## 構築手順

### Terraform でインフラ構築

Cloud Run を使ったプルリクエスト単位の検証環境を構築するためには、まずは Terraform でインフラを準備します。

ここでは、構成の主要なポイントのみを解説します。

#### ワイルドカード証明書の設定

一つ目のポイントはワイルドカード証明書を利用する点です。これにより `*.dev.example.com`（`*` は任意の値）の形式で各 PR の環境にアクセスできるようになります。

以下はワイルドカード証明書を設定するための Terraform コードのサンプルです。

```tf
resource "google_certificate_manager_dns_authorization" "wildcard" {
  name    = "wildcard-dev-example"
  domain  = "dev.example.com"
  project = var.project_id
}

resource "google_certificate_manager_certificate" "wildcard" {
  name    = "wildcard-dev-example"
  project = var.project_id

  managed {
    domains = ["*.dev.example.com"]
    dns_authorizations = [
      google_certificate_manager_dns_authorization.wildcard.id,
    ]
  }
}

resource "google_certificate_manager_certificate_map" "wildcard" {
  name    = "wildcard-dev-example"
  project = var.project_id
}

resource "google_certificate_manager_certificate_map_entry" "wildcard" {
  name         = "wildcard-dev-example"
  map          = google_certificate_manager_certificate_map.wildcard.name
  hostname     = "*.dev.example.com"
  project      = var.project_id
  certificates = [google_certificate_manager_certificate.wildcard.id]
}

# 証明書検証用のDNSレコード（CNAME）の設定
resource "google_dns_record_set" "cname_record" {
  name         = google_certificate_manager_dns_authorization.wildcard.dns_resource_record[0].name
  type         = "CNAME"
  ttl          = 300
  managed_zone = var.managed_zone
  project      = var.project_id
  rrdatas      = [google_certificate_manager_dns_authorization.wildcard.dns_resource_record[0].data]
}
```

このコードにより `*.dev.example.com` に対するワイルドカード証明書が生成され、各 PR の検証環境（`pr-1.dev.example.com`, `pr-2.dev.example.com` など）で HTTPS 通信が可能になります。

#### Serverless NEG の設定

二つ目のポイントは Serverless NEG における URL マスクの設定です。

URL マスクとは、簡単に言えば「URL のパターンに基づいて適切なサービスやリビジョンにリクエストを振り分ける仕組み」です。これを使うことで、プルリクエスト番号に応じた環境に自動的にリクエストを転送できます。

例えば、URL マスクに`<tag>.dev.example.com`を設定すると：

- ユーザーが `pr-1.dev.example.com` にアクセスした場合 → `pr-1` タグが付いた Cloud Run リビジョンにリクエストが転送
- ユーザーが `pr-2.dev.example.com` にアクセスした場合 → `pr-2` タグが付いた Cloud Run リビジョンにリクエストが転送

これにより、プルリクエスト番号に対応する独立した環境に自動的にルーティングされます。URL マスクの `<tag>` 部分がプルリクエスト番号に対応するタグに置き換わる仕組みです。

URL マスクの詳細については[こちら](https://cloud.google.com/load-balancing/docs/negs/serverless-neg-concepts?hl=ja#url_masks)をご覧ください。

Serverless NEG（Network Endpoint Group）の Terraform コードのサンプルは以下の通りです。

```tf
resource "google_compute_region_network_endpoint_group" "serverless_neg" {
  name                  = "neg-dev-example"
  network_endpoint_type = "SERVERLESS"
  project               = "your-project-id"
  region                = "asia-northeast1"

  cloud_run {
    service  = "your-cloud-run-service"
    url_mask = "<tag>.dev.example.com"
  }
}
```

### GitHub Actions でワークフローを実装

開発者がプルリクエストに `dev` ラベルを付与した際に自動的に環境をデプロイし、プルリクエストがクローズされた際に環境をクリーンアップする GitHub Actions ワークフローを実装します。

#### デプロイワークフロー

プルリクエストに `dev` ラベルが付与されたときに実行される環境構築用のワークフローです。このワークフローでは、プルリクエストの番号に基づいて一意の環境を作成し、その URL をプルリクエストにコメントします。

```yml
name: プルリクエスト検証環境のデプロイ

on:
  pull_request:
    types:
      - labeled

permissions:
  contents: read
  id-token: write
  pull-requests: write

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

defaults:
  run:
    shell: bash

env:
  APP_NAME: your-app-name
  PROJECT_ID: your-project-id
  REGION: asia-northeast1
  WORKLOAD_IDENTITY_PROVIDER: your-workload-identity-provider
  SERVICE_ACCOUNT: your-service-account
  DNS_ZONE: your-dns-zone
  BASE_DOMAIN: dev.example.com
  LOAD_BALANCER_IP: your-load-balancer-ip

jobs:
  deploy-dev:
    if: contains(github.event.pull_request.labels.*.name, 'dev')
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: リポジトリのチェックアウト
        uses: actions/checkout@v4

      - name: 環境変数の設定
        run: |
          echo "IMAGE_URI=${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/containers/${{ env.APP_NAME }}" >> $GITHUB_ENV
          echo "IMAGE_TAG=pr-${{ github.event.pull_request.number }}-${{ github.sha }}" >> $GITHUB_ENV
          echo "REVISION_TAG=pr-${{ github.event.pull_request.number }}" >> $GITHUB_ENV
          echo "SERVER_NAME=pr-${{ github.event.pull_request.number }}.${{ env.BASE_DOMAIN }}" >> $GITHUB_ENV
          echo "DEPLOY_URL=https://pr-${{ github.event.pull_request.number }}.${{ env.BASE_DOMAIN }}" >> $GITHUB_ENV

      - name: Google Cloudの認証
        id: auth
        uses: google-github-actions/auth@v2
        with:
          token_format: access_token
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: アプリケーションのビルド
        run: |
          # ここでアプリケーションのビルド処理を実行

      - name: Artifact Registryへのログイン
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGION }}-docker.pkg.dev
          username: oauth2accesstoken
          password: ${{ steps.auth.outputs.access_token }}

      - name: Dockerイメージのビルドとプッシュ
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ env.IMAGE_URI }}:${{ env.IMAGE_TAG }}

      - name: Cloud Runサービスのデプロイ
        run: |
          gcloud run deploy your-cloud-run-service \
            --region=${{ env.REGION }} \
            --project=${{ env.PROJECT_ID }} \
            --image=${{ env.IMAGE_URI }}:${{ env.IMAGE_TAG }} \
            --tag=${{ env.REVISION_TAG }} \
            --min-instances=0 \
            --max-instances=2 \
            --ingress=internal-and-cloud-load-balancing

      - name: DNSレコードの作成
        id: create-dns
        run: |
          RECORD_NAME="${{ env.SERVER_NAME }}"

          if ! gcloud dns record-sets list \
            --zone=${{ env.DNS_ZONE }} \
            --name="${RECORD_NAME}." \
            --project ${{ env.PROJECT_ID }} | grep -q "${RECORD_NAME}."; then

            gcloud dns record-sets create "${RECORD_NAME}." \
              --zone=${{ env.DNS_ZONE }} \
              --type=A \
              --ttl=300 \
              --rrdatas=${{ env.LOAD_BALANCER_IP }} \
              --project ${{ env.PROJECT_ID }}

            echo "dns_created=true" >> $GITHUB_OUTPUT
          else
            echo "DNSレコードは既に存在します"
            echo "dns_created=false" >> $GITHUB_OUTPUT
          fi

      - name: プルリクエストへのコメント
        if: steps.create-dns.outputs.dns_created == 'true'
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: ${{ github.event.pull_request.number }},
              body: `### :rocket: 検証環境がデプロイされました！

              **${process.env.DEPLOY_URL}**

              修正内容をプッシュした場合、再度devラベルを付け直すと最新の修正内容が反映されます`
            });
```

#### クリーンアップワークフロー

プルリクエストがクローズされたときに実行される環境クリーンアップ用のワークフローです。このワークフローによって、不要になった環境を自動的に削除し、リソースの無駄遣いを防ぎます。

```yml
name: プルリクエスト検証環境のクリーンアップ

on:
  pull_request:
    types:
      - closed

permissions:
  contents: read
  id-token: write

defaults:
  run:
    shell: bash

env:
  PROJECT_ID: your-project-id
  WORKLOAD_IDENTITY_PROVIDER: your-workload-identity-provider
  SERVICE_ACCOUNT: your-service-account
  DNS_ZONE: your-dns-zone
  BASE_DOMAIN: dev.example.com

jobs:
  cleanup-dev:
    runs-on: ubuntu-latest
    timeout-minutes: 5

    steps:
      - name: リポジトリのチェックアウト
        uses: actions/checkout@v4

      - name: Google Cloudの認証
        uses: google-github-actions/auth@v2
        with:
          token_format: access_token
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: DNSレコードの削除
        run: |
          RECORD_NAME="pr-${{ github.event.pull_request.number }}.${{ env.BASE_DOMAIN }}"

          if gcloud dns record-sets list \
              --zone=${{ env.DNS_ZONE }} \
              --name="${RECORD_NAME}." \
              --project ${{ env.PROJECT_ID }} | grep -q "${RECORD_NAME}."; then

            gcloud dns record-sets delete "${RECORD_NAME}." \
              --zone=${{ env.DNS_ZONE }} \
              --type=A \
              --project ${{ env.PROJECT_ID }}

            echo "✅ DNSレコード ${RECORD_NAME} が正常に削除されました"
          else
            echo "ℹ️ DNSレコード ${RECORD_NAME} が見つかりません"
          fi
```

これらのワークフローにより、開発者がプルリクエストに `dev` ラベルを付けるだけで自動的に検証環境が構築されます。また、プルリクエストがクローズされると自動的に環境がクリーンアップされます。

## おわりに

本記事では、Cloud Run を活用したプルリクエスト単位の検証環境構築方法について解説しました。

この仕組みを導入することで、以下のようなメリットが得られます：

- 各プルリクエストに対して独立した検証環境が自動的に構築されるため、並行開発がスムーズになります
- レビュアーが実際の動作を確認しながらレビューできるため、フィードバックが効率化されます
- プルリクエストがクローズされると自動的に環境がクリーンアップされるため、リソースの無駄がありません
- 本番と同等の環境でテストできるため、環境差異によるバグを開発初期段階で発見可能になります

必要に応じて各チームの要件に合わせてカスタマイズしていただければ幸いです。

最後までお読みいただき、ありがとうございました。この記事が皆様のプロジェクトの参考になれば幸いです。
