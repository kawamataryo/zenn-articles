---
title: "Mock Service Worker で開発用のモックAPIを作る"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "mockserviceworker"]
published: true
---

フロントエンドの開発時に仮の API を使いたいってシチュエーションはわりとあると思います。
そんな時に、[Mock Service Worker](https://github.com/mswjs/msw) を使うと便利だったのでまとめます。

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

[Vite](https://github.com/vitejs/vite) で作った Vue のプロジェクトで利用する場合を例にまとめます。

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

コマンドを実行すると public 配下に `mockServiceWorder.js` が追加されるはずです。


```bash
public/
├── favicon.ico
└── mockServiceWorker.js # 追加
```

`mockServiceWorder.js`では、Service Worker でのイベント処理が書かれています。

## モック API の定義を追加

プロジェクトの src 配下に`mock`ディレクトリを作成し、そこに`handler.js`と`browser.js`を追加します。

`handler.js`では、どの URL のリクエストに対して、どのようなレスポンスを返すのかを定義しています。

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

`browser.js` では `handler.js` で作成した定義をもとに Service Worker のセットアップを行っています。

```js:src/browser.js
import { setupWorker } from 'msw'
import { handlers } from './handlers'

export const worker = setupWorker(...handlers)
```

:::message
今回は REST API のモックのコードを例にあげていますが、Mock Service Worker では GraphQL の API でもモック可能です。
https://mswjs.io/docs/api/graphql
:::

## Service Workerのスタート処理の追加

これまでの作業で準備が出来たので、プロジェクトのエントリーポイントにService Worker のスタート処理を追加します。

Vue の`main.js`に追記する例です。

```js:src/main.js
import { createApp } from 'vue'
import App from './App.vue'

if (process.env.NODE_ENV === 'development') {
  const { worker } = require('./mocks/browser')
  worker.start()
}

createApp(App).mount('#app')
```

開発ビルドの場合のみ、Worker を起動しています。
これでプロジェクトのスタート時にモックが動作するようになります。

:::message
`worker.start()`は非同期処理です。モック対象の API を呼ぶタイミングによっては Service Worker の起動前に呼ばれてしまい、リクエストのインターセプトが動作しない場合もあります。その場合は以下 URL の Deferred mounting を試すと良さそうです。
https://mswjs.io/docs/recipes/deferred-mounting
:::

## モックの動作確認

実際にモックの動作を確認します。

`/users`のAPIを呼び出す処理を追加します。

```vue:components/App.vue
<template>
  <h1>Users</h1>
  <ul>
    <li v-for="user in users" :key="user.id">
      id: {{user.id}}, age: {{user.name}}
    </li>
  </ul>
</template>

<script lang="ts">
import { ref, defineComponent, onMounted } from 'vue'

type User = {
  name: string;
  age: number
}

export default defineComponent({
  name: 'App',
  setup() {
    const users = ref<User[]>([])

    onMounted(async () => {
      const res = await fetch('/users')
      users.value = await res.json()
    })

    return { users }
  }
})
</script>
```

`yarn dev`で Vue の開発サーバーを起動し、localhost にブラウザでアクセスすると Console に `Mocking enabled` と表示されるはずです。これが Mock Service Worker によるモックが動作していることを示します。

![](https://i.gyazo.com/d0a592bc9de1f17dcd1584fd82327297.png)

また、DevTools の Network を確認すると、Service Worker にインターセプトされた XHR リクエストが確認出来るはずです。

![](https://i.gyazo.com/4392bde6be6da2f1bc1f5603a1a33664.png)

レスポンスを見ると `handler.js` で定義した値が返却されています。
ページも問題なく描画されます。

![](https://i.gyazo.com/4392bde6be6da2f1bc1f5603a1a33664.png)

# 終わりに

以上、開発時の API リクエストのモック処理での Mock Service Worker の利用例でした。
別で開発サーバーのプロセスを立てるより、手軽にモックの API を使えるようになるのでとても良い感じだなと思いました。今後も使っていきたいです。