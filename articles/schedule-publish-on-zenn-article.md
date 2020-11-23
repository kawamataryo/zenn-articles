---
title: "GitHub ActionsでZenn記事の予約投稿を実現する"
emoji: "📆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "GitHubActions", "zenn", "ci"]
published: true
---

「Zenn で記事の予約投稿が出来ると便利だなー」と思って調べてたら GitHub Actions を使って実現できたので紹介します。

# 仕組み

予約投稿の仕組みは記事追加のプルリクエストの**日時指定のマージ**です。
その実現のために`Merge Schedule`という GitHub Action を利用します。

https://github.com/marketplace/actions/merge-schedule

動きとしては以下の通りです。

1. ブランチを切り、公開したい記事を追加。プルリクエストを作成
2. GitHub Actions が起動し、Merge Schedule が登録
3. 日時指定でプルリクエストがマージされて master が更新
4. Zenn で記事が公開

![](https://storage.googleapis.com/zenn-user-upload/8uuxlktwapge6mnn28cvzyu95rec)

# 詳細

## GitHub Actions の登録

Zenn の記事管理リポジトリに移動して GitHub Actions のディレクトリを作ります。

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
      - uses: gr2m/merge-schedule-action@1ef6893192811edbec08113370a9d973922e84c7
        with:
          time_zone: "Asia/Tokyo"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

そして変更をコミット & プッシュします。
これでスケジュール投稿の準備が整いました。

:::message
上記コードでは`gr2m/merge-schedule-actio`のバージョンをコミット指定にしています。このコミットは私が `gr2m/merge-schedule-actio` に送った Time Zone 対応の PR がマージされたときのコミットです。2020/11/23 時点で、このコミットが最新のリリースに取り込まれていないので、このような指定としています。
（この変更がないと UTC 標準時での時刻指定が必要になります）。
https://github.com/gr2m/merge-schedule-action/pull/29
:::

## プルリクエストの作成

ローカルでプルリクエスト作成用のブランチを切ります。

```bash
$ git checkout -b introduce-xvim2
```

このブランチ上で記事を作成します。記事ファイルの公開指定は`published: true`としてください。
作成が完了したらコミットしてプルリクエストを作成します。

```
$ git add introduce-xvim2.md
$ git commmit -m "feat: introduce-xvim2"
$ git push --set-upstream origin introduce-xvim2
```

そして、プルリクエスト 作成の際の Descriptions に以下のように ISO 形式で公開したい日時を入力してください。これがマージ実行日時（記事公開日時）となります。

```
/schedule 2020-11-21T20:00
```

この状態でプルリクエストをオープンし、起動されている GitHub Actions をみてみます。

![](https://storage.googleapis.com/zenn-user-upload/sn2p1i10c090ykys61xe1me7band)

詳細をみると先ほど指定した公開日時でスケジュールされているのがわかります。

![](https://storage.googleapis.com/zenn-user-upload/phk5d1q2y6lvi43bpiki6yjjoukx)

あとは、その指定時間がくるとプルリクエストがマージされ記事が投稿されます。
便利！

:::message
Merge Schedule の CI が投稿完了まで終わらないので「GitHub Actions の無料枠を余裕で超えてしまう？」と思ったのですが、それは大丈夫でした。GitHub Actions の時間計測のものとは別枠のようです。
:::

## その他注意

[merge-schedule](https://github.com/marketplace/actions/merge-schedule) の GitHub Actions がどのように動いているのかを確認したら、以下のようになっていました。

https://github.com/gr2m/merge-schedule-action/blob/master/lib/handle_schedule.js

注目するのは以下処理です。

```js
// ...
const duePullRequests = pullRequests.filter(
  (pullRequest) => new Date(pullRequest.scheduledDate) < new Date()
);
```

どうやら、cron での GitHub Actions 実行時点で、scheduleDate を経過しているプルリクエストを抽出してマージ実行対象とするようです。

なので、**厳密に Descriptions に記載した日時にマージされるのではなく、cron での CI 実行時間により左右**されます。

もし分単位の公開指定をしたい場合は、GitHub Actions の cron の指定を変える必要があります。
サンプルの例だと`cron: 0 * * * *`で、1 時間に 1 回しか実行されません。
例え`/schedule 2020-11-21T20:45`と、UTC20 時 45 分に実行を指定しても、それは`2020-11-21T21:00`前後に実行されます。

もし、ほぼ 45 分に実行して欲しい場合は cron の指定を`45 * * * *`とするか、`0/15 * * * *`, `* * * * *`実行などに変更する必要があります。

ただ、このような指定とするとその分 GitHub Actions が実行されるので、 予約投稿日時によっては **GitHub Actions の実行時間の無料枠を超過する可能性がある** ことに注意してください。

# おわりに

以上「GitHub Action で Zenn の予約投稿を実現する」でした。
こういう応用が効くのも Zenn の良さですね。

自分のライフサイクル的に朝に記事を書きあげることが多いので、予約投稿はとても欲しかった機能でした。来月からスタートするアドベントカレンダーでも使っていきたいです。
