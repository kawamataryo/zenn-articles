---
title: "Svelte + WebSocketでシンプルなチャットアプリを作る"
emoji: "🧣"
type: "tech"
topics: ["svelte", "WebSocket", "typescript"]
published: true
---

Svelte と WebSocket の勉強のため簡単なチャットアプリを作ったので備忘録としてまとめます。

# 基本
## Svelte とは？
Svelte は React や Vue.js のように宣言的 UI で Web アプリケーションを作れるツールです。Vue.js の SFC のように、`.svelte`ファイルで HTML、JavaScript、CSS を単一コンポーネントとして管理し、それを組み合わせてアプリケーションを構築していきます。

React や Vue.js との大きな違いは、事前にアプリケーションコードを純粋な JavaScript ファイルとしてコンパイルすることで、クライアントではネイティブの DOM スクリプトとして実行される点です。仮想 DOM を使用しないため、ビルド後のコードには専用のランタイムライブラリが含まれず軽量です。

https://svelte.dev/

## WebSocket とは？
WebSocket は単一の TCP コネクション上で双方向通信を実現するための通信プロトコルです。WebSocket を使うことで、一度通信を確立すれば通常の HTTP 通信では実現できないサーバー起点のデータ送信が可能になります。そのため従来は pooling などで実現していたリロードなしのリアルタイムのデータ更新が容易に実装できます。

