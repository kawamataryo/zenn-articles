---
title: "GitHub ActionでZennの記事予約投稿を実現する"
emoji: "📆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "GitHubAction", "zenn", "ci"]
published: false
---

「Zenn で記事の予約投稿が出来ると便利だなー」と思って調べてたら GitHub Actions を使って実現できたので紹介します。

# 仕組み

予約投稿の仕組みは記事追加のプルリクエストの日時指定のマージです。
その実現のために`Merge Schedule`という GitHub Action を利用します。

https://github.com/marketplace/actions/merge-schedule

動きとしては以下の通りです。

1. ブランチを切り、公開したい記事を執筆。プルリクエストを作成
2. プルリクエストをフックに GitHub Actions が起動。Schedule Merge を登録
3. 日時指定でプルリクエストがマージされて master が更新
4. Zenn で記事が公開

![](https://storage.googleapis.com/zenn-user-upload/8uuxlktwapge6mnn28cvzyu95rec)


# 詳細

## GitHub Actionsの登録


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
      - cron: 0 * * * *

jobs:
  merge_schedule:
    runs-on: ubuntu-latest
    steps:
      - uses: gr2m/merge-schedule-action@v1.2.4
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

そして変更をコミット & プッシュします。
これでスケジュール投稿の準備が整いました。

## プルリクエストの作成

あとはローカルで master からプルリクエスト作成用のブランチを切ります。

```bash
$ git checkout -b introduce-xvim2
```
このブランチ上で記事を作成します。
作成が完了したらコミットしてプルリクエストを作成します。

```
$ git add introduce-xvim2.md
$ git commmit -m "feat: introduce-xvim2"
$ git push --set-upstream origin introduce-xvim2
```

この時に Descriptions に以下のようにISO 形式で**UTC 協定世界時**で投稿したい日付を入力してください。これがマージ実行日時となります。

:::message alert
最初、UTC 標準日時で設定するというのに気づかず、「全然マージされん🤔」と悩みました。
必ず UTC 標準日時に直して（-9 時間）設定してください。
:::


```
/schedule 2020-11-21T20:30
```

この状態でプルリクエストをオープンし、起動されている GitHub Actions をみてみます。

![](https://storage.googleapis.com/zenn-user-upload/sn2p1i10c090ykys61xe1me7band)

詳細をみると先ほど指定した公開日時でスケジュールされているのがわかります。

![](https://storage.googleapis.com/zenn-user-upload/phk5d1q2y6lvi43bpiki6yjjoukx)

あとは、その指定時間がくるとプルリクエストがマージされ記事が投稿されます。

:::message
Merge Schedule の CI が投稿完了まで終わらないので、GitHub Actions の無料枠を余裕で超えてしまう？　と思ったのですが、それは大丈夫でした。GitHub Actions の時間計測のものとは別枠で動いているようです。
:::

# おわりに

以上「GitHub Action で Zenn の予約投稿を実現する」でした。
こういう応用が効くのも Zenn の良さですね。

自分のライフサイクル的に朝記事を書きあげることが多いので、予約投稿はとても欲しかった機能でした。今後もこの機能を使って色々記事を書いていきたいです。