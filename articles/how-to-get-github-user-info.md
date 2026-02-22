---
title: "【GitHub】組織、チーム、およびチームメンバーの情報を取得する"
emoji: "🙂"
type: "tech"
topics:
  - "github"
  - "bash"
  - "tips"
  - "shell"
  - "gh"
published: true
published_at: "2024-11-14 07:26"
---

GitHub CLI (gh) を使用して、ユーザーが所属する組織、チーム、およびチームメンバーの情報を取得し、それをCSVファイルに保存するシェルスクリプトです。

以下のシェルスクリプトを実行することで、`teams_and_members.csv`に対象の情報が保存されます。

```sh
#!/bin/bash

# CSVファイルを作成
echo "Organization,Team,Member" > teams_and_members.csv

# Organizationsの一覧を取得して処理
gh api user/orgs --jq '.[].login' | while read org; do
  gh api orgs/$org/teams --jq '.[].slug' | while read team; do
    gh api orgs/$org/teams/$team/members --jq '.[].login' | while read member; do
      echo "$org,$team,$member" >> teams_and_members.csv
    done
  done
done
```

コードの説明は以下の通り。

**1. シェバン行**
```bash
#!/bin/bash
```
- スクリプトがBashシェルで実行されることを指定

**2. CSVファイルの作成**
```bash
echo "Organization,Team,Member" > teams_and_members.csv
```
- ヘッダー行を含むCSVファイルを作成

**3. 組織の一覧を取得して処理**
```bash
gh api user/orgs --jq '.[].login' | while read org; do
```
- gh api user/orgsコマンドでユーザーが所属する組織の一覧を取得し、`jq`で組織のログイン名を抽出
- 各組織についてループ処理を行う

**4. チームの一覧を取得して処理**
```bash
gh api orgs/$org/teams --jq '.[].slug' | while read team; do
```
- gh api orgs/$org/teamsコマンドで組織内のチームの一覧を取得し、`jq`でチームのスラッグを抽出
- 各チームについてループ処理を行う

**5. メンバーの一覧を取得して処理**
```bash
gh api orgs/$org/teams/$team/members --jq '.[].login' | while read member; do
```
- gh api orgs/$org/teams/$team/membersコマンドでチーム内のメンバーの一覧を取得し、`jq`でメンバーのログイン名を抽出
- 各メンバーについてループ処理を行う

**6. CSVファイルに書き込み**
```bash
echo "$org,$team,$member" >> teams_and_members.csv
```
- 組織、チーム、メンバーの情報をCSVファイルに入力する
