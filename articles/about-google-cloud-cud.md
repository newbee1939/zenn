---
title: "Google Cloudの確約利用割引(CUD)について"
emoji: "🐷"
type: "tech"
topics:
  - "インフラ"
  - "googlecloud"
  - "cud"
  - "確約利用割引"
published: true
published_at: "2024-12-09 22:59"
---

## Google Cloudの確約利用割引(CUD)の概要

確約利用割引（CUD）は、**特定の期間に指定したリソースを使用することの確約（コミットメント）と引き換えに、割引料金が適用されるGoogle Cloudの仕組み**です。

要は「〜年の間、〇〇いうサービスに毎月n円分を支払う」という約束をする代わりに、いくらか料金を割り引いてくれるということです。

確約利用割引は、大きく分けて以下の2種類に分かれます。

- **費用ベースの割引**
- **リソースベースの割引**

Compute Engine以外は費用ベースの割引のみ利用でき、使用する Google Cloud サービスごとに購入する必要があります。（e.g. Dataflow, Spanner, Cloud SQL...）

費用ベースの割引は、Cloud 請求先アカウントで支払いが行われているすべてのプロジェクトで、対象となる使用量に適用されます。

Compute Engineの場合は**リソースベースの割引**と**Compute フレキシブル CUD**（費用ベースの割引の別名と思ってよい）を選択できます。

リソースベースの割引とCompute フレキシブル CUDは、割引率や適用範囲が異なっています。

[参考: 1年間の割引率]
- リソースベースCUD: 37%
- フレキシブルCUD: 28%

どちらか片方しか使えないわけではないので、状況に合わせて使い分けるのが良いでしょう。

## Compute フレキシブル CUDについて

Compute フレキシブル CUDは、最近対象サービスが増えました。

元々はCompute Engineのみだったのですが、**1 回の CUD 購入で、Compute Engine、GKE(Standard, Autopilot)、Cloud Run の 3 つのプロダクトすべてで対象となる費用をカバー**できるようになりました。（名前もCompute EngineのフレキシブルCUDからCompute フレキシブル CUDに変更されています）

参考: [フレキシブル確約利用割引がさらに柔軟に](https://cloud.google.com/blog/ja/products/containers-kubernetes/compute-flexible-cud-expands-to-gke-autopilot-and-cloud-run/)

これまではGKEとCloud Runで別々に費用ベースの割引を購入する必要があったのですが、Compute フレキシブルCUDを使えば、これらをまとめて購入することができます。


なるべくCUDの管理を複雑にしたくない人には良い選択肢となるでしょう。


## CUDは購入のタイミングに注意

確約利用割引は、購入後少ししてから有効となり、月中に購入しても過去に遡って適用はされません。

例えば、仮に4月に1年契約で購入する場合、4月1日に買っても4月30日に買っても契約（割引）期間は「購入時点〜翌年3月31日」となります。

つまり、毎月$100支払うようなコミットメントを1年契約するとして、購入したのが1日だろうが30日だろうが翌月に$100、年間で$1200支払うことに変わりはないということです。

そのため、30日に購入すると$100の大部分が無駄になってしまいます。

CUDを購入する際は、購入するタイミングに注意しましょう。

## まとめ

- 確約利用割引（CUD）は、特定の期間に指定したリソースを使用することの確約（コミットメント）と引き換えに、割引料金が適用されるGoogle Cloudの仕組み
- Compute Engine以外は 費用ベースの割引 のみ利用できる
  - 使用する Google Cloud サービスごとに購入する必要がある（e.g. Dataflow, Spanner, Cloud SQL ..）
- Compute Engineは リソースベースの割引 と Compute フレキシブル CUD（費用ベースの割引） を選択できる
  - リソースベースの方が割引率は高い
  - Compute フレキシブル CUD は最近対象サービスが増加
  - 1 回の Compute Engineのフレキシブル CUD の 購入で、Compute Engine、GKE(Standard, Autopilot)、Cloud Run の 3 つのプロダクトをカバーできるようになった
- CUDは購入するタイミングに注意

## 参考資料

- [Google Cloudの利用費を節約できる確約利用割引を購入してみた
](https://zenn.dev/team_zenn/articles/zenn-used-cud)
- [確約利用割引](https://cloud.google.com/docs/cuds?hl=ja)
- [確約利用割引の料金とクレジットのアトリビューション](https://cloud.google.com/docs/cuds-attribution?hl=ja)
- [確約利用割引 Recommender](https://cloud.google.com/docs/cuds-recommender?hl=ja)
- [費用ベースの確約利用割引](https://cloud.google.com/docs/cuds-spend-based?hl=ja)
- [Google Cloud「確約利用割引（CUDs）」導入記（Cloud SQL / Compute Engine）](https://zenn.dev/ptna/articles/36ceb256dc32fc)
- [Google Cloud Platformの利用料金を抑えるための確約利用割引について](https://zenn.dev/rescuenow/articles/6c1da4155e8414)
- [Compute Engine の確約利用割引（CUD）](https://cloud.google.com/compute/docs/instances/committed-use-discounts-overview?hl=ja#spend_based)
- [Google Kubernetes Engine (GKE)の確約利用割引](https://cloud.google.com/kubernetes-engine/cud?hl=ja)
- [Cloud Runの確約利用割引](https://cloud.google.com/run/cud)
- [GCPで確約利用割引を購入してみた](https://symphonict.nesic.co.jp/tech-blog/527/)
- [GCPの確約利用割引が難しい](https://n-s.tokyo/2022/03/gcp-cud-tips/)
- [確約利用割引(Compute Engine)を解説。AWSとの違いも確認](https://blog.g-gen.co.jp/entry/committed-use-discounts-explained)
- [Compute フレキシブル CUD](https://cloud.google.com/compute/docs/instances/committed-use-discounts-overview?hl=ja#spend_based)
