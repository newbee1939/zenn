---
title: "【GitHub Actions】ワークフローからPull Requestにlabelを付与する"
emoji: "💡"
type: "tech"
topics:
  - "github"
  - "githubactions"
  - "tips"
  - "cicd"
published: true
published_at: "2024-09-30 19:55"
---

GitHub ActionsでワークフローからPull Requestにlabelを付与する方法のメモ。

## 【GitHub Actions】ワークフローからPull Requestにlabelを付与する方法

[Labeler](https://github.com/marketplace/actions/labeler)を使う方法もありますが、単に特定のlabelを付与したいだけなら、以下のように`ghコマンド`を使った方がシンプルです。

```yml
name: Add Sample Label

on:
  pull_request_review:
    types: [submitted]

jobs:
  add-label:
    if: github.event.review.state == 'approved'
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Add 'Sample' label
        run: |
          gh pr edit ${{ github.event.pull_request.number }} --add-label 'Sample'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`gh pr edit`のドキュメントは以下になります。

https://cli.github.com/manual/gh_pr_edit

他にも実現方法はあると思いますが、一つの方法として参考になれば幸いです。
