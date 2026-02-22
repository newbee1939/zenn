---
title: "Private Service Connectチュートリアル【Google Cloud】"
emoji: "🐷"
type: "tech"
topics:
  - "network"
  - "インフラ"
  - "googlecloud"
  - "vpc"
  - "sre"
published: true
published_at: "2024-12-09 06:30"
---

この記事では、Private Service Connectを実際に構築しながら構成や作り方を理解していきます。

少しでもこれからPrivate Service Connectを利用しようと思っている方の参考になれば幸いです。

## Private Service Connectとは何か？

Private Service Connectとは、**異なるVPC間のサービスをプライベートに接続するための機能**です。

Private Service Connectを用いることで、Private IPのみで相互に接続することができます。


似たような機能に「VPCピアリング」があります。

VPCピアリングは、その名の通りVPC同士を接続する機能です。

こちらもピアリングの設定をすることで、Private IPのみで接続することができます。

しかし、VPCピアリングには**VPC間で同一のCIDR（IP範囲）を使えない**などの制約があります。また、相互接続を許可するためのファイアウォールの設定などもそれぞれのVPCで行う必要があります。

一方Private Service Connectにはそのような制約はありません。

そのため、VPC間で仕様を調整することが難しい場合などではPrivate Service Connectを使う機会が多くなるでしょう。


## Private Service Connectの構成

Private Service Connectの構成要素は大きく分けて以下の二つです。

- プロデューサー
    - サービスアタッチメント
- コンシューマー
    - エンドポイント

プロデューサーはサービスを提供する側で、コンシューマーはサービス利用側です。

さらに、以下のような関連リソースも構築する必要があります。

- ロードバランサ
- VPC
- サブネット
- ファイアウォールルール


今回は、実際にこれらのリソースを全て0から構築していきます。


## Private Service Connect(PSC)を構築するチュートリアル

今回作成する構成は以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/de6a95062fe9-20241208.png)

VPC内にあるCompute Engine同士を内部通信で接続します。

それでは、一つ一つ必要なリソースを構築していきましょう。

[バージョン情報]
- Google Cloud SDK: 477.0.0


### 1. プロデューサーVPCネットワークの作成

まずはプロデューサー側のVPCネットワークを作成します。

```shell
gcloud compute networks create vpc-psc-producer \
  --project=sample123 \
  --subnet-mode=custom
```

`--subnet-mode=custom`を指定することで、手動でサブネットを作成することができます。

