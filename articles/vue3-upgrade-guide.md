---
title: "Vue3 ガイド"
emoji: "🍃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "typescript"]
published: false
---

LAPRAS 社内で開いた Vue3 の勉強会が好評だったので、一部内容を変更して資料を公開します！

# 🏎️ Vue3 で何が変わる？

https://v3-migration.vuejs.org/

- 新たな API の追加
  - Teleport, Fragments、Suspense など便利な API が多数追加
- リアクティブの改善
  - Vue2 は[Object.defineProperty](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)を利用したリアクティブの実装だったが、Vue3 では[Proxy](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Proxy) をベースに変更
- パフォーマンスの向上
  - 内部実装の変更で、バンドルサイズが低下、パフォーマンスも向上した
  - ex: [LINE MUSIC のパフォーマンスを向上させた Vue3 マイグレーション](https://engineering.linecorp.com/ja/blog/vue3-migration-with-improved-line-music-performance)
- TypeScript 対応の強化
  - template 内部での TS 構文の利用が可能に
  - 型推論が改善し、より安全に使えるように

# 💫 新たに追加された API

## script setup 構文

https://ja.vuejs.org/api/sfc-script-setup.html

単一ファイルコンポーネント（SFC）内で Composition API を使用する際のシンタックスシュガー。Vue2.7 でも使用できる。
以下のような利点がある。

- ボイラープレートが少なくて、より簡潔なコード
- 純粋な TypeScript を使ってプロパティと発行されたイベントを宣言する機能
- 実行時のパフォーマンスの向上
- IDE で型推論のパフォーマンス向上

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

<details><summary>script setupを使わずに書いた場合は...</summary>

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

</details>

## Teleport

https://ja.vuejs.org/guide/built-ins/teleport.html

`<Teleport>`  は、コンポーネントにあるテンプレートの一部を、そのコンポーネントの DOM 階層の外側に存在する DOM ノードに「テレポート」するもの。[portal-vue](https://portal-vue.linusb.org/) と同等の機能。新たなスタッキングコンテキストを生成することで z-index での重なり制御を回避できる。

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

## CSS v-bind

https://vuejs.org/api/sfc-css-features.html#v-bind-in-css

CSS のプロパティに対してリアクティブな値のバインディングが可能になった。
内部的には、CSS 変数の値をリアクティブに更新することで値を書き換えている 。

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

## Fragments

https://v3-migration.vuejs.org/new/fragments.html

Vue3 から`<template>`直下に同じ階層で複数の要素を配置できるようになった。不要な wrapper の`div` を作らなくて済む。

ただし、fragment を使っている場合、親要素から class などの属性を追加する際に、子要素側で、明示的に`v-bind="$attrs"`を設定しないと属性が付与されないので注意。さらに scoped css にしている場合は`:deep()`が必要。

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

`<Suspense>`  は、コンポーネントツリーの非同期な依存関係を制御するための組み込みコンポーネント。コンポーネントツリーの下にある複数のネストされた非同期な依存関係が解決されるのを待つ間、ローディング状態をレンダリングすることができる。

非同期コンポーネントを使う際に有効だが、まだ Experimental な機能で API が変わる可能性があるので使用は控えたほうが良いかも。

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

## Reactive API

https://ja.vuejs.org/api/reactivity-utilities.html

リアクティブ周りの API が多数追加されている。リアクティブの制御で困ったら使ってみても良いかも。以下一部抜粋。

- [shallowRef()](https://ja.vuejs.org/api/reactivity-advanced.html#shallowref) `ref()`の浅いバージョン。
- [customRef()](https://ja.vuejs.org/api/reactivity-advanced.html#customref) 依存関係の追跡と更新のトリガーを明示的に制御して、カスタマイズされた ref を作成する
- [toRaw()](https://ja.vuejs.org/api/reactivity-advanced.html#toraw) Vue で作成されたプロキシの元のオブジェクトを返す
- [v-memo](https://ja.vuejs.org/api/built-in-directives.html#v-memoA) テンプレートのサブツリーのメモ化

# 🚨 注意すべき Breaking change

## v-model の仕様変更

https://v3-migration.vuejs.org/breaking-changes/v-model.html

Vue3 では v-model の仕様がいろいろ変更。

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

## Array, Object の変更で Vue.set, Vue.remove の利用が不要に

Vue2 では配列や、[配列、オブジェクトのキー指定の書き換え・削除がリアクティブにならない](https://jp.vuejs.org/v2/guide/reactivity.html#%E3%82%AA%E3%83%96%E3%82%B8%E3%82%A7%E3%82%AF%E3%83%88%E3%81%AB%E9%96%A2%E3%81%97%E3%81%A6)という注意点があったが、Vue3 では解消された。`Vue.set`や`Vue.remove` が不要になった。

### Vue2

@[codesandbox](https://codesandbox.io/embed/vue2-reactive-array-obj-q0tb5k?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

### Vue3

@[codesandbox](https://codesandbox.io/embed/sweet-snyder-rc60fr?fontsize=14&hidenavigation=1&module=%2Fsrc%2FApp.vue&theme=dark)

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

## deep セレクターの書き方が変更

https://ja.vuejs.org/api/sfc-css-features.html#scoped-css

`scoped` 環境で子コンポーネントの要素に対してスタイルを当てたい場合のセレクタの指定方法が変更されました。`:deep()` で指定してください 。

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

他にもいろいろある。詳細は[マイグレーションガイド](https://v3-migration.vuejs.org)を

- [トランジション機能のクラス名変更](https://v3-migration.vuejs.org/breaking-changes/transition.html)
  - 古い書き方だと警告もでないので注意!!
- [v-bind="obj"は書く位置で動作が変わるように](https://v3-migration.vuejs.org/breaking-changes/v-bind.html)
  - 以前は、他の属性と被っていると、それで上書きされていた。Vue3 からは、後に書いた属性が優先されるようになった
- [vm.$listeners は削除](https://v3-migration.vuejs.org/breaking-changes/listeners-removed.html)
  - `$attrs`に統合された。つまり`v-bind="$attrs"`でリスナーも登録される
- [vm.$on、vm.$off、vm.$once の削除](https://v3-migration.vuejs.org/breaking-changes/events-api.html#migration-strategy)
  - 同様のことをするには [mitt](https://github.com/developit/mitt) や [tiny-emitter](https://github.com/scottcorgan/tiny-emitter)などのライブラリが必要
- [v-on.native 修飾子の削除](https://v3-migration.vuejs.org/breaking-changes/v-on-native-modifier-removed.html)
  - `emits`に定義されていないイベントは全てネイティブイベントと見做されるようになった
- [ライフサイクルフックの命名変更](https://v3-migration.vuejs.org/breaking-changes/vnode-lifecycle-events.html#migration-strategy)
  - `destroyed`は`unmounted`に、`beforeDestroy`は`beforeUnmount`に命名変更

# 💬 Q & A

## script setup 使う？

コンポーネントの記述方法が代わり、覚えることが増える & Grep で探しにくくなるという懸念はあるが、それ以上に記述量が大幅に減ってコンポーネントの見通しが良くなるという利点のほうが大ききので積極的に使っていきたい。

## ref()と reactive()のどちらを使う？

基本的には `ref()` で良いと思う。`reactive()` だとリアクティブの消失に意識を向ける必要がある。もし、独自 store を作りたいケースなどで、reactive を使う場合は、`toRef` `toRefs` を使って export 時に Ref に変換するのを忘れずに。

参考: [vue composition api における props のリアクティブ性について知っておくべきこと](https://zenn.dev/karino_m/articles/vue-composition-api-and-props)

reactive が消失するパター ン

```js
const state = reactive({ name: "foo" });

// 1. 分割代入を行うパターン
const { name } = state;

// 2. プロパティを直接参照して代入するパターン
const name = state.name;

// 3. プロパティを直接参照して他の処理に渡すパターン
const { length } = useLength(state.name);
```

reactive の消失を防ぐには

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
- [Others/しんノート/出る順・Vue3 仕様変更の要点 - lapras.esa.io](https://lapras.esa.io/posts/10806)
