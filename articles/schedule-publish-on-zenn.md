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

次に workflows に以下`merge-schedule.yml`を作成します。

```yml
name: Merge Schedule
on:
  pull_request:
    types:
      - opened
      - edited
      - synchronize
  schedule:
    - cron: 0 * * * *

jobs:
  merge_schedule:
    runs-on: ubuntu-latest
    steps:
      - uses: gr2m/merge-schedule-action@v1.x
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

これでpushします。