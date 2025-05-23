---
title: "Vue経験者向け Vue3 スタートガイド [実行環境付き]"
emoji: "🚴‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "typescript"]
publication_name: "lapras_inc"
published: true
---

[LAPRAS](https://corp.lapras.com/recruit-engineer/) 社内で開催した Vue3 勉強会の資料を一部内容を変更して公開します。
Vue3 の新機能、Breaking Change に対して Codesandbox でひとつひとつ実行環境を作っています。動作・コードを検証しながら読んでもらえると嬉しいです。

# 🙋‍♂️ 対象読者

- Vue2 を Vue3 に移行中 or 移行検討中の人
- 普段 Vue は触らないけど、教養的に Vue3 をキャッチアップしたい人
- Vue3 の機能・注意点を普段フロントエンドを触らないメンバーに説明したい人

:::message
Vue3 への移行方法ではないです！
Vue3 に移行したら何が変わるか、新機能、Breaking Change をまとめています。
:::

# 🏎️ Vue3 で何が変わる？

https://v3-migration.vuejs.org/

- 新たな API の追加
  - [Teleport](https://ja.vuejs.org/guide/built-ins/teleport.html), [Fragments](https://developer.mozilla.org/ja/docs/Web/API/DocumentFragment)、[Suspense](https://ja.vuejs.org/guide/built-ins/suspense.html#suspense)など便利な API が多数追加
- リアクティブの改善
  - Vue2 は[Object.defineProperty](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)を利用したリアクティブの実装だったが、Vue3 では[Proxy](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Proxy) をベースに変更
- パフォーマンスの向上
  - 内部実装の変更で、バンドルサイズが低下、パフォーマンスも向上
  - ex: [LINE MUSIC のパフォーマンスを向上させた Vue3 マイグレーション](https://engineering.linecorp.com/ja/blog/vue3-migration-with-improved-line-music-performance)
- TypeScript 対応の強化
  - template 内部での TS 構文の利用が可能に
  - 型推論が改善

# 💫 新たにサポートされた API・構文は？

## CSS v-bind

https://vuejs.org/api/sfc-css-features.html#v-bind-in-css

CSS のプロパティに対してリアクティブな値のバインディングが可能になりました
内部的には、CSS 変数の値をリアクティブに更新することで値を書き換えています。Devtools で確認するとおもしろいです。

※ Vue2.7 でも利用可

```vue
<template>
  <div class="text">hello</div>
</template>

<script setup>
import { ref } from "vue";

const color = ref("red"); // bind対象
</script>

<style>
.text {
  color: v-bind(color); /* この値がリアクティブになる */
}
</style>
```

@[codesandbox](https://codesandbox.io/embed/vue-3-css-binding-nolhg9?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## Teleport

https://ja.vuejs.org/guide/built-ins/teleport.html

`<Teleport>`  は、コンポーネントにあるテンプレートの一部を、そのコンポーネントの DOM 階層の外側に存在する DOM ノードへ「テレポート」するものです。

Vue2 でも使えた [portal-vue](https://portal-vue.linusb.org/) と同等の機能です。新たなスタッキングコンテキストを生成することで z-index での重なり制御を回避できます。

```vue
<template>
  <div class="app">
    <Teleport to="body">
      <!-- Teleport内がbody直下に展開される -->
      <Modal>teleport</Modal>
    </Teleport>
  </div>
</template>
```

@[codesandbox](https://codesandbox.io/embed/vue3-teleport-urro7r?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## Fragments

https://v3-migration.vuejs.org/new/fragments.html

Fragments を使うことで Vue3 から`<template>`直下に同じ階層で複数の要素を配置できるようになりました。不要な wrapper の`div` を作らなくて済みます。

ただし、fragment を使っている場合、親要素から class などの属性を追加する際に子要素側で、明示的に`v-bind="$attrs"`を設定しないと属性が付与されないので注意です。さらに scoped css にしている場合は`:deep()`も必要です。

```vue
<template>
  <div>...</div>
  <div v-bind="$attrs">...</div>
  <!-- この要素に親要素側で指定した属性が付与される -->
  <div>...</div>
</template>
```

@[codesandbox](https://codesandbox.io/embed/vue3-fragments-0mbqxh?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## Suspense（Experimental）

https://ja.vuejs.org/guide/built-ins/suspense.html

`<Suspense>`  は、コンポーネントツリーの非同期な依存関係を制御するための組み込みコンポーネントです。コンポーネントツリーの下にある複数のネストされた非同期な依存関係が解決されるのを待つ間、ローディング状態をレンダリングできます。

非同期コンポーネントを使う際に有効だが、まだ Experimental な機能で API が変わる可能性もあるので使用は控えたほうが良いかもです。

```html
<Suspense>
  <!-- ネストされた非同期な依存関係を持つコンポーネント -->
  <Dashboard>

  <!-- #fallback スロットでローディング状態を表す -->
  <template fallback>
    Loading...
  </template>
</Suspense>
```

@[codesandbox](https://codesandbox.io/embed/vue3-suspense-gdwqny?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## script setup 構文

https://ja.vuejs.org/api/sfc-script-setup.html

単一ファイルコンポーネント（SFC）内で Composition API を使用する際のシンタックスシュガーです。以下のような利点があります。

- ボイラープレートが少なく、より簡潔なコードが書ける
- 純粋な TypeScript を使ってプロパティと発行されたイベントを宣言する機能
- 実行時のパフォーマンスの向上
- IDE で型推論のパフォーマンス向上

※ Vue2.7 でも利用可

```vue
<template>
  <ChildA />
  {{ text }}
  <button @click="click">{{ msg }}</button>
  <ChildB />
  <ChildC />
</template>

<script setup lang="ts">
// 子コンポーネントの定義
import ChildA from "./ChildA.vue";
import ChildB from "./ChildB.vue";
import ChildC from "./ChildC.vue";

// reactiveの定義
const text = ref("");

// propsの定義
const props = withDefaults(defineProps<{ msg?: string }>(), {
  msg: "hello",
});

// emitの定義
const emit = defineEmits<{
  (e: "change", msg: string): void;
}>();
const click = () => {
  emit("change", "ok");
};
</script>
```

:::details script setup を使わずに書いた場合は...

```vue
<template>
  <ChildA />
  {{ text }}
  <button @click="click">{{ msg }}</button>
  <ChildB />
  <ChildC />
</template>

<script setup lang="ts">
import { defineComponent, ref }
import ChildA from './ChildA.vue'
import ChildB from './ChildB.vue'
import ChildC from './ChildC.vue'

export default defineComponent(() => {
  name: "App",
  // 子コンポーネントの定義
  components: {
    ChildA,
    ChildB,
    ChildC,
  },
  // propsの定義
  props: {
    msg: {
      type: String,
      default: "hello"
    }
  },
  // emitの定義
  emits: ['change'],
  setup(_, { emit }) {
    // reactiveの定義
    const text = ref("")

    const click = () => {
      emit('change', "ok")
    }

    return {
      text,
      click,
    }
  }
})
</script>
```

:::

## Reactive API の追加

https://ja.vuejs.org/api/reactivity-utilities.html

リアクティブ周りの API が多数追加されてました。リアクティブの制御で困ったら使ってみても良いかもです。以下一部抜粋。

- [shallowRef()](https://ja.vuejs.org/api/reactivity-advanced.html#shallowref) `ref()`の浅いバージョン
- [customRef()](https://ja.vuejs.org/api/reactivity-advanced.html#customref) 依存関係の追跡と更新のトリガーを明示的に制御して、カスタマイズされた ref を作成
- [toRaw()](https://ja.vuejs.org/api/reactivity-advanced.html#toraw) Vue で作成されたプロキシの元のオブジェクトを返す
- [v-memo](https://ja.vuejs.org/api/built-in-directives.html#v-memoA) テンプレートのサブツリーのメモ化

# 🚨 注意すべき Breaking Changes は？

## v-model の仕様変更

https://v3-migration.vuejs.org/breaking-changes/v-model.html

Vue3 では v-model の仕様が諸々変更されました。

- v-model を受け取るカスタムコンポーネントの props、event 名が変更
  - prop: `value` -> `modelValue`
  - event: `input` -> `update:modelValue`
- `v-bind.sync`の廃止
- `v-model`の複数定義が可能に
- `v-model`の`modifier` が増加

### Vue2

@[codesandbox](https://codesandbox.io/embed/vue2-v-model-wbk6u0?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fcomponents%2FCustomForm.vue&theme=dark)

### Vue3

@[codesandbox](https://codesandbox.io/embed/vue-3-v-model-te3owy?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fcomponents%2FCustomForm.vue&theme=dark)

## 配列の要素の変更が標準で Watch されない

https://v3-migration.vuejs.org/breaking-changes/watch.html

配列の内部の要素の変更を`watch` で検知しようとしても、検知されなくなりました（配列自体の置き換えは検知する）。配列の要素の変更を検知する場合は、`deep` オプションをつけてください。

```js
watch(
  reactiveVal,
  (newVal, oldVal) => {
    console.log(`${oldVal} -> ${newVal}`);
  },
  {
    deep: true, // 配列の要素の変更にはこれが必要
  }
);
```

@[codesandbox](https://codesandbox.io/embed/vue-3-array-watch-s8530x?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## Array, Object の変更で Vue.set, Vue.remove の利用が不要に

Vue2 では配列や、[配列、オブジェクトのキー指定の書き換え・削除がリアクティブにならない](https://jp.vuejs.org/v2/guide/reactivity.html#%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%81%AB%E9%96%A2%E3%81%97%E3%81%A6)という注意点がありましたが、Vue3 では解消され、`Vue.set`や`Vue.remove` が不要になりました。

### Vue2

@[codesandbox](https://codesandbox.io/embed/vue2-reactive-array-obj-q0tb5k?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

### Vue3

@[codesandbox](https://codesandbox.io/embed/sweet-snyder-rc60fr?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## deep セレクターの書き方が変更

https://ja.vuejs.org/api/sfc-css-features.html#scoped-css

`scoped` 環境で子コンポーネントの要素に対してスタイルを当てたい場合のセレクターの指定方法が変更されました。`:deep()` で指定する必要があります。

```vue
<style scoped>
/* 子コンポーネントの`.b`に適応できる */
.a :deep(.b) {
  /* ... */
}
</style>
```

@[codesandbox](https://codesandbox.io/embed/vue-3-deep-qhjirb?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

## その他いろいろ

他にもいろいろある。詳細は[マイグレーションガイド](https://v3-migration.vuejs.org)を確認してください。

- [トランジション機能のクラス名変更](https://v3-migration.vuejs.org/breaking-changes/transition.html)
  - 古い書き方だと警告もでないので注意
- [`v-bind="obj"`は書く位置で動作が変わるように](https://v3-migration.vuejs.org/breaking-changes/v-bind.html)
  - 以前は、他の属性と被っていると、それで上書きされていた。Vue3 からは、後に書いた属性が優先されるように
- [vm.$listeners は削除](https://v3-migration.vuejs.org/breaking-changes/listeners-removed.html)
  - `$attrs`に統合されました。つまり`v-bind="$attrs"`でリスナーも登録される
- [`vm.$on`、`vm.$off`、`vm.$once` の削除](https://v3-migration.vuejs.org/breaking-changes/events-api.html#migration-strategy)
  - 同様のことをするには [mitt](https://github.com/developit/mitt) や [tiny-emitter](https://github.com/scottcorgan/tiny-emitter)などのライブラリが必要
- [`v-on.native` 修飾子の削除](https://v3-migration.vuejs.org/breaking-changes/v-on-native-modifier-removed.html)
  - `emits`に定義されていないイベントはすべてネイティブイベントと見做されるように
- [ライフサイクルフックの命名変更](https://v3-migration.vuejs.org/breaking-changes/vnode-lifecycle-events.html#migration-strategy)
  - `destroyed`は`unmounted`に、`beforeDestroy`は`beforeUnmount`に命名変更

# 💬 Q & A

## script setup 使う？

コンポーネントの記述方法が代わり、覚えることが増える & Grep で探しにくくなるという懸念はありますが、それ以上に記述量が大幅に減ってコンポーネントの見通しが良くなるという利点のほうが大ききので積極的に使っていきたいです。

## ref()と reactive()のどちらを使う？

基本的には `ref()` で良いと思います。`reactive()` だとリアクティブの消失に意識を向ける必要があるので。もし、独自 store を作りたいケースなどで、reactive を使う場合は、`toRef` `toRefs` を使って export 時に Ref に変換するのを忘れずに！

参考: [vue composition api における props のリアクティブ性について知っておくべきこと](https://zenn.dev/karino_m/articles/vue-composition-api-and-props)

reactive が消失するパターン

```js
const state = reactive({ name: "foo" });

// 1. 分割代入を行うパターン
const { name } = state;

// 2. プロパティを直接参照して代入するパターン
const name = state.name;

// 3. プロパティを直接参照して他の処理に渡すパターン
const { length } = useLength(state.name);
```

reactive の消失を防ぐには...

```js
const state = reactive({ name: "foo" });

// 1. 分割代入を行うパターン
const { name } = toRefs(state);

// 2. プロパティを直接参照して代入するパターン
const name = toRef(state, "name");

// 3. プロパティを直接参照して他の処理に渡すパターン
const { length } = useLength(toRef(state, "name"));
```

# 📔 参考資料

- [Vue.js - The Progressive JavaScript Framework | Vue.js](https://ja.vuejs.org/)
- [Vue 3 Migration Guide | Vue 3 Migration Guide](https://v3-migration.vuejs.org/)
- Special Thanks [@Shin2357](https://twitter.com/Shin2357)

# おわりに

以上 Vue3 スタートガイドでした。
Vue3 がリリースされてもう 2 年ほど経ちますが、まだまだ Vue2 で開発しているチームも多いと思います。そのようなチームが、いざ Vue3 移行を進める際の説明資料などに使ってもらえたら嬉しいです。

![](/images/vue3-upgrade-guide/capture2.png)
_社内勉強会の様子。Codesandbox で実際に編集しながら説明_

最後になりますが、LAPRAS ではソフトウェアエンジニアを募集中です！

採用資料

https://corp.lapras.com/recruit-engineer/

カジュアル面談

https://herp.careers/v1/laprasinc/640n2WllqS21
