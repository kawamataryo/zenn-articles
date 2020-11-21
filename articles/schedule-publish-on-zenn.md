---
title: "GitHub ActionでZennの予約投稿を実現する"
emoji: "📆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "GitHubAction", "zenn", "ci"]
published: false
---

Zenn の予約投稿が出来ると便利だなーと思って調べてたら GitHub Actions を使って実現できたので紹介します。

# リポジトリにMerge Schedule の追加

Zenn の記事管理リポジトリに移動して GitHub Workflow のディレクトリを作ります。

```
$ mkdir -p .github/workflows
```

次に workflows に以下`schedule-publish.yml`を作成します。

```yml
name: Schedule Publish
on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
  schedule:
    - cron: */15 * * * *

jobs:
  merge_schedule:
    runs-on: ubuntu-latest
    steps:
      - uses: gr2m/merge-schedule-action@v1.2.4
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

これで変更を commit して push します。

# プルリクエストの作成

あとは、master(main)ブランチで投稿したい記事を作成して、`published: false`の状態でcommit & push。

次に公開用ブランチを切ります。

```bash
$ git branch -b publish
```

そして公開したい記事を`published: true`に変更してCommitします。

```
$ git add hogehoge.md
$ git commit -m "hogehogeを公開"
```

このリポジトリをリモートにpushします。

```
$ git push --set-upstream origin publish
```
