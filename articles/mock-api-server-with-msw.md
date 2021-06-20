---
title: "Mock Service Worker で開発用のモックAPIを立てる"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["msw", "TypeScript", "Test"]
published: false
---

[Mock Service Worker](https://github.com/mswjs/msw) で開発用のモック API を立ててみたのでメモ。

# Mock Service Worker とは？
[Mock Service Worker](https://github.com/mswjs/msw) は、ネットワークレベルで API リクエストをインターセプトして mock のデータを返すためのライブラリです。API リクエストを含む処理のテストや、開発時の mock サーバーの代替として利用出来ます。

https://mswjs.io/

テストでの利用については以前こちらの記事でまとめました。

https://zenn.dev/ryo_kawamata/articles/introduce-mock-service-worker

今回は開発時のモック API としての利用について書きます。
開発用の API というと、[JSON Server](https://github.com/typicode/json-server)が有名ですが、Mock Service Worker では Service Worker を使ってリクエストを返すため、別プロセスでローカルサーバーを立てる必要もなく手軽に利用できます。

仕組みは公式のこちらのビデオがわかりやすいです。

https://youtu.be/HcQCqboatZk

# Mock Service Worker での開発用APIの立て方

[Vite](https://github.com/vitejs/vite) で作った Vue のプロジェクトで開発用サーバーを利用する場合を、例にまとめます。

## パッケージの追加

Mock Service Worker のパッケージをプロジェクトに追加します。

```bash
$ yarn add -D msw
```

## Service Workerのコードを生成

Mock Service Worker は Service Worker で API リクエストをインターセプトします。その Service Worker のコードをプロジェクトの公開ディレクトリに追加します。
ファイルは Mock Service Worker の提供する CLI を使って生成できます。


```js
$ npx msw init public/ --save
```

Vue プロジェクトの公開ディレクトリは`./public`なのでそちらを指定しています。

コマンドを実行する public 配下に `mockServiceWorder.js` が追加されるはずです。


```bash
public/
├── favicon.ico
└── mockServiceWorker.js # 追加
```

`mockServiceWorder.js`では、Service Worker でのイベント処理が書かれています。ブラウザでこれを読み込むことで Service Worker を使ったインターセプトを実現しています。

## モック API の定義を追加

どの URL に対して、どのようなレスポンスを返すのかモックの定義を行います。
プロジェクトの src 配下に`mock`ディレクトリを作成し、そこに`handler.js`と`browser.js`を追加します。

```js:src/handler.js
import { rest } from 'msw'

export const handlers = [
  rest.get('/users', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        {
          id: 1,
          name: 'foo',
        },
        {
          id: 2,
          name: 'bar',
        }
      ]),
    )
  }),
]
```

```js:src/browser.js
import { setupWorker } from 'msw'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
```
