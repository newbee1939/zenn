---
title: "Gemini CLIが登場"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gemini", "cli", "ai"]
published: true
---
# Gemini CLIが登場

Googleが2024年6月25日にリリースした「Gemini CLI」は、コマンドラインでGemini 2.5 Proの強力な機能を活用できるツールです。本記事では、Gemini CLIの基本的な使い方から応用例まで、わかりやすく解説します。

## Gemini CLIとは？

Gemini CLIは、ターミナル上でAIを活用したワークフローを実現するオープンソースのコマンドラインツールです。従来のChatGPTやGeminiのWebインターフェースとは異なり、開発者のワークフローに直接統合され、コードベースの理解、ファイル操作、コマンド実行、動的なトラブルシューティングを自然言語で行えます。

### 主な特徴

**1. 大規模なコンテキストウィンドウ**
- Gemini 2.5 Proの100万トークンのコンテキストウィンドウを活用
- 大規模なコードベースも一度に理解・編集可能

**2. 充実した無料枠**
- 個人のGoogleアカウントでログインするだけで利用開始
- 毎分60リクエスト、1日1,000リクエストまで無料
- 個人利用には十分な無料利用枠

**3. 拡張性とオープンソース**
- Apache 2.0ライセンスで完全にオープンソース
- Model Context Protocol (MCP)対応で機能拡張が容易
- コミュニティによる継続的な改善

**4. 統合された強力なツール**
- Google検索によるリアルタイム情報の取得
- 既存ワークフローとの統合
- マルチモーダル機能（画像、PDFからのアプリ生成）

## インストール方法

### 前提条件
- Node.js 18以上がインストールされていること

### 最も簡単な方法（推奨）

```bash
npx https://github.com/google-gemini/gemini-cli
```

### グローバルインストール

```bash
npm install -g @google/gemini-cli
gemini
```

### 初回セットアップ

1. **カラーテーマの選択**
   - 初回起動時にターミナルの見た目を設定できます

2. **認証**
   - 個人のGoogleアカウントでサインインします
   - 無料枠（毎分60リクエスト、1日1,000リクエスト）が自動適用されます

## 基本的な使い方

### 新しいプロジェクトの開始

```bash
cd new-project/
gemini
> Write me a Gemini Discord bot that answers questions using a FAQ.md file I will provide
```

### 既存プロジェクトでの作業

```bash
git clone https://github.com/google-gemini/gemini-cli
cd gemini-cli
gemini
> Give me a summary of all of the changes that went in yesterday
```

## 高度な機能

### API キーによる高度な利用

無料枠を超えるリクエストが必要な場合や、特定のモデルを使用したい場合は、APIキーを設定できます：

1. [Google AI Studio](https://aistudio.google.com/apikey)でAPIキーを生成します
2. 環境変数に設定します：

```bash
export GEMINI_API_KEY="YOUR_API_KEY"
```

### MCP サーバーとの連携

Model Context Protocol (MCP)を使用して、外部ツールやサービスと連携：

- メディア生成ツール（Imagen、Veo、Lyria）
- エンタープライズコラボレーションツール
- カスタムツールやスクリプト

### Google検索との統合

リアルタイムの情報取得が可能：

```text
> Search for the latest React best practices and apply them to our codebase
```

## トラブルシューティング

### よくある問題

1. **認証エラー**
   - 個人のGoogleアカウントでログインしているか確認
   - 企業アカウントの場合は、APIキーの使用を検討

2. **リクエスト制限**
   - 無料枠：毎分60リクエスト、1日1,000リクエスト
   - 制限に達した場合は、APIキーまたは有料プランを検討

3. **インストールの問題**
   - Node.js 18以上がインストールされているか確認
   - ネットワーク環境の確認

詳細なトラブルシューティングは、[公式ドキュメント](https://github.com/google-gemini/gemini-cli/blob/main/docs/troubleshooting.md)を参照してください。

## まとめ

Gemini CLIは、開発者の生産性を大幅に向上させる便利なツールです。ターミナルから直接Gemini 2.5 Proの強力な機能を活用でき、コード理解、生成、デバッグ、ワークフロー自動化を自然言語で実行できます。

**始めるメリット：**
- 無料で始められる（充実した無料枠）
- 既存のワークフローに簡単に統合
- オープンソースで安心
- 継続的なアップデートとコミュニティサポート

今すぐ以下のコマンドで始めてみましょう。

```bash
npx https://github.com/google-gemini/gemini-cli
```

---

**参考リンク：**
- [Gemini CLI公式ブログ](https://blog.google/technology/developers/introducing-gemini-cli-open-source-ai-agent/)
- [GitHubリポジトリ](https://github.com/google-gemini/gemini-cli)
- [公式ドキュメント](https://github.com/google-gemini/gemini-cli/blob/main/docs/index.md)
