---
title: "Firebase Realtime Databaseでバックアップ画面が表示されない場合の対処法"
emoji: "😇"
type: "tech"
topics:
  - "firebase"
  - "エラー"
  - "googlecloud"
  - "realtimedatabase"
published: false
published_at: "2025-05-21 18:53"
---

Firebase Realtime Databaseでバックアップ画面が表示されない場合の対処法。

![](https://storage.googleapis.com/zenn-user-upload/e7cfdda2427b-20250521.png)

あくまで私の場合ですが、Defaultのデータベースを追加したら表示されるようになりました。

```hcl
resource "google_firebase_database_instance" "default" {
  provider    = google-beta
  project     = local.project_id
  region      = "asia-southeast1"
  // 任意の値
  instance_id = "prj-example-123-default-rtdb"
  type        = "DEFAULT_DATABASE"
}
```

参考: https://registry.terraform.io/providers/hashicorp/google/6.14.0/docs/resources/firebase_database_instance

こんな感じで。

![](https://storage.googleapis.com/zenn-user-upload/25988e048d32-20250521.png)

なぜデフォルトデータベースを追加したらバックアップ画面が表示されるようになるのかは不明です。

結構ハマる人いそう..
