---
title: "SlackBotを作ったら社内用語辞典の運用が3倍楽しくなった話"
emoji: "🦄"
type: "tech"
topics: ["slack","firebase","boltjs","typeScript","gcp"]
published: false
---

最近作った SlackBot が好評だったので、まとめてみました。
どこのSlackワークスペースでも導入できるように詳細に設定方法も記載しています。
# 🛠 作ったもの
tell-me-bot（社内では tell-me-paccho）という、社内用語辞典を**いい感じ**に管理してくれる SlackBot を作りました。

社内ではもともとスプレッドシートで社内用語を管理していたのですが、メンテナンスする人が限られ、あまり積極的には利用されていない状況でした。

そんな時に、[@しかじろう](https://twitter.com/shikajiro)さんの[こちらの記事](https://zenn.dev/shikajiro/articles/11b9e188ce6e94)を発見して、これはおもしろいアイデアだと思い、正月に Firebase + Bolt（TypeScript）にて作ってみました（アイデアをくれた[@しかじろう](https://twitter.com/shikajiro)さんに感謝🙏）。元記事の機能を参考に+αの機能も色々実装しています。

https://twitter.com/KawamataRyo/status/1480732294134796288
## 構成

Firebase for Cloud Functions で Slack アプリのフレームワークである [Bolt.js](https://github.com/slackapi/bolt-js) を動かし、Slack の EventAPI や ActionAPI のリクエストに応じて社内用語辞典のスプレッドシートを操作しています。
また、後述する曖昧検索の実現のために、[Fuse.js](https://github.com/krisk/fuse) を内部的に利用しています。
スプレッドシートとの通信は Cloud Functions のサービスアカウトを利用して [googlea-api-nodejs-client](https://github.com/googleapis/google-api-nodejs-client)を使っています。

![](https://i.gyazo.com/31e3d4b9465986f16a6b74f89dda9f5a.png)

実装も全て GitHub で公開しています（⭐を貰えると泣いて喜びます）。
https://github.com/kawamataryo/tell-me-bot

# 💡 機能詳細
tell-me-bot の機能を紹介します。
## 用語の表示
完全一致する用語を tell-me-bot にメンションした場合は、即結果が表示されます。もし説明が間違っていた場合は、メッセージ記載のスプレッドシートから編集できます。

![](https://i.gyazo.com/782e65b1d13566e25238295ddbb1ab5b.gif)

## 曖昧検索への対応
もし、完全一致はしないけれども似たような単語がある場合は、候補を表示します。候補のボタンを押すと、その用語の説明を答えてくれます。

内部的には Fuse.js の Fuzzy Search を利用しています。
これがこのボットの一番の売りの機能です。曖昧な記憶からも正確な情報を取得できます。

https://github.com/krisk/fuse

![](https://i.gyazo.com/a9ae6538be8c5119fafebd0339fa6b8b.gif)

## 用語の追加
もし曖昧検索の結果が間違っていたり、検索に該当する用語がない場合は、Slack のモーダルから用語を新規に追加することもできます（スプレッドシートを直接編集することでも可能です）。
手軽に追加できるので、気づいたときにどんどん用語を増やすことができます。


![](https://i.gyazo.com/9e878a49c8a675bf161761a1949193e5.gif)

## 質問チャネルへの代理質問（Optional）
用語を新規に登録しようにも、用語の意味がわからない場合が多いですよね。その場合は、社内の質問チャネルに tell-me-bot が代理質問する機能もあります。
自分の所属する [LAPRAS社](https://lapras.com/) では、`#ask-anything` という以下の理念で運用されているチャネルがあるのでそこに質問を投げるようにしています。

> #ask-anything  
> LAPRASに関するあらゆることについて、誰でもなんでもいつでも何回でも質問してよいチャンネル。大事なことは、質問されたことにLAPRASの誰かがかならず反応すること。反応は早ければ早いほどよい(回答が得られなくても仕方ない。反応があることがだいじ)。

:::message
#ask-anything チャネルの元ネタは自分の前職 [Misoca](https://www.misoca.jp/) の#asky-anything チャネルです。
:::

実際に以下のように投げられて、都度専門知識をもつメンバーが用語を登録してくれてとても助かっています。

|質問したチャネル|ask-anythingチャネル|
|---|---|
|![](https://i.gyazo.com/5f6a55b4ee64c3cc74817947d028382a.png)|![](https://i.gyazo.com/365d0ed3ff369edd70478cca27176355.png)|

:::message
この機能は質問チャネルの存在が前提にあるのでオプショナルです。もし利用したい場合は、後述するFirebaseの設定にて環境変数の`slack.ask_channel_id`に質問チャネルのチャネルIDを指定してください。
:::

# 💬 社内の声

会社の Slack でも嬉しい声を頂いたので一部紹介します！
やはり使って感想をもらえるのは開発者として一番嬉しいですね😆

|社内の声|
|---|
|![](https://i.gyazo.com/f79a7303c733e8ceb51fde3d2086fe4d.png)|
|![](https://i.gyazo.com/44ae53e0e045dc401c763fb3d1d43b99.png)|
|![](https://i.gyazo.com/201fdc6e86c4c04785258eb304c85a45.png)|
|![](https://i.gyazo.com/f41a1a7e553201bad7ed163500591a84.png)|

実際に社内用語辞典は急速に充実していっています。
個人開発のアプリを気軽に紹介できて、ポジティブなフィードバックを貰える社内環境はとても良いなーと思っています。

# 👨‍💻 設定方法
基本的にコードは全て公開しているので、以下手順でどの Slack にも導入できるはずです。

## 事前準備
事前に以下を準備します。

- [Firebase CLI](https://firebase.google.com/docs/cli)
  - 任意のシェルでセットアップしておいてください
- 空の Firebase プロジェクト
  - Blaze プラン（従量課金プラン）にプラン変更してください
- 任意の Spreadsheet
  - 1 枚目のシートの A1 に用語、B1 に説明というヘッダーを追加しておいてください
  - Spreadsheet の ID をメモしておいてください（[参考](http://amehal.blogspot.com/2015/10/id.html))

:::message
Firebase を Blaze プラン（従量課金プラン）としているのは、通常のプランだと Cloud FUnctions for Firebase の公開が制限されるからです。基本的にこの用途程度の利用なら費用はかからないと思います。
:::

## Slack APPの作成（1）
Slack の Your Apps にアクセスして `Create New App` を押します。

https://api.slack.com/apps 

選択後表示されるモーダルで `From app manifest` を選択します。

![](https://i.gyazo.com/e97427889949a1047ee55a8f58d5c2d1.png)

次にアプリを追加する Slack ワークスペースを選択します。

![](https://i.gyazo.com/7c0fcf43a2df60582295a1fafa34b6e2.png)

次に manifest の入力欄が表示されるので以下を入力します。

```yaml
display_information:
  name: tell-me-bot
  description: tell-me-bot
  background_color: "#323336"
features:
  bot_user:
    display_name: tell-me-bot
    always_online: true
oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - chat:write
      - im:history
      - im:read
      - im:write
      - users:read
      - channels:read
settings:
  event_subscriptions:
    request_url: https://example.com
    bot_events:
      - app_mention
      - message.im
  interactivity:
    is_enabled: true
    request_url: https://example.com
  org_deploy_enabled: false
  socket_mode_enabled: false
  is_hosted: false
  token_rotation_enabled: false
```

![](https://i.gyazo.com/6acf312ed83e83fe45eeb6a09ec420aa.png)

最後に確認が表示されるので、create ボタンを押せばアプリが作成されます。

![](https://i.gyazo.com/f19307f592b2de2e91b295062322384e.png)

アプリが作成されたら以下 2 の値をメモしておきます。
 - Basic Information > App Credentials の `Signin Secret` 
 - OAuth & Permission > OAuth Tokens for Your Workspace の `Bot User OAuth Token`

![](https://i.gyazo.com/55a9b4766cb98657ca7c361d17a3c298.png)
![](https://i.gyazo.com/f58f84335fdf3ba4b42c6c783491b7f4.png)

この値は次の Firebase の設定で使います。

## Firebase プロジェクトのデプロイ

tell-me-bot のリポジトリを任意のディレクトリでクローンします。

```bash
$ git clone https://github.com/kawamataryo/tell-me-bot
$ cd tell-me-bot
```

最初に Firebase CLI でプロジェクトの設定を行います。
プロジェクトの ID は `firebase projects:list` で確認できます。

```
$ firebase use <事前準備で作成したプロジェクトのID>
```

そして Firebase の環境変数を設定します。
ここで今まででメモしてきた値を設定します。

```bash
# Slack APPの作成でメモしたBot User OAuth Tokenの値
$ firebase functions:config:set slack.bot_token="xxxx"

# Slack APPの作成でメモしたSignin Secretの値
$ firebase functions:config:set slack.signin_secret="xxxx"

# 質問チャネルがある場合は、質問チャネルのIDを指定。ない場合は空文字を指定。
$ firebase functions:config:set slack.ask_channel_id=""

# 事前準備で用意したSpreadsheetのID
$ firebase functions:config:set sheet.id="xxxx"
```


次に functions ディレクトリの依存モジュールを install します。

```
$ npm --prefix ./functions i
```

最後にデプロイです。

```
$ npm --prefix ./functions run deploy
```

完了後のログで functions の URL が確認できるのでメモします。

![](https://i.gyazo.com/a4c8183f8cd031a1afc433f6026d5358.png)
## GCPの設定

GCP のダッシュボードにアクセスし、Firebase で作成したプロジェクトを指定してください。

https://console.cloud.google.com/apis/dashboard?hl=ja

そしてメニューから Google Sheets API を開き有効化します。
これは tell-me-bot の内部で Spreadsheet の操作に Google Sheet API を利用しているためです。

![](https://i.gyazo.com/096d11aec7af2d0da47008649706359e.png)

次に、メニューからサービスアカウントを開きます。
2 つのサービスアカウントが作成されていると思うので、`App Engine default service account`の方のメールアドレスをメモします。

最後に、事前準備で作成した Spreadsheet を開き、先程メモしたアカウントに対して編集権限を設定します。

![](https://i.gyazo.com/32a4ca88d9698ad7978a70d5517f9486.png)

これで GCP 側の準備は完了です。

## Slack APPの設定（2）
Slack APP に戻り、App Manifest を開きます。
そこで、manifest の設定で以前`example.com`としていた箇所を、先程メモした fucntions の `デプロイURL/events` を記載します。必ず後ろに`/events`をつけてください。

![](https://i.gyazo.com/dbd2256f80fd5cc112e9321fd67ef6e8.png)

そして Save Change を押します。ここで上部に Verify Error と出る場合は functions の環境変数の設定か URL が間違っている可能性があるので確認してください。functions の log を見るとわかりやすいです。

![](https://i.gyazo.com/edfb37ebac7a15c19d293fc9c513e377.png)

最後に Slack APP を指定のワークスペースにインストールします。

Install App から `Install Workspace` を実行してください。

![](https://i.gyazo.com/3f1ad96a6144880d606b5675e827db75.png)

## 動作確認
インストールした Slack Work スペースの任意のチャネルに tell-me-bot を追加してメンションしてみてください。

![](https://i.gyazo.com/8fddd854afdb412c1c6780fdb4bc4b33.png)

bot が答えてくれれば全て完了です。もしエラーが出る場合は Firebase のログを確認してください。
# おわりに
個人的にもとても便利で、どの会社にもあって良いなと思ってます。使ってもらえると嬉しいです！
設定方法に詰まりましたら、TwitterDM か記事コメントいただければ、ベストエフォートでサポートします！

https://twitter.com/KawamataRyo