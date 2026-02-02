---
title: "OpenTelemetry をざっくり理解する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## なぜOpenTelemetryが必要なのか

ざっくり図にまとめるとこんな感じです。

![](https://storage.googleapis.com/zenn-user-upload/9c98ba4fc130-20260115.png)

一昔前、システムは一枚岩で動作するのが基本でした。さらに、**ログ、メトリクス、トレース**（いわゆる**オブザーバビリティの3本柱**）もそれぞれ監視ツールとして独立していました。

この場合、もしシステム全体で何かしらの問題が発生したら原因を特定するのは困難です。それぞれの監視ツールを一つ一つチェックし、ボトルネックを発見するしかありません。これは干し草の中から針を見つけるようなもので、膨大な時間がかかります。

さらに、現在は**分散システム**の時代です。

こうなると、さらに原因を特定するのは困難になります。一つのシステムの中の視点だけでなく、横のシステムとの繋がりの話も出てくるからです。

そこで誕生したのが**OpenTelemetry**です。

OpenTelemetryを使うことで得られるのが、整理された**相関データ**です。相関データにより「横串」でデータを眺めることができます。

こうなればボトルネックを見つけるのが簡単になるのは想像に難くないでしょう。

さらに、OpenTelemetryは**ユニバーサルスタンダード**なツールとして浸透しています。主要なクラウドプロバイダー（Amazon, Google Cloud, Microsoft）はもちろんのこと、DatadogなどのSaaSもOpenTelemetryを前提としてシステムが作られています。

これにより、**ベンダーロックイン**が排除され、OpenTelemetryを使ってさえいれば、テレメトリーを送信するバックエンドを自由に選択できる状況になっています。

## OpenTelemetryに関連する用語

OpenTelemetryには特有の用語が存在します。まずはそれらの概要を掴んでおきましょう。

- **テレメトリー**
    - システムが何をしているかを示すデータ
- **コンテキスト**
    - **ハードコンテキスト**
        - リクエストごとの一意な識別子
        - テレメトリーを横断的に相関させるために必須の要素
        - e.g. リクエストID
    - **ソフトコンテキスト**
        - トレースやメトリクスと結びつく補助的なコンテキスト情報
        - e.g. ホスト名、タイムスタンプ
- **トレース**
    - 与えられたトランザクションに関連するログの集まり
    - ハードコンテキストによって相関づけられたテレメトリーの集まり
- **スパン**
    - 与えられたトランザクションに関連するログ
- **計装(インストゥルメンテーション)**
    - テレメトリーデータを出力するコード
    - サービスやシステムにオブザーバビリティコードを追加するプロセス
- **OTLP**
    - OpenTelemetryプロトコル

## OpenTelemetryのアーキテクチャ

以下は、一般的なOpenTelemetryを採用しているシステムのアーキテクチャです。

![](https://storage.googleapis.com/zenn-user-upload/da56a3738f67-20260118.png)

OpenTelemetryが担うのは、ApplicationとOpenTelemetry Collectorの部分です。

バックエンドシステム（e.g. Cloud Trace）に送信したあと、どのようにテレメトリーデータを可視化するかは、各ベンダーの裁量に委ねられています。

計装でアプリケーションから直接バックエンドに送信することも可能ではあるが、以下の理由からCollectorを挟むのが推奨されます。


理由は、サービスが素早くコレクターにデータをオフロードして、コレクターが再試行、バッチ処理、暗号化、機密データのフィルタリングなどの追加処理を行えるからです。

## 計装の方法

計装には大きく分けて2つのアプローチがあります。

- **ホワイトボックスアプローチ**
    - サービスやライブラリに直接テレメトリーコードを追加する
    - アプリケーション固有の詳細なデータが取得できる
- **ブラックボックスアプローチ**
    - 直接コードを変更することなくテレメトリーを生成するために外部のエージェントやライブラリを利用する
    - 既存のコードを変更せずに済む

どちらのアプローチも一長一短があり、要件に応じて使い分けることになります。

## まとめ

OpenTelemetryは、分散システム時代のオブザーバビリティを実現するための標準ツールです。

ログ、メトリクス、トレースという3本柱を統合し、相関データとして扱えるようにすることで、複雑なシステムのボトルネック特定を容易にします。

また、ベンダーニュートラルな設計により、特定のクラウドプロバイダーやSaaSにロックインされることなく、自由にバックエンドを選択できるのも大きな利点です。

OpenTelemetryの基本的な流れは以下の通りです。

1. アプリケーションに計装を追加してテレメトリーデータを生成
2. OpenTelemetry Collectorでデータを処理・変換
3. 各種バックエンド（Cloud Trace、Datadogなど）に送信

この仕組みを理解することで、現代的なシステムの監視・トラブルシューティングがスムーズに行えるようになります。

## 参考資料

- [OpenTelemetry Protocol が Google Cloud Observability に登場](https://cloud.google.com/blog/ja/products/management-tools/opentelemetry-now-in-google-cloud-observability/)
- [書籍『入門 OpenTelemetry』 / Intro of OpenTelemetry book](https://speakerdeck.com/ymotongpoo/intro-of-opentelemetry-book)
  - 一番いいかも
- [公式ドキュメント](https://opentelemetry.io/ja/docs/)
- [公式ドキュメント: JavaScript](https://opentelemetry.io/ja/docs/languages/js/)
- [TypeScriptでOpentelemetryを計装したらトレースががっつり欠損してた話](https://zenn.dev/ishii1648/articles/c8d12186ee8b40)
- [OpenTelemetry Protocol が Google Cloud Observability に登場](https://cloud.google.com/blog/ja/products/management-tools/opentelemetry-now-in-google-cloud-observability/)
- [Trace エクスポータから OTLP エンドポイントに移行する](https://cloud.google.com/trace/docs/migrate-to-otlp-endpoints?hl=ja)
- [OpenTelemetry超入門](https://qiita.com/tamura__246/items/7fc035eb59d04c9dd870)
  - Telemetry Data Flowの図が分かりやすい
- [Telemetry API がリリースされたので OTLP で Google Cloud にスパンを送ってみた](https://zenn.dev/cloud_ace/articles/9da89c5286ad60)
