---
title: "Cloud Runを爆速で切り戻す💨"
emoji: "💨"
type: "tech"
topics:
  - "cloudrun"
  - "googlecloud"
  - "cloud"
  - "クラウド"
published: true
published_at: "2025-02-26 17:05"
---

## 従来の切り戻し作業の課題

従来の切り戻し作業には以下のような手順が必要で、多くの時間を要します。
その結果として、ユーザーに多大な影響を及ぼしてしまいます。

1. 原因箇所の特定
2. コードの修正または git revert
3. PR 作成（自動テスト含む）
4. レビュー実施
5. 再デプロイ（イメージのビルド、プッシュ、デプロイ）

## Cloud Run のリビジョンによる解決策

この問題は、Cloud Run の**リビジョン**機能を活用することで解決できます。

### リビジョンとは

デプロイ履歴を保持する機能です。

デプロイごとに自動で作成され、**各リビジョンに対してトラフィックの振り分けが可能**です。

![](https://storage.googleapis.com/zenn-user-upload/7da439a828e2-20250226.png)

### Cloud Run のリビジョンを使用した切り戻し

**Cloud Run のトラフィック振り分け機能を使用して以前のリビジョンにトラフィックを転送する**ことで、**簡単かつ迅速に切り戻しを実現**できます。

この方法には以下のメリットがあります。

- コードの revert や再デプロイが不要
- 迅速な切り戻しが可能
- 切り戻し後に落ち着いて原因調査や修正が可能

## 切り戻し方法の選択肢

リビジョンを用いた切り戻し方法には以下の 2 つがあります。

1. **コンソールからの手動実施**

   - 操作ミスのリスクがある

2. **GitHub Actions からの実施**
   - より迅速な対応が可能
   - 操作ミスのリスクを軽減できる

今回は、より迅速かつ安全に切り戻しを実現するため、GitHub Actions のワークフローを使用した方法を紹介します。

## Cloud Run の切り戻しを行うワークフローの実装例

以下の 2 種類のワークフローを実装します。

- 切り戻し用ワークフロー
- デプロイ後のリビジョン管理用ステップ

### 切り戻し用ワークフロー

切り戻しを実行するための GitHub Actions ワークフローは以下のように実装できます。

リビジョン名を指定しない場合、最新のリビジョンの 1 つ前のリビジョンに全てのトラフィックを転送します。

リビジョン名を指定した場合は、指定したリビジョンにトラフィックを流します。

```yaml
name: Rollback Cloud Run Service

on:
  workflow_dispatch:
    inputs:
      revision_name:
        description: >-
          Target revision name for rollback (e.g. my-service-00001-abc).
          If not specified, the previous revision will be used by default.
        required: false
        type: string

env:
  SERVICE_NAME: my-service
  REGION: asia-northeast1
  PROJECT_ID: my-project-id
  WORKLOAD_IDENTITY_PROVIDER: projects/123456789012/locations/global/workloadIdentityPools/my-pool/providers/my-provider
  SERVICE_ACCOUNT: my-service-account@my-project-id.iam.gserviceaccount.com

jobs:
  rollback:
    runs-on: ubuntu-24.04
    permissions:
      contents: "read"
      id-token: "write"

    steps:
      - uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v2
        with:
          project_id: ${{ env.PROJECT_ID }}
          workload_identity_provider: ${{ env.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ env.SERVICE_ACCOUNT }}

      - name: Get previous revision
        if: inputs.revision_name == ''
        id: get-previous
        run: |
          REVISIONS=$(gcloud run revisions list \
            --service=${{ env.SERVICE_NAME }} \
            --region=${{ env.REGION }} \
            --project=${{ env.PROJECT_ID }} \
            --format='value(metadata.name)' \
            --sort-by='~metadata.creationTimestamp' \
            --limit=2)

          PREVIOUS_REVISION=$(echo "$REVISIONS" | sed -n '2p')
          echo "previous_revision=$PREVIOUS_REVISION" >> $GITHUB_OUTPUT

      - name: Execute rollback
        run: |
          REVISION_TO_USE="${{ inputs.revision_name }}"
          if [ -z "$REVISION_TO_USE" ]; then
            REVISION_TO_USE="${{ steps.get-previous.outputs.previous_revision }}"
          fi

          gcloud run services update-traffic ${{ env.SERVICE_NAME }} \
            --region=${{ env.REGION }} \
            --project=${{ env.PROJECT_ID }} \
            --to-revisions=$REVISION_TO_USE=100
```

### デプロイ後のリビジョン管理

デプロイワークフローには、以下のステップを追加します。これにより、最新のリビジョンに全てのトラフィックが転送されます。

このステップが無い場合、トラフィックは切り替わりませんのでご注意ください。

```yaml
# デプロイ後のステップ
- name: Update Traffic To Latest
  run: |
    gcloud run services update-traffic ${{ env.SERVICE_NAME }} \
      --to-latest \
      --region=${{ env.REGION }} \
      --project=${{ env.PROJECT_ID }}
```

## まとめ

Cloud Run のリビジョン機能を活用することで、迅速かつ安全な切り戻しが実現可能です。
