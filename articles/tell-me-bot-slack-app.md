---
title: "社内用語辞典を活性化するアプリを作る"
emoji: "➰"
type: "tech"
topics: ["javascript"]
published: false
---

# Firebase側の設定

## Spreadsheet APIの有効化

![](https://i.gyazo.com/096d11aec7af2d0da47008649706359e.png)

firebaseをデプロイ

デプロイすると作られる

deployする前に環境変数を設定

```
$ firebase functions:config:set slack.bot_token="xxxx"
$ firebase functions:config:set slack.signin_secret="xxxx"
$ firebase functions:config:set sheet.id="xxxx"
```

deploy

```
$ npm run deploy
```

service accountをコピー

![](https://i.gyazo.com/ec8ce09ed0d8fd19ea1caabdc189bf38.png)

シートの編集者に追加

![](https://i.gyazo.com/d2fd47584db04e46780a39c012c6ea73.png)

manifestのURLを変える

URLをeventsにするのを忘れずに。