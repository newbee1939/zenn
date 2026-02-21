---
title: "GitHub ActionsがQueuedで停止した場合に確認すべきこと"
emoji: "🔥"
type: "tech"
topics:
  - "githubactions"
  - "tips"
  - "cicd"
published: true
published_at: "2024-09-30 19:30"
---

以下のように、GitHub Acitonsのワークフローが`Queued`のまま停止した場合に確認すべきことのメモです。

![](https://storage.googleapis.com/zenn-user-upload/32e802a90d8c-20240930.png)

## GitHub ActionsがQueuedで停止した場合に確認すべきこと

基本的には、以下のいずれかのパターンだと思います。

- GitHub Actionsで障害が起きている
- 誤ったワークフローの書き方をしている

### GitHub Actionsで障害が起きている

GitHubで障害が起きている場合にQueuedのまま処理が停止することがあります。

以下のページで障害が起きていないかを確認しましょう。

https://www.githubstatus.com/

以下のページも参考になります。

https://downdetector.jp/shougai/github/

この場合は障害が解消するまで待つしかないです。

### 誤ったワークフローの書き方をしている

もう一つは誤ったワークフローの書き方をしている場合です。

大抵の場合は、ワークフローの記述方法にミスがあれば、ワークフローが失敗してFailureステータスに変わり、エラーメッセージが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/4e72d0af2bde-20240930.png)

しかし、以下の場合はエラーにならず、Queuedのまま処理が停止するようです。（他にもパターンはあるのかも知れませんが、私が知っているのはこのパターンのみです）

- **存在しないランナーを指定している場合**
    - e.g. runs-onに`ubuntu-22.04`ではなく、誤って（存在しない）`ubuntu-22.0.4`を指定

基本的に、特定のワークフローのみ失敗していて、他のワークフローは動いている場合は、GitHubの障害ではなくワークフローの記述を間違えている可能性が高いです。

ワークフローが動かなくなる前後のコードの差分を比較した上で、原因を切り分けるのが良いでしょう。

## まとめ

- GitHub ActionsがQueuedで停止した場合に確認すべきこと
    - GitHubの[ステータス](https://www.githubstatus.com/)
    - 存在しないランナーを指定していないか
