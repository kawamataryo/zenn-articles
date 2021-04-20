---
title: "特定のブランチ名のプルリクエストで動作するGitHub Actionsを作る"
emoji: "🚟"
type: "tech"
topics: ["githubactions", "github"]
published: true
---

特定のブランチ名のプルリクエストの場合にの実行される GitHub Actions のジョブを作ろうとしたときに詰まったのでメモ。

# よくある間違い
最初によくある間違いというか、自分が最初書いていて「なぜか実行されん🤔」と詰まった書き方です。
以下は dependabot の作ったプルリクエストの場合のみ実行される GitHub Actions のジョブを作ろうとした場合の例です。

```yml
name: Sample actions

on:
  pull_request:
    branches:
      - 'dependabot/**'

jobs:
  sample-job:
    runs-on: ubuntu-18.04
    steps:
      - name: Greeting
        run: echo 'Hello world'
```

`on.pull_request.branches` でブランチ名を指定しています。ですが、これでは動作しません。
というのも、ここの`on.pull_request.branches`で評価されるブランチ名はプルリクエストのマージ対象ブランチ名（ex: main, master, develop）だからです。

# Job単位で制御
Job 単位でブランチ名による制御を行う場合の書き方はこちらです。

```yml
name: Sample actions

on:
  pull_request:
    branches:
      - 'dependabot/**'

jobs:
  sample-job:
    runs-on: ubuntu-18.04
    if: contains(github.head_ref, 'dependabot')
    steps:
      - name: Greeting
        run: echo 'Hello world'
```

`jobs.*.if`で条件式を追加することで、特定のブランチ名の場合のみ実行する Job が作れます。
ここでは、`contains`式で`github.head_ref`から取得出来るブランチ名を判定することで制御しています。

# Step単位で制御
Step 単位でブランチ名による制御を行う場合の書き方はこちらです。

```yml
name: Sample actions

on:
  pull_request:
    branches:
      - 'dependabot/**'

jobs:
  sample-job:
    runs-on: ubuntu-18.04
    steps:
      - name: Greeting
      - if: contains(github.head_ref, 'dependabot')
        run: echo 'Hello world' # ここだけ上の条件式が真の場合のみ実行される
      - name: Greeting
        run: echo 'Good by world'
```

`jobs.*.steps.if`で条件式を追加することで、特定のブランチ名の場合のみ実行する Step が作れます。