参考: [gcloud compute networks create](https://cloud.google.com/sdk/gcloud/reference/compute/networks/create)

### 2. プロデューサーVPC内にVMインスタンスなどを配置するためのサブネットを作成

作成したVPC内にサブネットを作成します。

このサブネットはCompute Engineやロードバランサのコンポーネントに利用します。

```shell
gcloud compute networks subnets create subnet-vpc-psc-producer \
  --project=sample123 \
  --range=10.0.0.0/29 \
  --network=vpc-psc-producer \
  --region=us-central1
```

[--range](https://cloud.google.com/sdk/gcloud/reference/compute/networks/subnets/create#--range)には、サブネットのプライマリ IPv4 範囲（CIDR 表記）を指定します。

利用可能なサブネットの範囲は[こちら](https://cloud.google.com/vpc/docs/subnets?hl=ja#ipv4-ranges)に書かれています。
Private IPアドレスの仕様については、[こちら](https://datatracker.ietf.org/doc/html/rfc1918)のRFC1918にも記載があります。

### 3. サブネット内にプロデューサー側のサービス（VMインスタンス）を作成

サブネット内にリクエストを受ける側のVMインスタンスを作成します。

```shell
gcloud compute instances create vm-psc-producer \
  --project=sample123 \
  --machine-type=e2-micro \
  --subnet=subnet-vpc-psc-producer \
  --zone=us-central1-a \
  --private-network-ip=10.0.0.2 \
  --no-address \
  --tags=vm-psc-producer-tag \
  --metadata=startup-script='#! /bin/bash
    apt update
    apt -y install apache2
    cat <<EOF > /var/www/html/index.html
    <html><body><p>OK!</p></body></html>
    EOF'
```

zoneには、サブネットを作成したリージョン内のzoneを指定する必要があります。

また後の手順でLBなどのリソースと紐づけるため、tag（`vm-psc-producer-tag`）も指定しています。

参考: [予約済みの内部 IPv4 または IPv6 アドレスを使用して VM インスタンスを作成する](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-internal-ip-address?hl=ja#create_a_vm_instance_with_a_specific_internal_ip_address)
参考: [Linux VM での起動スクリプトの使用](https://cloud.google.com/compute/docs/instances/startup-scripts/linux?hl=ja)

### 4. インスタンスグループの作成

ロードバランサから転送する際のバックエンドにVMを指定するため、インスタンスグループ（`vm-psc-producer-instance-group`）を作成します。

ロードバランサのバックエンドに特定のインスタンスを指定することはできません。インスタンスを含むインスタンスグループを作成し、そのグループをバックエンドに指定する必要があります。

参考: [マネージド VM インスタンス グループのバックエンドを作成する](https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal?hl=ja#create-instance-group-backend)

また、インスタンスグループに対して名前付きポートも作成します。

名前付きポートはインスタンスグループのコンソールに「ポート名のマッピング」として表示され、後の手順でロードバランサのバックエンド サービスに名前付きポートを指定する際に使用します。

```shell
// インスタンスグループの作成
gcloud compute instance-groups unmanaged create vm-psc-producer-instance-group --project=sample123 --zone=us-central1-a

// インスタンスグループにインスタンスを追加
gcloud compute instance-groups unmanaged add-instances vm-psc-producer-instance-group --project=sample123 --zone=us-central1-a --instances=vm-psc-producer

// 名前付きポートを定義
gcloud compute instance-groups unmanaged set-named-ports vm-psc-producer-instance-group \
  --project=sample123 \
  --zone=us-central1-a \
  --named-ports=vm-psc-producer-instance-group-port:80
```

参考: [名前付きポート](https://cloud.google.com/load-balancing/docs/backend-service?hl=ja&_gl=1*1epu9l3*_ga*MjA1MDI4ODk3NS4xNjk4MTI1ODg3*_ga_WH2QY8WWF5*MTczMjc5NTAxMS44NTYuMS4xNzMyNzk4MzIxLjM1LjAuMA..#named-ports)

### 5. VMにリクエストを流すLoad Balancingの構築

Private Service Connectのサービス アタッチメントは、単一ロードバランサの転送ルール（forwarding rule）に関連付ける必要があります。

そのため、VMにリクエストを流すためのロードバランサの構築をします。

Google Cloudには、主に以下の種類の内部ロードバランサが存在しています。

- 内部パススルー ネットワーク ロードバランサ
- リージョン内部アプリケーション ロードバランサ
- 内部プロトコル転送
- リージョン内部プロキシ ネットワーク ロードバランサ

今回はバックエンドのVMで実行しているApacheはHTTPサーバーなので、リージョン内部アプリケーション ロードバランサ（Regional internal Application Load Balancer）を使用します。


まずは、VPC内に`us-central1`リージョンのプロキシ専用サブネットを作成します。

内部アプリケーション ロードバランサでは、使用する VPC ネットワークのリージョンごとにプロキシ専用サブネットを 1 つ作成する必要があるからです。

ロードバランサからバックエンド VM に送信されるパケットには、プロキシ専用サブネットからの送信元 IP アドレスが含まれます。

参考: [プロキシ専用サブネットの作成](https://cloud.google.com/load-balancing/docs/l7-internal/proxy-only-subnets?hl=ja#proxy_only_subnet_create)
参考: [プロキシ専用サブネットとロードバランサのアーキテクチャの関係](https://cloud.google.com/load-balancing/docs/proxy-only-subnets?hl=ja&_gl=1*150qnr3*_ga*MjA1MDI4ODk3NS4xNjk4MTI1ODg3*_ga_WH2QY8WWF5*MTczMjc5NTAxMS44NTYuMS4xNzMyODAxMDQ4LjI2LjAuMA..#proxy_only_subnet_in_context)

```shell
gcloud compute networks subnets create subnet-for-only-lb-proxy \
  --purpose=REGIONAL_MANAGED_PROXY \
  --role=ACTIVE \
  --project=sample123 \
  --range=10.129.0.0/23 \
  --network=vpc-psc-producer \
  --region=us-central1
```

次に、ネットワーク内のプロキシ専用サブネット トラフィック フローを許可するファイアウォール ルールを作成します。

このルールを作成することで、LBからVMへのアクセスが許可されます。

これらのファイアウォール ルールがない場合は、[デフォルトの上り（内向き）拒否ルール](https://cloud.google.com/firewall/docs/firewalls?hl=ja#default_firewall_rules)によってバックエンド インスタンスへの受信トラフィックがブロックされます。

参考: [ファイアウォール ルールを構成する](https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal?hl=ja#configure_firewall_rules)

```shell
gcloud compute firewall-rules create fw-allow-proxies \
  --project=sample123 \
  --network=vpc-psc-producer \
  --action=allow \
  --direction=ingress \
  --target-tags=vm-psc-producer-tag \
  --source-ranges=10.129.0.0/23 \
  --rules=tcp
```

さらに、ヘルスチェックからのアクセスも許可します。

ヘルスチェックのソースIPはロードバランサの種類、トラフィック タイプ、ヘルスチェックのタイプによって異なるので注意しましょう。

参考: [プローブ IP 範囲とファイアウォール ルール](https://cloud.google.com/load-balancing/docs/health-check-concepts?hl=ja#ip-ranges)

```shell
gcloud compute firewall-rules create fw-allow-health-check \
    --project=sample123 \
    --network=vpc-psc-producer \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --target-tags=vm-psc-producer-tag \
    --rules=tcp
```

次に、ロードバランサの IP アドレスを予約します。

デフォルトでは、転送ルール（forwarding rule）に 1 つの IP アドレスが使用されます。

共有 IP アドレスを予約すると、複数の転送ルールで同じ IP アドレスを使用できます。ただし、Private Service Connect を使用してロードバランサを公開する場合は、転送ルールに共有 IP アドレスを使用することはできません。

転送ルール の IP アドレス割り当てには、`subnet-vpc-psc-producer`（インスタンスを配置しているサブネット） を使用します。プロキシ専用サブネットを使用すると転送ルールの作成に失敗するので注意しましょう。

```shell
gcloud compute addresses create ip-address-for-forwarding-rule \
  --project=sample123 \
  --region=us-central1 \
  --subnet=subnet-vpc-psc-producer
```

次に、実際にロードバランサを構築していきます。作成するコンポーネントは以下の通りです。

- HTTP ヘルスチェック
- バックエンド サービス
- URL マップ
- ターゲット プロキシ
- 転送ルール

以下のgcloudコマンドをそれぞれ実行します。

```shell
// HTTP ヘルスチェックを作成
gcloud compute health-checks create http backend-service-health-check \
  --project=sample123 \
  --region=us-central1 \
  --use-serving-port

// バックエンドサービス
gcloud compute backend-services create lb-backend-service \
  --project=sample123 \
  --region=us-central1 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --health-checks=backend-service-health-check \
  --health-checks-region=us-central1 \
  --protocol=HTTP

// バックエンド サービスに名前付きポートを参照する設定を追加
gcloud compute backend-services update lb-backend-service \
  --project=sample123 \
  --region=us-central1 \
  --port-name=vm-psc-producer-instance-group-port

// バックエンド サービスにバックエンド（インスタンスグループ）を追加
gcloud compute backend-services add-backend lb-backend-service \
  --project=sample123 \
  --balancing-mode=UTILIZATION \
  --instance-group=vm-psc-producer-instance-group \
  --instance-group-zone=us-central1-a \
  --region=us-central1

// URL マップを作成
// リージョン URL マップはリクエストの URL を解析し、リクエスト URL のホストとパスに基づいて特定のバックエンド サービスにリクエストを転送する
gcloud compute url-maps create psc-lb-url-map \
  --project=sample123 \
  --default-service=lb-backend-service \
  --region=us-central1

// リージョン ターゲット HTTP プロキシ
// ユーザーからリクエストを受け取り、URL マップに転送する
gcloud compute target-http-proxies create psc-lb-target-proxy \
  --project=sample123 \
  --url-map=psc-lb-url-map \
  --url-map-region=us-central1 \
  --region=us-central1

// 転送ルール（forwarding rule）
// 受信リクエストをターゲット プロキシに転送する
gcloud compute forwarding-rules create psc-lb-forwarding-rule \
  --project=sample123 \
  --load-balancing-scheme=INTERNAL_MANAGED \
  --network=vpc-psc-producer \
  --subnet=subnet-vpc-psc-producer \
  --address=ip-address-for-forwarding-rule \
  --ports=80 \
  --region=us-central1 \
  --target-http-proxy=psc-lb-target-proxy \
  --target-http-proxy-region=us-central1
```

以上でロードバランサを構成する全ての要素が完成します。

[参考資料]

- [VM インスタンス グループのバックエンドを使用してリージョン内部アプリケーション ロードバランサを設定する](https://cloud.google.com/load-balancing/docs/l7-internal/setting-up-l7-internal?hl=ja)
- [External Application Load Balancer (外部アプリケーションロードバランサ) を徹底解説！](https://blog.g-gen.co.jp/entry/external-https-load-balancing-explained)
- [クロスリージョン内部アプリケーションロードバランサを試してみた](https://blog.g-gen.co.jp/entry/working-with-cross-region-internal-application-load-balancer)
- [GoogleマネージドSSL/TLS証明書でWebサーバーを構築してみた](https://blog.g-gen.co.jp/entry/web-server-with-google-managed-ssl-certs)
- [Private Service Connect でサービスを公開して使用する](https://codelabs.developers.google.com/cloudnet-psc-ilb?hl=ja#9)
- [Private Service Connect を使用してサービスを公開する](https://cloud.google.com/vpc/docs/configure-private-service-connect-producer?hl=ja)
- [バックエンド経由で公開サービスにアクセスする](https://cloud.google.com/vpc/docs/configure-private-service-connect-services-controls?hl=ja)

### 6. Private Service Connect 用のサブネットを作成

Private Service Connect で使用する専用サブネットを作成します。

サービスのロードバランサと同じリージョンに作成する必要がある点に注意しましょう。

```shell
gcloud compute networks subnets create subnet-for-only-psc \
  --project=sample123 \
  --network=vpc-psc-producer \
  --region=us-central1 \
  --stack-type=IPV4_ONLY \
  --range=192.168.0.0/24 \
  --purpose=PRIVATE_SERVICE_CONNECT
```

参考: [Private Service Connect 用のサブネットを作成する](https://cloud.google.com/vpc/docs/configure-private-service-connect-producer?hl=ja#add-subnet-psc)

### 7. コンシューマーVPCネットワークの作成

次に、コンシューマー側のリソースを作成していきます。

まずはVPCです。

```shell
gcloud compute networks create vpc-psc-consumer \
  --project=sample123 \
  --subnet-mode=custom
```

### 8. コンシューマー側でインスタンスを配置するためのサブネットを作成

コンシューマー側でVMインスタンスを配置するためのサブネットを作成します。

```shell
gcloud compute networks subnets create subnet-vpc-psc-consumer \
  --project=sample123 \
  --range=10.0.0.0/29 \
  --network=vpc-psc-consumer \
  --region=us-central1
```

### 9. 作成したサブネット内にコンシューマー側のサービス（VMインスタンス）を作成

以下のコマンドで、コンシューマー側のVMインスタンスを作成します。

```shell
gcloud compute instances create vm-psc-consumer \
    --project=sample123 \
    --machine-type=e2-micro \
    --tags=allow-ssh \
    --subnet=subnet-vpc-psc-consumer \
    --zone=us-central1-a \
    --private-network-ip=10.0.0.2 \
    --no-address
```

次に、このインスタンスに対してSSHで接続するためのファイアウォールルールを設定します。

```shell
// VM との SSH 接続を許可する ファイアウォール ルール
// source-ranges を省略すると、Google Cloud は任意の送信元としてルールを解釈する
gcloud compute firewall-rules create fw-allow-ssh \
  --project=sample123 \
  --network=vpc-psc-consumer \
  --action=allow \
  --direction=ingress \
  --target-tags=allow-ssh \
  --rules=tcp:22
```

### 10. サービス アタッチメントを使ってサービスを公開する

ここからは、Private Service Connectを使ってプロデューサー側のサービスをコンシューマー側に公開します。

サービスを公開するには、サービスのロードバランサと同じリージョンにサービス アタッチメントを作成します。

サービスは、以下のいずれかの方法で使用可能にできます。

- プロジェクトの自動承認でサービスを公開する
- 明示的なプロジェクト承認でサービスを公開する

参考: [サービスを公開する](https://cloud.google.com/vpc/docs/configure-private-service-connect-producer?hl=ja#publish-service)

今回は、`明示的なプロジェクト承認`でサービスを公開します。

各サービス アタッチメントには、コンシューマ承認リストとコンシューマ拒否リストがあります。

このリストにより、サービスに接続できるエンドポイントが決まります。

特定のサービス アタッチメントは、これらのリストにあるプロジェクトまたはネットワークのいずれかを使用できますが、両方を使用することはできません。

```shell
// コンシューマーVPCの詳細情報を表示
// selfLink:のところにNetwork URIが表示される（サービスアタッチメントの作成で、接続を許可するコンシューマー側のVPCを指定するために使う）
gcloud compute networks describe vpc-psc-consumer --project=sample123
```

```shell
// サービスアタッチメントを作成
// コンソールでいうとPrivate Service Connectの「公開サービス」にリソースが作成される
gcloud compute service-attachments create psc-service-attachment \
  --project=sample123 \
  --region=us-central1 \
  --producer-forwarding-rule=psc-lb-forwarding-rule \
  --connection-preference=ACCEPT_MANUAL \
  --consumer-accept-list=projects/myProjectId1/global/networks/vpc-psc-consumer=5 \
  --nat-subnets=subnet-for-only-psc
````

### 11. エンドポイントを通じて公開サービスにアクセスする

エンドポイントは、Private Service Connect 転送ルールを使用して、別の VPC ネットワーク内のサービスに接続するために利用します。

Private Service Connect を使用して別の VPC ネットワーク内のサービスに接続する場合は、VPC ネットワーク内の通常のサブネットから内部 IP アドレスを選択します。

このIP アドレスは、サービス プロデューサーのサービス アタッチメントと同じリージョン（us-central1）に存在する必要があります。

また、デフォルトでは、エンドポイントと同じリージョンと VPC ネットワーク（または共有 VPC ネットワーク）にあるクライアントのみがエンドポイントにアクセスできます。エンドポイントを他のリージョンで利用可能にするには、[グローバル アクセス](https://cloud.google.com/vpc/docs/about-accessing-vpc-hosted-services-endpoints?hl=ja#global-access)を利用します。（今回は利用しません）

```shell
// エンドポイントに割り当てる内部IPアドレスを予約するためのサブネットを作成（サービス アタッチメントと同じリージョン）
gcloud compute networks subnets create subnet-for-endpoint \
  --project=sample123 \
  --range=192.168.0.0/24 \
  --network=vpc-psc-consumer \
  --region=us-central1

// エンドポイントに割り当てる内部 IP アドレスを予約
gcloud compute addresses create ip-address-for-endpoint \
  --project=sample123 \
  --region=us-central1 \
  --subnet=subnet-for-endpoint \
  --ip-version=IPV4
```

サービス プロデューサーのサービス アタッチメントに接続するためのエンドポイントを作成します。

エンドポイントの実態は、先程のロードバランサのコンポーネントの一つである転送ルール（forwarding rule）です。

```shell
// 以下のコマンドを実行すると、コンソールのPrivate Service Connectの「接続エンドポイント」のところにエンドポイントが追加される
// 同一リージョンからのアクセスのみを許可するエンドポイント
gcloud compute forwarding-rules create psc-endpoint-forwarding-rule \
  --project=sample123 \
  --region=us-central1 \
  --network=vpc-psc-consumer \
  --address=ip-address-for-endpoint \
  --target-service-attachment=psc-service-attachment
```

参考: [エンドポイントを作成する](https://cloud.google.com/vpc/docs/configure-private-service-connect-services?hl=ja#create-endpoint)

### 12. 検証

sshでコンシューマー側のCompute Engineインスタンスに入ります。

そして、curlでエンドポイントのIPにリクエストを送ります。

```shell
// エンドポイントのIPに書き換えてください
curl ip-address-for-endpoint
```

プロデューサー側のインスタンスからレスポンスが返ってきたら動作確認は完了です。

Private Service Connectを使って内部通信を実現することができました。

## 参考資料

- [Private Service Connect でサービスを公開して使用する](https://codelabs.developers.google.com/cloudnet-psc-ilb?hl=ja#0)
- [Private Service Connect を使用してサービスを公開する](https://cloud.google.com/vpc/docs/configure-private-service-connect-producer?hl=ja)
- [エンドポイントを通じて公開サービスにアクセスする](https://cloud.google.com/vpc/docs/configure-private-service-connect-services?hl=ja)
- [Private Service Connect でマネージドサービスを公開する](https://blog.g-gen.co.jp/entry/managed-service-with-private-service-connect)
- [Private Service Connect経由でプライベート接続できているか確認してみた](https://blog.g-gen.co.jp/entry/check-access-to-private-service-connect)
