---
title: "Cloud Functions for Firebase で CORS エラーを回避する"
emoji: "🌐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "cors", "javascript"]
published: true
---

Firebase プロジェクトとは別サーバーにデプロイしているアプリから Cloud Functions for Firebase で公開している API を叩いたら CORS エラーが起こったので回避方法をメモします。

# 事象

Firebase プロジェクトとは別ホストにデプロイしているアプリから Functions の API をリクエストすると CORS エラーが発生する。

:::message
CORS についてはこちらの本がめちゃくちゃ分かりやすいです。
https://zenn.dev/jxck/books/origin-anatomia
:::

![](https://i.gyazo.com/b24072c8252e93dca6ee1766ceb3c1de.png)


# 回避策

[cors](https://www.npmjs.com/package/cors) のパッケージ経由でレスポンスを返す or レスポンスに `Access-Control-Allow-Origin` を設定する。

## cors 経由でレスポンスを返す方法

```bash
npm i cors
```

```js
const cors = require("cors")({ origin: true });

export const foo = functions.https.onRequest((req, res) => {
    cors(req, res, () => {
      res.send({
        data: "foo"
      })
    });
  });
```

## Access-Control-Allow-Origin を設定する方法

```js
const cors = require("cors")({ origin: true });

export const foo = functions.https.onRequest((req, res) => {
    res.set("Access-Control-Allow-Origin", "*");
    res.send({
        data: "foo"
    });

  });
```

# 参考

- https://stackoverflow.com/questions/42755131/enabling-cors-in-cloud-functions-for-firebase