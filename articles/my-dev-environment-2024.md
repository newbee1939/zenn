---
title: "俺の開発環境2024🚀"
emoji: "🚀"
type: "tech"
topics:
  - "vscode"
  - "開発環境"
  - "ツール"
  - "アイデア"
published: true
published_at: "2024-12-10 07:04"
---

今年の私の開発環境を共有します。

少しでも参考になるものがあれば嬉しいです。


## ダウンロードするツール

- [Visual Studio Code](https://code.visualstudio.com/Download)
- [Homebrew](https://brew.sh/)
- [Arc](https://arc.net/)
  - ブラウザ
  - 最近Chromeの代わりに使い始めた
- [Raycast](https://www.raycast.com/)
    - ランチャーツール
    - 便利すぎる..
- [WezTerm](https://wezfurlong.org/wezterm/)
  - ターミナル
- [Sequel Ace](https://sequel-ace.com/)
- [Docker for Mac](https://docs.docker.com/desktop/install/mac-install/)
  - もしくは[Rancher Desktop](https://docs.rancherdesktop.io/getting-started/installation/)などの代替ツール
- [f.lux](https://justgetflux.com/)
  - 現在の場所を入力したら、いい感じに画面の光量を調節してくれる
    - e.g. 夜間はブルーライトをカットする など

## Homebrewでインストールするツール

- jq
- starship
    - ターミナルをいい感じにするツール
- [k9s](https://github.com/derailed/k9s)
  - Kubernetesを楽に管理するためのツール
  - kubectlコマンドを打つ必要が無くなる
- gh
- git
  - デフォルトでインストールされている場合は不要

## VSCodeの設定

### `code .`でVSCodeを開けるようにする

1. `Cmd + Shift + P`でコマンドパレットを開く
2. `shell` と入力し、`Shell Command: install 'code' command in PATH`を選択する

### 拡張(extensions)をインストールする。

ディレクトリ内に以下の`extensions.list`を配置する。

```list
marp-team.marp-vscode
vscodevim.vim
github.copilot
github.copilot-chat
eamodio.gitlens
streetsidesoftware.code-spell-checker
exiasr.hadolint
gruntfuggly.todo-tree
yoavbls.pretty-ts-errors
github.vscode-pull-request-github
formulahendry.auto-rename-tag
shardulm94.trailing-spaces
pkief.material-icon-theme
github.github-vscode-theme
esbenp.prettier-vscode
firsttris.vscode-jest-runner
ChakrounAnas.turbo-console-log
Infracost.infracost
miguelsolorio.fluent-icons
```

### settings.jsonの設定

1. `Cmd + Shift + P`でコマンドパレットを開く
2. `user settings`と入力し、`Preferences: Open User Settings（JSON）`を選択
3. 以下の内容を記述する

```json
{
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.formatOnSave": true,
  "cSpell.userWords": [
    "astro",
    "biomejs",
    "browsi",
    "browsitag",
    "clearfix",
    "cloudrun",
    "cloudsql",
    "codegen",
    "datetime",
    "deno",
    "dprint",
    "envsubst",
    "esbenp",
    "fastify",
    "fluxtag",
    "fuga",
    "gcloud",
    "googletag",
    "hoge",
    "jotai",
    "kintone",
    "kustomize",
    "laravel",
    "luxon",
    "marp",
    "memorystore",
    "nestjs",
    "qiita",
    "testid",
    "traefik",
    "urql",
    "zenn"
  ],
  "window.zoomLevel": 2,
  // GitHub Copilot
  "github.copilot.chat.localeOverride": "ja",
  "github.copilot.chat.edits.enabled": true,
  // VSCodeの不要な要素を非表示
  "workbench.editor.showTabs": "none",
  "editor.minimap.enabled": false,
  "workbench.statusBar.visible": false,
  "window.customTitleBarVisibility": "windowed",
  "editor.glyphMargin": false,
  "editor.folding": false,
  "breadcrumbs.symbolPath": "off",
  "workbench.activityBar.location": "top",
  // VSCodeのテーマなど
  "workbench.iconTheme": "material-icon-theme",
  "workbench.colorTheme": "GitHub Dark Default",
  // プロダクトアイコン
  "workbench.productIconTheme": "fluent-icons",
  // Turbo Console Log
  "turboConsoleLog.addSemicolonInTheEnd": true,
  "turboConsoleLog.insertEnclosingClass": false,
  "turboConsoleLog.insertEnclosingFunction": false,
  "turboConsoleLog.quote": "'",
  "turboConsoleLog.wrapLogMessage": true,
  // エクスプローラーのインデント
  "workbench.tree.indent": 24,
  // タブ数制限
  "workbench.editor.limit.enabled":true,
  "workbench.editor.limit.value": 3,
  // ブラケットペアの範囲を可視化
  "editor.guides.bracketPairs": "active",
  // Monaspace
  "editor.fontFamily": "Monaspace Neon",
  // スペースの可視化
  "editor.renderWhitespace": "boundary",
  // save時に不要なスペースを削除
  "files.trimTrailingWhitespace": true,
  // 不要な効果音をOFFに
  "editor.accessibilitySupport": "off",
}
```

個人的にVSCodeの画面を広く使うのが好きなので、なるべく不要な要素は消すようにしています。

## Shell・ターミナルの設定

Macのデフォルトシェルである`zsh`を使用。

### gitのユーザー&エイリアス設定

以下のコマンドでユーザー名とメールアドレスを設定。

```shell
git config --global user.name "Hoge Taro"
git config --global user.email sample@example.com
```

`git config --global --edit`を実行して、以下のエイリアスを設定。

```
[alias]
st = status
co = checkout
br = branch
```

参考: [Gitでalias（エイリアス）を設定する方法をサクッと解説](https://www.engilaboo.com/how-to-set-git-alias/#google_vignette)

### GitとGitHubの連携設定

[gh auth login](https://cli.github.com/manual/gh_auth_login)を実行する（sshキーの設定）

参考: [俺たちはもう GitHub のために ssh-keygen しなくていい](https://zenn.dev/lovegraph/articles/529fe37caa3f19)

### Dockerコマンドのエイリアスを設定

以下のコマンドで.zshrcを開く。

```shell
vim ~/.zshrc
```

以下の設定を追加

```
alias dc='docker compose'
```

参考: [Dockerコマンドにalias(エイリアス)を設定する方法【作業効率UP】](https://www.engilaboo.com/docker-alias/)

### Starshipの設定

.zshrcを開く。

```shell
vim ~/.zshrc
```

以下の記述を追加。

```
eval "$(starship init zsh)"
```

より細かく設定。

```shell
mkdir -p ~/.config && vim ~/.config/starship.toml
```

starship.tomlに以下の設定を追加。

```toml
[character]
success_symbol = "[❯](bold green)"
error_symbol = "[✗](bold red)"

[git_branch]
symbol = "🌱 "
style = "bold #b8d200"

[[battery.display]]
threshold = 20
style = "bold red"
[battery]
discharging_symbol = "😞 "

[git_status]
modified = "📝"
staged = '[++\($count\)](green)'
```

参考: [Starship インストール](https://starship.rs/ja-JP/guide/)

## Chrome拡張のインストール

- [OneTab](https://chromewebstore.google.com/detail/onetab/chphlpgkkbolifaimnlloiipkdnihall?hl=ja)
- [Refined GitHub](https://chromewebstore.google.com/detail/refined-github/hlepfoohegkhhmjieoechaddaejaokhf)
  - GitHubの画面をいい感じにアップデート
  - 参考: https://github.com/refined-github/refined-github/blob/main/readme.md
- [Dark Reader](https://chromewebstore.google.com/detail/dark-reader/eimadpbcbfnmbkopoojfekhnkhdbieeh?hl=ja)
- [Send to Google Tasks](https://chromewebstore.google.com/detail/send-to-google-tasks/acomfpnllcpggnclcogaiceicgljnbac)
  - Google CalenderのTODOに気になった記事を登録できる
- [AutoPagerize](https://chromewebstore.google.com/detail/autopagerize/igiofjhpmpihnifddepnpngfjhkfenbp)
  - Webページのページャーを無くせる
- [AdBlock](https://chromewebstore.google.com/detail/adblock-%E2%80%94-%E6%9C%80%E9%AB%98%E5%B3%B0%E3%81%AE%E5%BA%83%E5%91%8A%E3%83%96%E3%83%AD%E3%83%83%E3%82%AB%E3%83%BC/gighmmpiobklfepjocnamgkkbiglidom?hl=ja)
  - 広告消せる
- [Redirect Path](https://chromewebstore.google.com/detail/redirect-path/aomidfkchockcldhbkggjokdkkebmdll?hl=ja)
- [daily.dev](https://chromewebstore.google.com/detail/dailydev-the-homepage-dev/jlmpjdjjbgclbocgajdjefcidcncaied)
- [DeepL](https://chromewebstore.google.com/detail/deepl%EF%BC%9Aai%E7%BF%BB%E8%A8%B3%E3%81%A8%E6%96%87%E7%AB%A0%E4%BD%9C%E6%88%90%E3%83%84%E3%83%BC%E3%83%AB/cofdbpoegempjloogbagkncekinflcnj?hl=ja)
- [EditThisCookie](https://chromewebstore.google.com/detail/fngmhnnpilhplaeedifhccceomclgfbg?hl=ja)
- [Vimium](https://chromewebstore.google.com/detail/vimium/dbepggeogbaibhgnhhndojpepiihcmeb?hl=ja-jp)
  - 参考: [ブラウザを Vim ライクに操作する Vimium の布教と知見まとめ](https://zenn.dev/mkobayashime/articles/vimium-vim-browser)
- [ModHeader](https://chromewebstore.google.com/detail/modheader-modify-http-hea/idgpnmonknjnojddfkpgkljpfnnfcklj)
- [FireShot](https://chromewebstore.google.com/detail/%E3%82%A6%E3%82%A7%E3%83%96%E3%83%9A%E3%83%BC%E3%82%B8%E5%85%A8%E4%BD%93%E3%82%92%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88-firesh/mcbpblocgmgfnpjjppndjkmgjaogfceg?hl=ja&pli=1)
- theme(以下のいずれか)
  - [Google Arts & Culture](https://chromewebstore.google.com/detail/google-arts-culture/akimgimeeoiognljlfchpbkpfbmeapkh)
  - 世界の名画が表示される

## Macの設定

- よく使う言葉の辞書登録
- キーボード操作時の音を消す
  - `System Preferences > Sound > Sound Effects`のVolumeを0に
- メニューバーを右にする
  - `System Preferences > Dock > Position on screen > Right`

## おわりに

今年使い始めたツールで一番よかったのは、なんといっても **Raycast** です。

かなり開発生産性を向上させることができたと感じています。

来年も、気になったツールを積極的に試しつつ開発環境をブラッシュアップさせていきたいと考えています。
