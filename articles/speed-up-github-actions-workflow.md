---
title: "すぐにできる！CIを高速化する3つの方法【GitHub Actions】"
emoji: "⚡"
type: "tech"
topics:
  - "github"
  - "githubactions"
  - "パフォーマンス"
  - "cicd"
published: true
published_at: "2024-09-30 21:06"
---

**CIの実行速度が遅くてイライラしている人**向け。

GitHub ActionsでCIの実行を高速化する方法を3つご紹介します。

## CIを高速化する3つの方法【GitHub Actions】

今回紹介するのは以下の3つの方法です。

- Jobの分割
- パッケージのキャッシュ処理を追加
- テストの分割と並列実行

### Jobの分割

**Jobは分割することで、それぞれのJobが並列に動作する**ようになります。

例えば、ユニットテストの実行とLinterの実行は、多くの場合独立して動作しても問題ないものです。

これらは一つのJobの中に直列に記述するのではなく、Jobを分けて記述した方が効率的でしょう。

```yml
jobs:
  test:
    runs-on: ubuntu-22.04
    steps:
    ...

  lint:
    runs-on: ubuntu-22.04
    steps:
    ...
```

### パッケージのキャッシュ処理を追加

**パッケージは、キャッシュしておくと時間のかかるパッケージのインストール処理をスキップできる**のでおすすめです。

公式が提供している[actions/cache](https://github.com/actions/cache)を利用して、キャッシュの処理を実装しましょう。

以下の場合は、OS、Nodeバージョン、もしくはパッケージ情報を管理しているファイル（package-lock.json）に変更がある場合のみ`npm ci`が実行され、それ以外の場合はキャッシュが利用されます。

```yml
- name: cache and restore packages
  id: cache-npm
  uses: actions/cache@v4.0.2
  with:
    path: node_modules
    key: ${{ runner.os }}-${{ steps.tool_versions.outputs.nodejs }}-${{ hashFiles('**/package-lock.json') }}

- name: install npm packages
  if: steps.cache-npm.outputs.cache-hit != 'true'
  run: npm ci
  shell: bash
```

### テストの分割と並列実行

テストの実行に時間がかかる場合は、**テストを分割した上で、それぞれを並列で実行する**ことで高速化を図れます。

例えば、Jestの場合は、GitHub Actionsの[matrix strategy](https://docs.github.com/ja/actions/writing-workflows/choosing-what-your-workflow-does/running-variations-of-jobs-in-a-workflow)とコマンドオプションの[--shard](https://jestjs.io/ja/docs/cli#--shard)を組み合わせることで、テストの分割と並列実行を簡単に実現できます。

`matrix strategy`は、一つのJob内で、変数に定義した値ごとにJobを動作させる手法で、`--shard`はテストを分割するオプションです。

これらを使って、以下のようなワークフローを定義します。

```yml
jobs:
  test:
    runs-on: ubuntu-22.04
    strategy:
      matrix:
        shard: [1/4, 2/4, 3/4, 4/4]
    steps:
      - name: checkout
        uses: actions/checkout@v3

      - name: setup environment
        uses: ./.github/actions/setup

      - name: run test
        run: npx jest --ci --shard=${{ matrix.shard }}
```

これにより、**4つのJobが並列で動作し、それぞれのJobが1/4ずつテストを実行**してくれます。

Jest以外にも`--shard`のようなオプションがあるかは分からないですが、考え方自体はどの言語にも応用できると思います。

#### ⚠️matrix strategyの注意点

あまりにJobを分割しすぎると、GitHub Actionsの課金時間（Billable time）が大幅に増加する可能性があります。

https://khasegawa.hatenablog.com/entry/2022/11/14/100000

どの程度Jobを分割するかは、ワークフローの実行時間（Total duration）だけでなく、課金時間（Billable time）も見た上で適切な値を設定するようにしましょう。

## 他にも方法はある

今回は手軽にできるCIの速度改善方法として、以下の3つを紹介しました。

- Jobの分割
- パッケージのキャッシュ処理を追加
- テストの分割と並列実行

ただ、これら以外にも[larger runner](https://docs.github.com/ja/actions/using-github-hosted-runners/using-larger-runners/about-larger-runners)を利用する方法や、変更があった部分のみテストを実行する方法など、速度改善の手法は色々とあります。

使える時間的・経済的なリソースを認識した上で、できる範囲で少しずつ改善していくのがおすすめです。
