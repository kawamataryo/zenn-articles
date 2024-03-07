---
title: "Hono + Cloudflareでもくもく会用のDiscord Botを作ってみた"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono", "cloudflare", "discord"]
published: false
---

# 🔥 作ったもの
茨城県水戸市で毎月[もくもく会](https://mito-web-engineer.connpass.com/) を開催してもうすぐ5年になります。今回そのコミュニティのチャットツールをSlackからDiscordに移行するタイミングで、もくもく会の運営を補助するDiscord BotをHonoとCloudflare Workersで作りました。

主要機能は以下です。

**📌 チェックイン**
`/checkin` コマンドでもくもく会のチェックインを行えます。モーダルが表示され、自己紹介と今日やることを入力して送信すると、フォーマットされた内容がチャンネルに投稿されます。

実際のもくもく会では開始前に、参加者にチェックインを実行してもらい、この投稿内容をもとにはじめの共有を行なっています。これがあることで、主催側としても参加者の顔と名前を覚えやすくて助かっています。

また、Cloudflare の D1 に投稿内容を保存しているので、ほぼ内容が変わらない自己紹介は2回目以降自動で入力されます。

![Image from Gyazo](https://i.gyazo.com/9a7ee44ccaaa1c297cdad0bf792f56c4.gif)


**📌 もくもく会の開始・終了の通知**
`/mokumoku-start`コマンドで、もくもく会のスケジュールやルールを投稿します。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-09-46.png =550x)

さらに、このコマンドが実行された日は、15:00にLT・テックトークの通知、17:50に終了の通知も送られるようになっています（Cron Triggerを利用）。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-09-58.png =550x)
![](/images/hono-cloudflare-discord-app/2024-03-06-09-10-09.png =550x)

もくもく作業に集中してしまい予定時刻を過ぎることが多々あるのでこれも便利です。

:::message
テックトークは[LAPRASのTech News Talk](https://www.youtube.com/playlist?list=PLKbaztxP2P4jpdF0P5YbJNJwFabB-pksK)を参考に、最近の気になったテック系ニュースや、お気に入りのツールを紹介するものです。特に資料も準備せず、各々好きな方法で発表してもらっています。その後の会話のきっかけにもなるので良いなと思っています。
:::

**📌 次回イベントの概要の作成**
`/generate-event-description`コマンドで、次回のもくもく会の概要を作成してくれます。この概要のなかで、前回のイベント紹介の文章があるので、そこにイベントのURLと前回皆の入力してくれた「今日やること」の文言を自動で埋め込んでいます。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-10-54.png =550x)


# 🛠️ 実装紹介

実装について簡単に紹介します。
[Discord.js](https://discord.js.org/)を使わないJSでのDiscord Bot実装例はあまりないと思うので、誰かの参考になれば嬉しいです。

今から紹介するコードはすべて以下リポジトリで公開しています。

https://github.com/Ibaraki-dev/mokumoku-bot

また、Hono + Cloudflare 環境での Discord Bot の開発全般については以下記事が大変参考になりました🙏

https://blog.lacolaco.net/posts/discord-bot-cfworkers-hono/

## Interactionsのハンドリング

今回のBotでは複数のスラッシュコマンドやモーダルの送信をハンドリングする必要があったので、`/interaction`ルートで、ApplicationCommandObjectのTypeをみて処理を分岐させています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/app.ts#L21-L72

そしてTypeごとに登録された関数をループで回してを実行するようにしています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/handleApplicationCommands.ts#L11-L44

実際に処理を行う関数ではDrizzle ORMでD1のデータを操作したりしながら、適切なレスポンスを整形します。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/applicationCommands/checkin.ts#L6-L29

## モーダルの表示
Botでモーダルを表示するには、まずスラッシュコマンドのレスポンスでモーダルのコンポーネントを返します。そして、モーダルの送信のリクエストをハンドリングします。

以下のコードは、`/checkin`コマンドでモーダルを表示するためのレスポンスです。`ここで指定しているcustom_idで、モーダルの送信結果をハンドリングするためのコードと紐づけられます。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/responses/checkinCommandResponse.ts#L9-L52

そして、モーダルからの送信結果を処理する部分はこちらです。いろいろやっていますが、基本的にはモーダルの送信結果を受け取って、それを元に投稿内容を作成しています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/modalSubmits/checkinModal.ts#L6-L54

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/responses/checkinModalSubmitResponse.ts#L20-L58


## Cron Triggerを利用した通知の実装

`/mokumoku-start`コマンドが実行された日は、15:00と17:50に指定のチャネルにメッセージを送るようにしています。この機能は、Cloudflare Workersの[Cron Trigger](https://developers.cloudflare.com/workers/configuration/cron-triggers/)を利用しています。

実装はシンプルで、毎週土日15:00と17:50にCron Triggerが実行され、当日のEventモデル（`/mokumoku-start`コマンドで作成される）がある場合のみ、通知を送るというものです。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/scheduled.ts#L10-L43


:::message
Cron Triggerのcronの指定はUTC基準なので、日本時間の場合は9時間ずらして指定する必要があります。
:::

# 🔥 Hono + Cloudflareの良かったところ
今回はじめてHonoやCloudflare、Drizzleを使って開発したので簡単に所感をまとめます。

**📌 開発スタート・デプロイが爆速**
`bunx create-hono` でHonoのプロジェクトを作り、`bunx deploy` でデプロイするだけでCloudflareにデプロイされます。追加設定はLinter・Formatterくらいで、他は何も設定することなく開発をスタートできました。開発スタートからデプロイまでのスピードがとても速いです。また、Cloudflareのスタック（D1など）を利用する際にも、HonoのBindingsで簡単に追加できるのも良いですね。

**📌 Hono、Drizzle ORMの型補完が強力**
HonoとDrizzle ORMは型補完が強力で、APIを調べずとも補完に任せてほぼ迷うことなくコードを書くことができました。かなり開発体験が良かったです。

**📌 Testが書きやすい**
Honoには便利なTestHelperが提供されているのでAPIのテストがとても楽です。今回のBotでもほぼすべての処理にテストを書けました。

**📌 @yusukebeさんの存在**
Honoの開発者でCloudflareに在職してる[@yusukebe](https://twitter.com/yusukebe)さんの存在も大きいです。@yusukebeさんのXやブログからCloudflareやHono最新の情報を日本語でキャッチアップできます。今回のBot開発時にもZennのスクラップでメモしていたら、[Yusukebeさんからアドバイス](https://zenn.dev/link/comments/8a912a10634481)をもらえました。感謝🙏

# 🏁 おわりに

Discord Botを作るのもHonoやCloudflareを使うのも今回はじめての試みで、いろいろ詰まるところはあったのですが、HonoとCloudflareの開発体験がとても良く楽しく作れました。

今後、他のアプリもこの技術スタックで作ってみたいなと思っています。

最後に宣伝を・・
Ibaraki.devでは毎月もくもく会を開催しています。ゆるい勉強会で、初参加はいつでも大歓迎なので、ぜひ〜🤗

https://mito-web-engineer.connpass.com/
