---
title: "Dependabotのパッチ更新を自動マージするGitHub Actionsワークフロー"
emoji: "🐙"
type: "tech"
topics:
  - "github"
  - "githubactions"
  - "actions"
  - "cicd"
  - "dependabot"
published: true
published_at: "2025-12-03 09:17"
---

## はじめに

Dependabotは依存関係の更新を自動で検出してプルリクエストを作成してくれますが、毎回手動でマージするのは手間がかかります。

特にパッチ更新（セキュリティ修正やバグ修正）は、影響範囲が限定的で自動マージしても問題ないことが多いです。

この記事では、Dependabotのパッチ更新を自動的にマージするGitHub Actionsワークフローの実装方法を解説します。

## ワークフローの全体像

以下のワークフローは、Dependabotが作成したプルリクエストのうち、パッチ更新（`version-update:semver-patch`）のみを自動的にマージします。

```yaml
name: Dependabot auto-merge

on:
  pull_request_target:
    types:
      - opened
      - synchronize
      - reopened
      - ready_for_review

permissions: {}

defaults:
  run:
    shell: bash

jobs:
  dependabot:
    runs-on: ubuntu-24.04
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    permissions:
      contents: write
      pull-requests: write
    steps:
      - name: Fetch Dependabot metadata
        id: metadata
        uses: dependabot/fetch-metadata@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}

      - name: Auto-merge Dependabot patch updates
        if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --merge --auto "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 各ステップの詳細解説

### トリガー設定

```yaml
on:
  pull_request_target:
    types:
      - opened
      - synchronize
      - reopened
      - ready_for_review
```

- `pull_request_target`: プルリクエストが作成されたブランチのコンテキストで実行されます。これにより、Dependabotのプルリクエストに対して適切な権限でアクセスできます
- `opened`: プルリクエストが作成されたとき
- `synchronize`: プルリクエストに新しいコミットがプッシュされたとき
- `reopened`: 閉じられたプルリクエストが再オープンされたとき
- `ready_for_review`: ドラフトからレビュー可能な状態になったとき

### ジョブの条件分岐

```yaml
if: github.event.pull_request.user.login == 'dependabot[bot]'
```

この条件により、Dependabotが作成したプルリクエストの場合のみジョブが実行されます。他のユーザーが作成したプルリクエストでは実行されないため、誤って自動マージされることを防げます。

### 権限設定

```yaml
permissions:
  contents: write
  pull-requests: write
```

- `contents: write`: リポジトリへの書き込み権限（マージに必要）
- `pull-requests: write`: プルリクエストの操作権限（マージに必要）

### ステップ1: Dependabotメタデータの取得

```yaml
- name: Fetch Dependabot metadata
  id: metadata
  uses: dependabot/fetch-metadata@v2
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

`dependabot/fetch-metadata@v2`アクションを使用して、Dependabotのプルリクエストに関するメタデータを取得します。このアクションは以下のような情報を出力します：

- `update-type`: 更新の種類（`version-update:semver-patch`, `version-update:semver-minor`, `version-update:semver-major`など）
- `dependency-names`: 更新される依存関係の名前
- `directory`: 更新が発生したディレクトリ

### ステップ2: パッチ更新の自動マージ

```yaml
- name: Auto-merge Dependabot patch updates
  if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
  run: gh pr merge --merge --auto "$PR_URL"
  env:
    PR_URL: ${{ github.event.pull_request.html_url }}
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- `if`条件: 更新タイプがパッチ更新（`version-update:semver-patch`）の場合のみ実行
- `gh pr merge --merge --auto`: GitHub CLIを使用してプルリクエストをマージします
  - `--merge`: マージコミットを作成してマージ
  - `--auto`: すべてのチェックが成功したら自動的にマージ

## セットアップ方法

### 1. ワークフローファイルの作成

`.github/workflows/dependabot-auto-merge.yml` に上記のワークフローを保存します。

### 2. Dependabotの設定確認

`dependabot.yml` または GitHubの設定でDependabotが有効になっていることを確認します。

## 注意点とベストプラクティス

### パッチ更新のみを自動マージする理由

- **パッチ更新（1.0.0 → 1.0.1）**: バグ修正やセキュリティパッチ。破壊的変更がないため、自動マージが安全
- **マイナー更新（1.0.0 → 1.1.0）**: 新機能の追加。影響範囲が大きい可能性があるため、レビューが必要
- **メジャー更新（1.0.0 → 2.0.0）**: 破壊的変更を含む可能性が高い。必ずレビューが必要

## まとめ

このワークフローを導入することで、Dependabotのパッチ更新を自動的にマージでき、セキュリティパッチやバグ修正を迅速に適用できます。パッチ更新は破壊的変更を含まないことが多いため、自動マージしても安全です。

ただし、プロジェクトの特性やチームの方針に応じて、自動マージの条件を調整することをおすすめします。特に重要な依存関係については、手動レビューを必須にするなどのカスタマイズを検討してください。
