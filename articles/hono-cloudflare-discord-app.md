---
title: "Hono + Cloudflareでもくもく会用のDiscord Botを作ってみた"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["hono", "cloudflare", "discord"]
published: false
---

# 🤖 作ったもの
茨城県水戸市で毎月[もくもく会](https://mito-web-engineer.connpass.com/) を開催してもうすぐ5年になります。今回そのコミュニティのチャットツールをSlackからDiscordに移行するタイミングで、もくもく会の運営を補助するDiscord Botを作りました。

[Slackで使っていたBot](https://qiita.com/ryo2132/items/1a7d5e2a1b80414700e3)はBolt + Firebaseで作っていたのですが、今回は最近話題のHonoとCloudflare Workersを使っています。

主要機能は以下です。

**📌 チェックイン**
`/checkin` コマンドでもくもく会のチェックインを行えます。モーダルが表示され、自己紹介と今日やることを入力して送信すると、フォーマットされた投稿がチャンネルに投稿されます。

もくもく会では開始前に、参加者にチェックインを実行してもらい、この投稿内容をもとにはじめの共有を行なっています。これがあることで主催側としても、参加者の顔と名前を覚えやすくて助かっています。

また、Cloudflare の D1 に投稿内容を保存しているので、ほぼ内容が変わらない自己紹介は2回目以降自動で入力されます。

![Image from Gyazo](https://i.gyazo.com/9a7ee44ccaaa1c297cdad0bf792f56c4.gif)


**📌 もくもく会の開始・終了の通知**
`/mokumoku-start`コマンドで、もくもく会のスケジュールやルールを投稿します。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-09-46.png =550x)

さらに、このコマンドが実行された日は、15:00にテックトーク（中休みに行う技術雑談）の通知、17:50に終了の通知も送られるようになっています。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-09-58.png =550x)
![](/images/hono-cloudflare-discord-app/2024-03-06-09-10-09.png =550x)

もくもく作業に集中してしまい予定時刻を過ぎることが多々あるのでこれも便利です。

:::message
テックトークは[LAPRASのTech News Talk](https://www.youtube.com/playlist?list=PLKbaztxP2P4jpdF0P5YbJNJwFabB-pksK)を参考に、最近の気になったテック系ニュースや、お気に入りのツールを紹介するものです。特に資料も準備せず、各々好きな方法で発表してもらっています。その後の会話のきっかけにもなるので良いなと思っています。
:::

**📌 次回イベントの概要の作成**
`/generate-event-description`コマンドで、次回のもくもく会の概要を作成してくれます。この概要のなかで、前回のイベントの紹介の文章があるので、そこに前回イベントのURLと前回皆の入力してくれた「今日やること」の文言を埋め込んでいます。

![](/images/hono-cloudflare-discord-app/2024-03-06-09-10-54.png =550x)


# 🛠️ 実装紹介

このBot特有の実装について簡単に紹介します。
コードはすべて以下リポジトリで公開しています。[Discord.js](https://discord.js.org/)を使わないDiscord Botの実装例はまだあまりないと思うので、誰かの参考になれば嬉しいです。

https://github.com/Ibaraki-dev/mokumoku-bot

また、Hono + Cloudflare 環境での Discord Bot の開発全般については以下記事が大変参考になりました。感謝🙏

https://blog.lacolaco.net/posts/discord-bot-cfworkers-hono/

## Interactionsのハンドリング

今回のBotでは複数のスラッシュコマンドやモーダルの送信をハンドリングする必要があるので、`/interaction`ルートで、ApplicationCommandObjectのTypeによって処理を分岐させる形式を撮りました。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/app.ts#L21-L72

そしてTypeごとに登録された関数をループで回してを実行するようにしています。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/handleApplicationCommands.ts#L11-L44

実際に処理を行う関数ではDrizzle ORMでD1のデータを操作したりしながら、適切なレスポンスを整形します。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/applicationCommands/checkin.ts#L6-L29

## モーダルの表示
Botでモーダルを表示するには、まずSlack Commandのレスポンスでmodalのコンポーネントを返します。そして、modalの送信のリクエストをハンドリングして処理を行います。

以下のコードは、`/checkin`コマンドでモーダルを表示するためのコードです。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/applicationCommands/checkin.ts#L6-L29

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/responses/checkinCommandResponse.ts#L9-L52

そして、モーダルからの送信結果をハンドリングするためのコードです。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/interactions/modalSubmits/checkinModal.ts#L6-L54

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/responses/checkinModalSubmitResponse.ts#L20-L58


## Schedule Triggerを利用した通知の実装

`/mokumoku-start`コマンドが実行された日は、15:00と17:50に指定のチャネルにメッセージを送るようにしています。この機能は、Cloudflare WorkersのSchedule Triggerを利用しています。

実装はシンプルで、毎日15:00と17:50にSchedule Triggerが実行され、当日のEventモデル（`/mokumoku-start`コマンドで作成される）がある場合のみ、通知を送るというものです。

https://github.com/Ibaraki-dev/mokumoku-bot/blob/main/src/scheduled.ts#L10-L43

:::message
Schedule Triggerのcronの指定はUTC基準なので、日本時間の場合は9時間ずらして指定する必要があります。
:::

# 🔥 Hono + Cloudflareの良かったところ
今回はじめてHonoやCloudflare、Drizzleを使って開発したので簡単に所感をまとめます。

**📌 開発スタート・デプロイが爆速**
`bunx create-hono` でHonoのプロジェクトを作り、`bunx deploy` でデプロイするだけでCloudflareにデプロイされます。追加設定はLinter・Formatterくらいで、他は何も設定することなく開発をスタートできます。開発スタートからデプロイまでのスピードがとても速いです。また、Cloudflareのスタック（D1など）を利用する際にも、HonoのBindingsで簡単に追加できるのも良いですね。

**📌 Hono、Drizzle ORMの型補完が強力**
HonoとDrizzle ORMは型補完が強力で、APIを調べずとも補完に任せてほぼ迷うことなくコードを書くことができました。TypeScriptの機能を最大限に活用できるので、開発がとても楽でした。

**📌 Testが書きやすい**
Honoには便利なTestHelperが提供されているのでAPIのテストがとても楽です。今回のBotでもほぼすべての処理にテストを書いています。

**📌 @yusukebeさんの存在**
Honoの開発者でCloudflareに在職してる[@yusukebe](https://twitter.com/yusukebe)さんの存在も大きいです。@yusukebeさんのXやブログからCloudflareやHono最新の情報を日本語でキャッチアップできます。今回のBot開発時にZennのスクラップでメモしていたら、[Yusukebeさんからアドバイス](https://zenn.dev/link/comments/8a912a10634481)をもらえました。感謝。

# 🏁 おわりに

Discord Botを作るのもHonoやCloudflareを使うのも今回はじめての試みで、いろいろ詰まるところはあったのですが、HonoとCloudflareの開発体験がとても良く楽しく作れました。

今後、他のアプリもこの技術スタックで作ってみたいなと思っています。

最後に宣伝です！
Ibaraki.devでは毎月もくもく会を開催しています。ゆるい勉強会で、初参加はいつでも大歓迎なので、ぜひ〜🤗

https://mito-web-engineer.connpass.com/
