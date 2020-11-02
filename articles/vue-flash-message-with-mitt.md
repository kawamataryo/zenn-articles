---
title: "Mitt + Vue3でEvent駆動のFlashメッセージコンポーネントを作る"
emoji: "🥊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Vue", "TypeScript", "JavaScript", "Mitt"]
published: true
---

Mitt + Vue で Flash メッセージのコンポーネントを作る機会があったので備忘録としてまとめます。

# 作ったもの
Mitt と Vue を使って、以下 CodeSandbox のようなコンポーネントを作成しました。

https://codesandbox.io/s/flash-message-sample-on-vue-with-mitt-n4cwq?from-embed

@[codesandbox](https://codesandbox.io/embed/sandbox-flash-message-with-mitt-n4cwq?fontsize=14&hidenavigation=1&theme=dark)

この Flash メッセージ は以下の特徴を持ちます。

```
- 起動後、一定時間後に自動的に非表示となる
- どの階層のコンポーネントからも手軽に呼べる
```

フォームの完了メッセージや、その他通知などに汎用的に使えると思います。

# 実装のポイント
詳細のコードは、CodeSandbox を見てもらうとしてポイントのみまとめます。

## Mittによるpub/subで起動を制御する
Flash メッセージのコンポーネントは、コンポーネントの親子関係に依存せずどこからでも呼べるようにしています。
その実現のために、軽量なイベント管理ライブラリである[Mitt](https://github.com/developit/mitt)を使っています。

:::message
実はこの Event 駆動な実装は Vue2 系までは Mitt を使わずとも、Vue の EventBus で実装出来ました。
しかし、EventBus は Vue3 で廃止されています。Vue の公式ドキュメントでも EventBus の代わりに[Mitt](https://github.com/developit/mitt)や[tiny-emitter](https://github.com/scottcorgan/tiny-emitter)の利用が推奨されています。
https://v3.vuejs.org/guide/migration/events-api.html#overview
:::

Sample アプリのコンポーネントの概要は以下の通りです。

![](https://storage.googleapis.com/zenn-user-upload/mztsjzw3vbowims91r7tn9ctovcv)

Flash メッセージの表示を実行する Buttons コンポーネントと、FlashMessage コンポーネントでは直接の親子関係はありません。なので、通常のコンポーネントの親子関係で使える emit は使えず、mitt による event の pub/sub で表示を管理しています。

Flash メッセージコンポーネント側では onMounted のタイミングで Event の購読（publish）を設定しています。
`FLASH_EVENT`が発行されると、showMessage を実行して、Flash メッセージを表示します。

```ts:FlashMessage.vue
onMounted(() => {
  emitter.on<FlashPayload>(FLASH_EVENT, e => {
    if (!e) {
      return;
    }
    showMessage(e);
  });
});
```


そして、Buttons コンポーネント側では、ボタン押下での Event の発行（emit）を実行します。
以下 success カラーのボタンを押下した際の例ですが、 onSuccess が実行され、そこで Event の発行を行っています。
Event の引数として、Flash メッセージで表示するメッセージの内容と、表示色を設定しています。

```ts:Buttons.vue
const onSuccess = () => {
  emitter.emit<FlashPayload>(FLASH_EVENT, {
    message: "success message",
    color: "success"
  });
};
```

mitt の設定自体はとても簡単です。
emitter.ts で mitt のインスタンスを作り、export しているだけです。

```typescript:emitter.ts
import mitt from "mitt";

export type FlashPayload = { color: string, message: string }
export const FLASH_EVENT = Symbol("flashMessage")

export const emitter = mitt();
```

## 起動後一定時間後にFlashメッセージを非表示にする

一般的に Flash メッセージは情報の表示後、一定時間後に自動的に非表示になります。

それを実現するために Event の購読から showMessage が実行された際に、表示と同時に`setDelayedHide()`を実行して、`setTimeout()`で 3000ms 後に非表示としています。
`timeoutId`を保持して、`clearTimeout`を実行しているのは、連続して Event が発行された際に前の Event 時に設定したタイマーを破棄するためです。

```ts:FlashMessage.vue
setup() {
  const display = ref(false);
  const message = ref("");
  const color = ref("");
  const timeoutId = ref<number | null>(null);

  const hideMessage = () => {
    display.value = false;
  };

  const setDelayedHide = () => {
    if (timeoutId.value) {
      clearTimeout(timeoutId.value);
    }
    timeoutId.value = setTimeout(hideMessage, 3000);
  };

  const showMessage = (payload: FlashPayload) => {
    message.value = payload.message;
    color.value = payload.color;
    display.value = true;

    setDelayedHide();
  };

  //...
}
```

## 下からポップアップするアニメーション

下からポップアップするアニメーションは、Vue の transition を利用して実装しています。

![](https://storage.googleapis.com/zenn-user-upload/lfjij3xdlh1876ijw1sje2q62xdp)

transition は`v-if`, `v-show`などの表示を制御するコンポーネントを、`<transition>`で囲むことで、表示非表示の実行時に任意の class を付与できる機能です。
今回は以下のように、`popup`という name をもつ transition で Flash メッセージを囲み、css で popup のスタイルを実装してます。

初めてまともに transition を使ったのですが、割と気持ちの良いアニメーションが出来た気がします。

```markup:FlashMessage.vue
<template>
  <transition name="popup">
    <div
      class="notification _flash-message has-text-weight-bold"
      :class="`is-${color}`"
      v-show="display"
    >
      <button class="delete" @click="hideMessage"></button>
      {{ message }}
    </div>
  </transition>
</template>

<script lang="ts">
// ...
</script>

<style scoped>
// ...

.popup-leave-active {
  bottom: 30px;
  opacity: 1;
}
.popup-leave-to {
  bottom: -30px;
  opacity: 0;
}

.popup-enter-active {
  bottom: -30px;
  opacity: 0;
}
.popup-enter-to {
  bottom: 30px;
  opacity: 1;
}
</style>
```


# 終わりに

以上、「Mitt + Vue で Event 駆動の Flash メッセージのコンポーネントを作る」でした。
今回の Flash メッセージのように親子関係のないコンポーネント間での制御には Event 駆動の実装はとても便利ですね。
global な store を利用しても表示制御をすることも出来るのですが、store を抱えるのもやりすぎかな？　という時に重宝します。

何かコードに間違いや「もっとこうした方がやりやすいよ」などありましたら気軽にコメントを頂けると嬉しいです。