---
title: "GCE経由でMemorystore for RedisにSSHポートフォワーディングする方法"
emoji: "🔐"
type: "tech"
topics:
  - "redis"
  - "tips"
  - "ssh"
  - "googlecloud"
  - "sre"
published: true
published_at: "2025-11-13 22:29"
---

## はじめに

Google Cloud の Memorystore for Redis は、プライベート IP でのみアクセス可能なマネージド Redis サービスです。そのため、異なる VPC やローカル環境から直接接続することはできません。

このような場合、GCE インスタンスを踏み台サーバーとして SSH ポートフォワーディングを利用することで、安全に接続できます。SSH ポートフォワーディングとは、SSH 接続を使ってネットワーク通信を中継する仕組みで、暗号化された経路を通じて安全にデータをやり取りできます。

本記事では、GCE インスタンスを経由して Memorystore for Redis に接続する方法を解説します。

## アーキテクチャ概要

![](https://storage.googleapis.com/zenn-user-upload/d3324295126c-20251113.png)
_Compute Engine の Public IP 経由で Memorystore for Redis に接続_

## 手順

### 1. Memorystore for Redis インスタンスの作成

まずは Redis を配置する VPC（memorystore VPC）を作成します。

```bash
gcloud compute networks create memorystore \
    --subnet-mode=auto \
    --project=prj-sample
```

次に、この VPC に紐付ける形で Memorystore for Redis インスタンスを作成します。

```bash
gcloud redis instances create redis-instance \
    --region=us-central1 \
    --network=memorystore \
    --project=prj-sample
```

### 2. GCE インスタンスの作成

memorystore VPC 内に SSH ポートフォワーディング用の GCE インスタンスを作成します。

```bash
gcloud compute instances create redis-jump-server \
  --network=memorystore \
  --project=prj-sample \
  --tags=redis-proxy \
  --zone=us-central1-a \
  --machine-type=e2-micro
```

### 3. GCE インスタンスに接続してユーザー作成

GCE インスタンスに対して ssh 接続を許可するためのファイアウォールルールを追加します。

```bash
gcloud compute firewall-rules create fw-allow-ssh \
  --project=prj-sample \
  --network=memorystore \
  --action=allow \
  --direction=ingress \
  --target-tags=redis-proxy \
  --rules=tcp:22
```

ssh で GCE インスタンスに接続します。

```bash
gcloud compute ssh redis-jump-server \
  --zone=us-central1-a \
  --project=prj-sample
```

インスタンスに接続したら、SSH ポートフォワーディング専用のユーザーを作成します。

```bash
sudo adduser redis-proxy
```

ユーザーを切り替えます。

```bash
su redis-proxy
```

SSH 接続用の秘密鍵と公開鍵のペアを作成します。プロンプトが表示されたら、Enter キーを押してデフォルト設定で進めます。

```bash
cd
ssh-keygen
```

以下のように `.ssh` ディレクトリ配下に鍵が作成されます。

![](https://storage.googleapis.com/zenn-user-upload/0409b538dfc4-20251112.png)

公開鍵を `authorized_keys` に追加します。これにより、この秘密鍵を使った SSH 接続が許可されます。

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 不要になった公開鍵ファイルを削除
rm ~/.ssh/id_rsa.pub
```

次に、`~/.ssh/config` に以下の設定を追加します。これにより、ローカルホスト（自分自身）への SSH 接続を簡単に行えるようにします。

```
Host redis-jump-server
HostName 127.0.0.1
User redis-proxy
Port 22
IdentityFile ~/.ssh/id_rsa
```

- `HostName 127.0.0.1`: 自分自身（ローカルホスト）を指定
- `IdentityFile`: 先ほど作成した秘密鍵を指定

この状態で、VM 内から SSH ログインできるか確認します。

```bash
ssh redis-jump-server

# 確認できたら抜ける（VM ログイン時のユーザーに戻す）
exit
```

### 4. SSH ポートフォワーディングの設定

ここからが本題の SSH ポートフォワーディングの設定です。systemd を使ってコマンドをサービス化することで、VM 再起動時にも自動的にポートフォワーディングが起動するようにします。

これにより、安定した運用が可能になります。

`sudo vim /lib/systemd/system/redis-proxy.service` で編集画面を開き、以下の内容を入力します。

**重要:**

- `xx.xx.xx.xx` には Memorystore for Redis の Private IP を指定してください
- VM 側のポートには任意の値を指定（以下の例では 7379）

```
[Unit]
Description=Memorystore Proxy
After=network.target

[Service]
User=redis-proxy
ExecStart=ssh -N -o ServerAliveInterval=60 -o ExitOnForwardFailure=yes -L 0.0.0.0:7379:xx.xx.xx.xx:6379 redis-jump-server

RestartSec=3
Restart=always

[Install]
WantedBy=multi-user.target
```

**設定の説明:**

- `-L 0.0.0.0:7379:xx.xx.xx.xx:6379`: VM の 7379 ポートを Memorystore の 6379 ポートに転送
- `ServerAliveInterval=60`: 60 秒ごとに Keep Alive を送信して接続を維持
- `Restart=always`: サービスが停止した場合に自動的に再起動

以下のコマンドを実行してサービスを起動します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable redis-proxy
sudo service redis-proxy start
```

サービスの起動状態を確認します。

```bash
systemctl status redis-proxy
```

`Active: active (running)` と表示されていれば OK です。

![](https://storage.googleapis.com/zenn-user-upload/ae23befe4d36-20251113.png)

もし起動できていない場合は、以下のコマンドでデバッグしましょう。

```bash
# ポート 7379 が解放されているか確認
ss -anp | grep 7379

# systemd サービスのログを確認
journalctl -xeu redis-proxy.service
```

### 5. ローカル環境から GCE 経由で Memorystore for Redis に接続する

外部から VM の 7379 ポートに接続できるよう、ファイアウォールルールを作成します。

```bash
gcloud compute firewall-rules create fw-allow-redis-proxy \
  --project=prj-sample \
  --network=memorystore \
  --action=allow \
  --direction=ingress \
  --target-tags=redis-proxy \
  --rules=tcp:7379
```

- `--target-tags=redis-proxy`: VM 作成時に指定したタグに対してルールを適用
- `--rules=tcp:7379`: TCP の 7379 ポートを許可

最後に、ローカル環境から `redis-cli` を使って接続確認をします。

※`oo.oo.oo.oo` は VM の Public IP に置き換えてください

```bash
redis-cli -h oo.oo.oo.oo -p 7379
```

接続に成功すると、Redis のプロンプトが表示され、通常の Redis コマンドが実行できます。

## まとめ

本記事では、GCE インスタンスを踏み台サーバーとして使用し、SSH ポートフォワーディング経由で Memorystore for Redis に接続する方法を解説しました。

**主なポイント:**

- プライベート IP のみの Memorystore for Redis に外部からアクセス可能
- systemd によるサービス化で安定した運用を実現
- SSH による暗号化された安全な通信

この方法により、異なる VPC 間やプロジェクト間、ローカル開発環境からでも安全に Memorystore for Redis へアクセスできるようになります。

少しでも同じような作業をしたいと思っている方の参考になれば幸いです。

## 参考文献

- [SSH ポートフォワーディングを使ったサーバー間接続](https://support.genbasupport.com/techblog/topics-68457/)
- [ssh ポートフォワーディング とは](https://qiita.com/miyuki_samitani/items/07c8b05f74dca25b33d0)
- [犬でも分かる ssh ポートフォワーディング](https://runningdog.mond.jp/blog/2024/09/15/ssh-portforwarding/#toc5)
- [Systemd で SSH の LocalForward をサービス化](https://qiita.com/_mmasaki/items/0904c6d3cd1b6b021a53)
- [ポート転送](https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/6/html/deployment_guide/s2-ssh-beyondshell-tcpip)