![](https://i.gyazo.com/fa06a2e23ea20d9fe9746c9fb3b11841.png)


https://livelibrary.osisoft.com/LiveLibrary/content/en/web-api-v8/GUID-EA5DCD55-A957-4B47-9556-22A7920BD82F#addHistory=true&filename=GUID-021731EE-C9B6-4236-9BE5-1ED92C43B319.xml&docid=GUID-EA5DCD55-A957-4B47-9556-22A7920BD82F&inner_id=&tid=&query=&scope=&resource=&toc=false&eventType=lcContent.loadDocGUID-EA5DCD55-A957-4B47-9556-22A7920BD82F

# チャットアプリの作成
以下のように別ブラウザで開いていても、リアルタイムで同期的に動くチャットアプリを作りました。

![](https://i.gyazo.com/0c5b48f575bc2cb3d758aba7daafb7c2.gif)

ソースコードはこちらにあります。

https://github.com/kawamataryo/svelte-simple-chat

以下ポイントだけメモします。

## サーバー側の実装
サーバー側の実装はとても簡単です。
Node.js で WebSocket を扱うためのライブラリである[ws](https://github.com/WebSockets/ws)を使って実装しています。

https://github.com/websockets/ws

```ts
import WebSocket from "ws"

const wss = new WebSocket.Server({ port: 8082 });

type Chat = {
  userId: string,
  message: string
}

// インメモリでデータを保持
const chatBoard: Chat[] = [];

wss.on('connection', (ws) => {
  ws.on('message', (payload) => {
    if(typeof payload === 'string') {
      chatBoard.push(JSON.parse(payload));
    }

    // 全ての接続先に送信
    wss.clients.forEach((client) => {
      client.send(JSON.stringify(chatBoard));
    })
  });

  ws.send(JSON.stringify(chatBoard));
});
```

`new WebSocket.Server`でサーバーのインスタンスを作り、`wss.on('connect', callback)`で WebSocket 接続時の処理を。接続時にコールバックで受け取る`ws`を使って、`ws.on('message', callback)`でリクエスト受信時の処理を書いています。

リクエスト受信時には、インメモリで保持しているチャットのメッセージ配列の更新を行うとともに、`wss.clients`で全ての接続中のクライアントを取得し、それぞれのクライアントに対して`send`で新しいチャットメッセージの配列を送っています。`ws.send`を使った場合、リクエストを送信したクライアントだけにメッセージを送ることになり、別ブラウザでの同期的な表示はできないので注意です。

## Client側の実装

Client 側では Svelte の TypeScript テンプレートを元にチャットアプリを作っています。
TypeScript でのプロジェクト作成はこちらを参考にしました。

https://svelte.dev/blog/svelte-and-typescript

コンポーネントはボード全体を表示する`App.svelte`と、ひとつひとつのメッセージを表示する`Message.svelte`と、メッセージの入力フォームを表示する`MessageForm.svelte`の 3 つです。

まず[Message.svelte](https://github.com/kawamataryo/svelte-simple-chat/blob/master/client/src/components/Message.svelte)です。
Svelte での Props は、script タグの中で`export`している変数となります。このコンポーネントでは、`currentUserId`・`userId`・`message`の 3 つの Props を受け取り、その内容をもとにメッセージを表示しています。
その他、`class:mine="{currentUserId === userId}"`の部分で、currentUserId と userId が一致する場合のみ`mine`という class を付与しています。
アバターの画像は、[Avatars](https://avatars.dicebear.com/)という文字列からアバター画像を生成してくれる API を利用しています。

https://avatars.dicebear.com/


```html:client/src/components/Message.svelte
<script lang="ts">
  export let currentUserId: string;
  export let userId: string;
  export let message: string;
</script>

<div class="message-container" class:mine="{currentUserId === userId}">
  <img width="40px" src="{`https://avatars.dicebear.com/api/bottts/` + userId + '.svg'}" alt="Avatar" />
  <div class="balloon">{message}</div>
</div>

<style>
/* ... */
</style>
```

次に[MessageForm.svelte](https://github.com/kawamataryo/svelte-simple-chat/blob/master/client/src/components/MessageForm.svelte)です。
このコンポーネントでは親コンポーネントにクリックイベントを送るために、`createEventDispatcher`を使って送信ボタンのクリックで submit イベントをディスパッチしています。`createEventDispatcher`は Vue でいう emit と同様ですね。また、親コンポーネントから受け取った`message`Props の内容を`bind:value`で input 要素に割当ています。親コンポーネント側で MessageForm コンポーネントの定義時に`bind:message`とすることで、透過的に message の変更を親コンポーネントに反映できます。

※ フォーム要素のコンポーネント間のデータのやり取りはこちらを参考にしました。

https://svelte.dev/tutorial/component-bindings

```html:client/src/components/MessageForm.svelte
<script lang="ts">
  import { createEventDispatcher } from 'svelte';

  export let message;

  const dispatch = createEventDispatcher();
  const handleSubmit = () => {
    dispatch('submit');
  };
</script>

<input bind:value="{message}" class="input" type="text" placeholder="Your message" />
<button class="button is-link" on:click="{handleSubmit}">submit</button>
```

最後に[App.svelte](https://github.com/kawamataryo/svelte-simple-chat/blob/master/client/src/App.svelte)です。
ライフサイクルフックの`onMount`で WebSocket の接続とメッセージ受信時のイベントを定義しています。メッセージ受信時には、内部で保持するリアクティブな変数である`messages`にデータを代入して、チャットデータの更新を行っています。
他、`socket.onopen`・`socket.onmessage`で WebSocket 接続時、メッセージ受信時の処理を登録しています。メッセージ受信時に受信データを`JSON.parse`でパースして`messages`に代入しています。

また、`handleSubmit`という関数で送信ボタン押下時に、WebSocket に対して追加のメッセージを送っています。先に`messages`の値を更新しているのは、自分起因のデータ更新は WebSocket の通信を待たず反映させるためです。

```html:client/src/App.svelte
<script lang="ts">
  import { onMount, tick } from 'svelte';
  import Message from './components/Message.svelte';
  import MessageForm from './components/MessageForm.svelte';
  import { getUserId } from './lib/getUserId';

  let message = '';
  let messages: { userId: string; message: string }[] = [];
  let socket: null | WebSocket = null;
  let board: null | Element;

  const currentUserId = getUserId();

  const scrollToBottom = async () => {
    await tick();
    board.scrollTop = board.scrollHeight;
  };

  const handleSubmit = () => {
    if (!message) {
      return;
    }
    messages = [...messages, { userId: currentUserId, message }];
    socket.send(JSON.stringify({ userId: currentUserId, message }));
    message = '';
    scrollToBottom();
  };

  onMount(() => {
    socket = new WebSocket('ws://localhost:8082');
    socket.onopen = () => {
      console.log('socket connected');
    };
    socket.onmessage = (event) => {
      if(!event.data) { return }
      messages = JSON.parse(event.data);
      scrollToBottom();
    };
  });
</script>

<main>
  <div class="wrapper">
    <h1>Svelte simple chat</h1>
    <div class="board" bind:this="{board}">
      {#each messages as { userId, message: mg }}
        <Message currentUserId="{currentUserId}" userId="{userId}" message="{mg}" />
      {/each}
    </div>
    <div class="message-form-wrapper">
      <MessageForm bind:message on:submit="{handleSubmit}" />
    </div>
  </div>
</main>

<style>
/* ... */
</style>
```

# 参考
- https://zenn.dev/toshitoma/articles/what-is-svelte
- https://svelte.dev/
- https://www.codegrid.net/articles/2020-svelte-1/
- https://ja.wikipedia.org/wiki/WebSocket
- http://wild-data-chase.com/index.php/2019/03/17/post-630/#outline__4
- https://www.html5rocks.com/ja/tutorials/websockets/basics/

# 終わりに
以上「Svelte + WebSocketでシンプルなチャットアプリを作る」でした。
Svelte も WebSocket も触りの触りですが、思いのほか使いやすかったです。また折りを見て書いていきたいです。