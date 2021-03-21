---
title: "Slackの分報を社内Twitterに！皆の分報を一つのチャネルに集約するSlackボットを作ってみた"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["slack", "serverlessframework", "lambda", "typescript"]
published: false
---

Slack の分報（`times-xxx`チャネル）は、自分の作業ログや思考の整理に使えたり、チームメンバーの状況を知るためにも使えたりとても良いですよね。

ただ、チームメンバーの数だけチャネルが増えるので、人数が増えるとそれぞれの分報チャネルを見てみ回るのが地味に大変です。

それを解決するためにそれぞれの**分報チャネルの投稿を全て 1 つのチャネルに共有する Slack アプリ**を作ってみました。このチャネルがあると、そのチャネルを見るだけで 社内 Twitter 的に皆の分報を閲覧することが出来るので便利です。
この記事ではそのアプリの紹介と設定方法を書きます。

※ 元ネタは[前職 Misoca](https://www.wantedly.com/stories/s/team_misoca)の`current-all`チャネルです。感謝 🙏
※ 分報の概要は[こちら](https://giginc.co.jp/blog/giglab/slack-hunho)

# 作ったもの

各分報の投稿をひとつのチャネルに集約する Slack Bot です。
この Bot を各分報チャネルと、分報の集約チャネルに追加すると、各分報の投稿を全て集約チャネルにシェアしてくれます。

![](https://i.gyazo.com/a8f99f12f2badbf9e1d334dcba0c3a77.gif)

リポジトリはこちらです。

https://github.com/kawamataryo/times-all-bot

以降でセットアップ方法を解説します。

# セットアップ

## Slackアプリの作成
https://api.slack.com/apps にアクセスし、`Create New App`で新しいアプリを作成します。

![](https://i.gyazo.com/ec073dbdc32aebfd81d04ef2fdc2626c.png)

作成後、サイドバーから `OAuth & Permissions` にアクセスしてアプリの権限のスコープを設定します。設定するスコープは`chat:write`と`channels:history`です。

![](https://i.gyazo.com/d318576f6bff9d4963793570d2d0695c.png)

その後、ページ上部の Install to Workspace でワークスペースにアプリを追加します。
アプリ追加後表示される OAuth Token は後で使うのでメモしましょう。

![](https://i.gyazo.com/1f114e3934718cfbba2930d4fc877318.png)

あと、サイドバーにある`Basic Information`にアクセスして`Signing Secret`もメモしてください。

![](https://i.gyazo.com/b460141ebf59d9212e80ab6f390e7259.png)

以上でいったん Slack アプリ側の設定は完了です。

## Lambdaへのデプロイ
まず任意のディレクトリにプロジェクトをクローンします。

```
$ git clone https://github.com/kawamataryo/times-all-bot.git
$ cd times-all-bot
$ npm i
```

デプロイ時に Slack の情報を参照するために環境変数使っています。プロジェクト直下に以下内容で`.env`を作成してください。`SLACK_SIGNING_SECRET`、`SLACK_BOT_TOKEN`は前述の Slack アプリの作成でメモした`Signing Secret`と`OAuth Token`と値です。`AGGREGATE_CHANNEL_ID`は分報の投稿を集約するチャネル（`times-all`等）の ID です。

※ チャネル ID の確認方法は[こちら](https://qiita.com/YumaInaura/items/0c4f4adb33eb21032c08)から。

```.env
SLACK_SIGNING_SECRET=<YOUR_TOKEN>
SLACK_BOT_TOKEN=<YOUR_TOKEN>
AGGREGATE_CHANNEL_ID=<CHANNEL_ID>
```

これで準備は完成です。以下コマンドで AWS Lambda にデプロイされます。

```
$ npm run deploy
```

:::message
このアプリは Serverless フレームワークを使って AWS Lambda にアプリをデプロイします。事前に AWS のクレデンシャルを`~/.aws/credentials`に設定しておいてください（[詳細](https://www.serverless.com/framework/docs/providers/aws/guide/credentials/)）。
:::


無事デプロイが完了するとエンドポイントの URL が標準出力に出ます。
それをメモしてください。

```bash
# ...
endpoints:
  POST - https://u6d9kerzyb.execute-api.us-east-1.amazonaws.com/dev/slack/events
# ...
```

## Slackアプリでのエンドポイント登録

最後にまた Slack アプリ側の設定です。
Slack アプリの管理画面からサイドバーの`Event Subscriptions`にアクセスし、`Enable Events`のチェックを ON にしてイベント購読を有効化します。

`Request URL`は先程デプロイ時に取得した Lambda のエンドポイントを設定してください。
そして`Subscribe to bot events`に`message.channels`を追加します。

![](https://i.gyazo.com/6a5d160f2eeb5d850e6b76d0cef79b34.png)

設定後、アプリを再インストールする必要があるので、`OAuth & Permissions`から必ず行いましょう。これでアプリを追加したチャネルでイベントが発生するたびに Lambda にリクエストが飛びます。

## 分報チャネル・集約チャネルへのアプリの追加

各分報チャネルと集約チャネルに先程作成したアプリをインストールすれば完了です 🎉
**※ 必ず両方に追加する必要があるので注意**

以降は各分報チャネルに投稿した内容の全てが集約チャネルにシェアされます。

![](https://i.gyazo.com/a8f99f12f2badbf9e1d334dcba0c3a77.gif)

# 実装

実装はとてもシンプルです。
Lambda で [Bolt for JavaScript](https://slack.dev/bolt-js/concepts) を起動して分報のチャネルのイベントを購読し、集約チャネルに投稿しています。

ポイントは、パーマリンクの形式で投稿している点です。
`chat.postMessage`で直接分報の投稿内容を集約チャネルに投稿することもできるのですが、それを行うと`@here`などのメンションも展開されてしまいます。また、逆に添付ファイルなどは展開されないという問題もあります。

なので、集約チャネルにはパーマリンクを投稿し、チャネルでのスニペット展開で分報の投稿内容を確認できる形式にしています。

```ts:src/functions/slack/handler.ts
import { App, ExpressReceiver } from "@slack/bolt";
import serverlessExpress from "@vendia/serverless-express";

const aggregateChannelId = process.env.AGGREGATE_CHANNEL_ID;

const expressReceiver = new ExpressReceiver({
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  processBeforeResponse: true,
});

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  receiver: expressReceiver,
});

app.message(async ({ client, event }) => {
  // 集約チャネル内の投稿か、投稿の編集・削除の場合は除外
  if (
    event.channel == aggregateChannelId ||
    event.subtype == "message_changed" ||
    event.subtype == "message_deleted"
  ) {
    return;
  }

  try {
    // 投稿のpermalinkの取得
    const permalinkRes = await client.chat.getPermalink({
      channel: event.channel,
      message_ts: event.event_ts,
    });

    // 集約チャネルへの投稿
    await client.chat.postMessage({
      channel: aggregateChannelId,
      text: permalinkRes.permalink as string,
    });
  } catch (e) {
    console.error(e);
  }
});

export const handler = serverlessExpress({
  app: expressReceiver.app,
});
```


詳細は、リポジトリをご覧ください。

https://github.com/kawamataryo/times-all-bot

# 終わりに

以上「Slack の分報を社内 Twitter に！　皆の分報を 1 つのチャネルに集約する Slack ボットを作ってみた」でした。
現職の Slack にも導入して、幸いにも好評です。この Bot 自体はただ投稿を別のチャネルにシェアするだけなので、分報の集約以外にも色々使い道はあるかなと思います。是非使ってもらえると嬉しいです。

また何か使ってみて不具合があれば気軽にコメントを下さい。