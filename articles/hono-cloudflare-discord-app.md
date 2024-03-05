---
title: "Hono + Cloudflareでもくもく会用のDiscord Botを作ってみた"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono", "cloudflare", "discord"]
published: false
---

# 🔥 作ったもの

地元の茨城県水戸市で毎月[もくもく会](https://mito-web-engineer.connpass.com/) を開催してもうすぐ5年になります。今回そのコミュニティのチャットツールをSlackからDiscordに移行するタイミングで、もくもく会の運営を補助するDiscord Botを作りました。

[Slackで使っていたBot](https://qiita.com/ryo2132/items/1a7d5e2a1b80414700e3)はBolt + Firebaseで作っていたのですが、今回は最近話題のHonoとCloudflare Workersを使ってみました。

主要機能はこちらです。

### チェックイン

`/checkin` コマンドでもくもく会のチェックインを行えます。モーダルが表示され、自己紹介と今日やることを入力して送信すると、フォーマットされた投稿がチャンネルに投稿されます。

もくもく会では開始前に、参加者にチェックインをしてもらい、この投稿内容をもとにはじめの共有を行なっています。これがあることで主催側として、参加者の顔と名前を覚えやすくて助かっています。

![Image from Gyazo](https://i.gyazo.com/9a7ee44ccaaa1c297cdad0bf792f56c4.gif)

さらに、Cloudflare の D1 に投稿内容を保存しているので、毎回内容が変わらない自己紹介は2回目以降自動で入力されるようになっています。

### もくもく会の開始・終了の通知

`/mokumoku-start`コマンドで、もくもく会のスケジュールやルールを投稿します。さらに、このコマンドが実行された日は、15:00にテックトーク（中休みに行う技術雑談）の通知、17:50に終了の通知も送られるようになっています。

もくもく作業に集中してしまい予定時刻を過ぎることが多々あるのでこれも便利です。

:::message
テックトークは[LAPRASのTech News Talk](https://www.youtube.com/playlist?list=PLKbaztxP2P4jpdF0P5YbJNJwFabB-pksK)を参考に、最近の気になったテック系ニュースや、お気に入りのツールを紹介するものです。特に資料も準備せず、各々好きな方法で発表してもらっています。その後の会話のきっかけにもなるので良いなと思っています。
:::

### 次回イベントの概要の作成

`/generate-event-description`コマンドで、次回のもくもく会の概要を作成してくれます。この概要のなかで、前回のイベントの紹介の文章があるのですが、そこに前回イベントのURL、皆の入力してくれた「今日やること」の文言を埋め込んでいます。

# 🛠️ 実装紹介

実装を簡単に紹介します。
Botのコードは以下リポジトリで公開しています。

https://github.com/Ibaraki-dev/mokumoku-bot

## Interactionsのハンドリング

Hono + CloudflareでDiscord Botを作る場合は、BotのWebhook エンドポイントにCloudflareにデプロイしたHonoのエンドポイントを設定し、そこにPOSTで送られる[application-command-object](https://discord.com/developers/docs/interactions/application-commands#application-command-object)をハンドリングすることになると思います。

Botでは、`/interaction`パスの中で、Request BodyのTypeによって処理を分岐させる形式にしています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/app.ts#L21-L72

そして、ループで回してTypeごとに登録された関数を実行するようにしています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/handleApplicationCommands.ts#L11-L44

最後に実際に処理を行う関数です。
Drizzle ORMでD1のデータを操作したりしながら、適切なレスポンスを整形します。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/applicationCommands/checkin.ts#L6-L29

## Schedule Triggerを利用した通知の実装

`/mokumoku-start`コマンドが実行された日は、15:00と17:00に指定のチャネルに通知を送るようにしています。この機能は、Cloudflare WorkersのSchedule Triggerを利用しています。

実装はシンプルで、毎日15:00と17:00にSchedule Triggerが実行され、当日のEventモデル（`/mokumoku-start`コマンドで作成される）がある場合のみ、通知を送るというものです。

cronの指定はUTCが基準なので、9時間ずれて指定する必要があるので注意が必要です。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/scheduled.ts#L10-L43

# 🏁 おわりに
