---
title: "SlackBotを作ったら社内用語辞典の運用が100倍楽しくなった話"
emoji: "💬"
type: "tech"
topics: ["slack","firebase","boltjs","typeScript","spreadsheet"]
published: false
---

最近作った SlackBot の紹介です。
# 作ったもの
tell-me-bot（社内では tell-me-paccho）という社内用語辞典を**いい感じ**に管理してくれる SlackBot を作りました。

社内ではもともとスプレッドシートで社内用語を管理していたのですが、メンテナンスする人が限られて、あまり積極的に利用されていない状況でした。

そんな時に、[@しかじろう](https://twitter.com/shikajiro)さんの[こちらの記事](https://zenn.dev/shikajiro/articles/11b9e188ce6e94)を発見して、これはおもしろいアイデアだと思い、正月に Firebase + Bolt（TypeScript）にてフルスクラッチで作ってみました（しかじろうさんに感謝）。元記事のアイデアを元に+αの機能も多く追加しています。

https://twitter.com/KawamataRyo/status/1480732294134796288
## 構成

Firebase for Cloud Functions で Slack アプリのフレームワークである [Bolt.js](https://github.com/slackapi/bolt-js) を動かし、Slack の EventAPI や ActionAPI のリクエストに応じて社内用語辞典のスプレッドシートを操作しています。
また、後述する曖昧検索の実現のために、[Fuse.js](https://github.com/krisk/fuse) を内部的に利用しています。
スプレッドシートとの通信は Cloud Functions のサービスアカウトを利用して kj[googlea-api-nodejs-client](https://github.com/googleapis/google-api-nodejs-client)を使っています。

![](https://i.gyazo.com/31e3d4b9465986f16a6b74f89dda9f5a.png)

実装も全て GitHub で公開しています（Starを貰えると泣いて喜びます）。
https://github.com/kawamataryo/tell-me-bot

# 機能詳細
## 用語の表示
完全一致する用語を tell-me-paccho にメンションした場合は、即結果が表示されます。もし間違っていた場合は、メッセージ記載のスプレッドシートから編集します。

![](https://i.gyazo.com/782e65b1d13566e25238295ddbb1ab5b.gif)

## 曖昧検索への対応
もし、完全一致はしないけれども似たような単語がある場合は、一致度が高い順で候補を表示します。候補のボタンを押すと、その用語の説明を答えてくれます。

内部的には Fuse.js の Fuzzy Search を利用しています。
これがこのボットの一番の売りの機能です。曖昧な記憶からも正確な情報を取得できます。

https://github.com/krisk/fuse

## 用語の追加
もし曖昧検索の結果が間違っていたり、検索に該当する用語がない場合は、Slack のモーダルから用語を新規に追加することもできます（スプレッドシートを直接編集することでも可能です）。
手軽に追加できるので、気づいた時にどんどん用語を増やす事ができます。


![](https://i.gyazo.com/9e878a49c8a675bf161761a1949193e5.gif)

## 質問チャネルへの代理質問（Optional）
用語を新規に登録しようにも、用語の意味がわからない場合が多いですよね。その場合は、社内の質問チャネルに tell-me-bot が代理質問する機能もあります。
LAPRAS では、#ask-anything という以下の理念で運用されているチャネルがあるのでそこに質問を投げるようにしています。

> #ask-anything  
> LAPRASに関するあらゆることについて、誰でもなんでもいつでも何回でも質問してよいチャンネル。大事なことは、質問されたことにLAPRASの誰かがかならず反応すること。反応は早ければ早いほどよい(回答が得られなくても仕方ない。反応があることがだいじ)。

実際に以下のように投げられて、専門知識をもつメンバーが用語を登録してくれてとても助かっています。

**質問したチャネル**
![](https://i.gyazo.com/5f6a55b4ee64c3cc74817947d028382a.png)

**ask-anythingチャネル**

![](https://i.gyazo.com/365d0ed3ff369edd70478cca27176355.png)


:::message
この機能は質問チャネルの存在が前提にあるのでオプショナルです。LAPRAS の GitHub リポジトリの tell-me-bot の実装を参考に設定してください

https://github.com/lapras-inc/tell-me-bot/blob/main/functions/src/v1/bolt/actions/useAskAction.ts
:::

:::message
#ask-anything チャネルの元ネタは自分の前職 [Misoca](https://www.misoca.jp/) の#asky-anything チャネルです。
:::

# 同僚の声

LAPRAS 社の Slack でも嬉しい声を頂いたので一部紹介します！
やはり使って感想をもらえるのは開発者として一番嬉しいですね😆

![](https://i.gyazo.com/f79a7303c733e8ceb51fde3d2086fe4d.png)
![](https://i.gyazo.com/44ae53e0e045dc401c763fb3d1d43b99.png)
![](https://i.gyazo.com/201fdc6e86c4c04785258eb304c85a45.png)
![](https://i.gyazo.com/f41a1a7e553201bad7ed163500591a84.png)

個人開発のアプリを気軽に紹介できて、ポジティブなフィードバックを貰える社内環境はとても良いなーと思っています。
# 設定方法
基本的にコードは全て公開しているので、以下手順でどの Slack にも導入できます。
# Firebase側の設定

## Spreadsheet APIの有効化

![](https://i.gyazo.com/096d11aec7af2d0da47008649706359e.png)

firebase をデプロイ

デプロイすると作られる

deploy する前に環境変数を設定

```
$ firebase functions:config:set slack.bot_token="xxxx"
$ firebase functions:config:set slack.signin_secret="xxxx"
$ firebase functions:config:set sheet.id="xxxx"
```

deploy

```
$ npm run deploy
```

service account をコピー

![](https://i.gyazo.com/ec8ce09ed0d8fd19ea1caabdc189bf38.png)

シートの編集者に追加

![](https://i.gyazo.com/d2fd47584db04e46780a39c012c6ea73.png)

manifest の URL を変える

URL を events にするのを忘れずに。