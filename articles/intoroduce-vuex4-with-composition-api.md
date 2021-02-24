---
title: "Vuex4 を Composition API + TypeScript で入門する"
emoji: "️️🔋"
type: "tech"
topics: ["vuex", "vue", "typescript"]
published: true
---

今月初めにリリースされた Vuex4 を Composition API + TypeScript で試してみたのでそのメモです。
この記事は以下バージョンにて検証しています。
- [vuejs/vue](https://github.com/vuejs/vue) 3.0.5
- [vuejs/vuex](https://github.com/vuejs/vuex) 4.0.0

# Vuexとは？
Vuex は、Vue.js 公式の状態管理ライブラリです。
Vue アプリケーション内に、どの階層のコンポーネントからでもデータを取得・更新できる単一のデータストアを作ることができます。Vuex を使うことで複数のコンポーネントで使う共有データの Props/Emit による過剰なバケツリレーが不要になります。

また、複雑になりがちな状態管理において以下の図のように特定のルールで制約を与えることでデータの流れを一元化して、コードの構造と保守性を向上させることができます。

![](https://i.gyazo.com/57159fbe2e8c50e475808076c325832f.png)
（[What is Vuex?](https://next.vuex.vuejs.org/)）

# 使い方
Vue CLI で Vue3 + TypeScript の環境構築が行われている前提で、簡単な Todo アプリの状態管理を例に Vuex の一連の流れをみていきます。

## インストール + 初期設定

まず Vuex を依存に追加します。

:::message
Vuex4 は 2021/02/24 時点で vuex ではなく vuex@next というパッケージ指定なので注意してください。
:::

```
yarn add vuex@next --save
```

つぎに Vuex のストアの構造を定義する`store.ts`を作成します。

```
mkdir src/store
touch store/store.ts
```

```ts:src/store/store.ts
import { InjectionKey } from 'vue';
import { createStore, Store, useStore as baseUseStore } from "vuex";

// stateの型定義
type State = {};

// storeをprovide/injectするためのキー
export const key: InjectionKey<Store<State>> = Symbol();

// store本体
export const store = createStore<State>({});

// useStoreを使う時にキーの指定を省略するためのラッパー関数
export const useStore = () => {
  return baseUseStore(key);
}
```

`useStore`のラッパー関数はなくても良いのですが、毎回コンポーネントでストアを取得するために`useStore(key)`と key を指定するのが面倒なので定義しています。コンポーネントからは`store.ts`の`userStore()`を使うようにします（[参考](https://next.vuex.vuejs.org/guide/typescript-support.html#simplifying-usestore-usage))。

そしてストアを`main.ts`でプラグインとして設定します。

```ts:main.ts
import { createApp } from "vue";
import App from "./App.vue";
import { key, store } from "./store/store";

const app = createApp(App);

app.use(store, key);

app.mount("#app");
```

これで準備は完了です。

## State
state は Vuex で管理する状態そのものです。state はリアクティブなデータとなり、state の変更は Vue.js の変更検知の対象となります。

### ストアでの定義

まず `store.ts`で state の型定義を追加します。

```ts:store.ts
// ...
type TodoItem = {
  id: number;
  title: string;
  content: string;
  completed: boolean;
};

type State = {
  todoItems: TodoItem[];
};
// ...
```

そしてストアの実装である`createStore`の引数オブジェクトに`state`プロパティを追加して初期値を設定します。

```ts:store.ts
// ...
export const store = createStore<State>({
   state: {
     todoItems: [
      {
         id: 1,
        title: "foo",
        content: "bar",
        completed: false
      }
    ]
  }
});
// ...
```

### コンポーネントでの利用

state の利用は`store.ts`に定義した`useStore`を使います。ラッパー関数にて`provide/inject`のキーは設定しているので、この場でキーの指定は不要です。

```vue:src/store/store.ts
<template>
  <div>
    <div v-for="item in todoItems" :key="item.id">
      <p>{{ item.title }}</p>
      <p>{{ item.content }}</p>
      <p>{{ item.completed ? "✅" : "-" }}</p>
      <hr />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, computed } from "@vue/runtime-core";
import { useStore } from "@/store/store"; // store/store.tsのものを利用

export default defineComponent({
  setup() {
    const store = useStore();
    const todoItems = computed(() => store.state.todoItems);

    return {
      todoItems
    };
  }
});
</script>
```

`store.sate`で取得できる値は`store.ts`で型定義した`type State`の型が設定されています。 なので state は型安全に使えます。

```ts
// 例
// idはnumberと推論される
const id = store.state.todoItems[0].id
```

## Mutations
mutations は state を変更する関数です。
Vuex で state を変更する際は直接 state を書き換えるのではなく、必ず mutation を経由して行うのが Vuex のルールです。

### ストアでの定義

まず、mutation の関数名を管理する`mutationType.ts`を作成します。
関数名を定数や Enum で一元管理しておくと、後述する commit の際に定数で指定できるので便利です。

```ts:src/store/mutationType.ts
export const ADD_TODO_ITEM = "ADD_TODO_ITEM";
```

次に`store.ts`の createStore の引数のオブジェクトに mutations をプロパティを追加します。
mutation の関数の第 1 引数には state、第 2 引数には後述する commit の際の引数を受け取れます。その引数を使って関数の内部で state を更新します。


```ts:src/store/store.ts
// ...
import * as MutationTypes from "./mutationTypes";

// ...
export const store = createStore<State>({
  // ...
  mutations: {
    [MutationTypes.ADD_TODO_ITEM](state, todoItem: TodoItem) {
      state.todoItems.push(todoItem);
    }
  }
});
```

### コンポーネントでの利用
コンポーネント側での mutations の実効は`store.commit`を使います。
以下 TodoItem を追加する例です。

```vue:src/components/TodoItemForm.vue
<template>
  <form>
    <label for="title">
      title
      <input type="text" id="title" v-model="form.title" />
    </label>
    <label for="content">
      content
      <input type="text" id="content" v-model="form.content" />
    </label>
    <input type="submit" value="submit" @click.prevent="onSubmit" />
  </form>
</template>

<script lang="ts">
import { defineComponent, reactive } from "@vue/runtime-core";
import { useStore } from "@/store/store";
import * as MutationTypes from "@/store/mutationTypes";

export default defineComponent({
  setup() {
    const form = reactive({
      title: "",
      content: ""
    });

    const clearForm = () => {
      form.title = "";
      form.content = "";
    };

    const store = useStore();

    const onSubmit = () => {
      store.commit(MutationTypes.ADD_TODO_ITEM, {
        id: Math.floor(Math.random() * 100000), // 仮でランダムなIDを設定
        content: form.content,
        title: form.title
      });
      clearForm();
    };

    return {
      onSubmit,
      form
    };
  }
});
</script>
```

state の変更は、`onSubmit`の`store.commit`部分で行っています。
第 1 引数に mutation の関数名の文字列（ここでは mutationTypes の定数を利用）を指定して、第 2 引数で対象の mutation に渡す引数を指定しています。

```ts
const onSubmit = async () => {
  await store.commit(MutationTypes.ADD_TODO_ITEM, {
    id: Math.floor(Math.random() * 100000), // 仮でランダムなIDを設定
    content: form.content,
    title: form.title
  });
  clearForm();
};
```

注意する点として、mutation は同期的に実効されます。非同期処理を mutation の関数内で行はないようにしてください。
※ mutation 内部で非同期処理を書くことは出来ますが、store.commit が Promise を返さないので非同期処理を待つことが出来ない。

:::message
コンポーネントから mutation を実効せず、必ず後述の actions を介して mutation を実効という思想もあります。記述は増えますがそちらのほうがストアのデータの流れはシンプルになるので良さそうな気がします。
:::

## Actions
action は同期・非同期を問わない state に関するあらゆる操作を定義するものです。
一般的には非同期処理を含む state の変更のトリガーに用いられます。

### ストアでの定義
TodoItems の初期値を API 経由で取得する例です。
action も mutation と同様に定数で管理するためにまず`actionTypes.ts`を追加します。

```ts:src/store/actinonTypes.ts
export const INITIALIZE_TODO_ITEMS = "INITIALIZE_TODO_ITEMS";
```

store の変更は mutation 経由して行うので`mutationTypes.ts`にも定数を追加します。

```ts:src/store/mutationTypes.ts
// ...
export const INITIALIZE_TODO_ITEMS = "INITIALIZE_TODO_ITEMS";
```

次に`store.ts`の createStore の引数のオブジェクトに actions プロパティを追加します。
actions の関数の第 1 引数には`state`, `commit`, `getters`を含むオブジェクトが受け取れます。actions の関数で async/await を使うことで非同期処理をハンドルできます。

```ts:src/store/store.ts
// ...
import * as ActionTypes from "./actionTypes";

// ...
export const store = createStore<State>({
  // ...
  mutations: {
    // ...
    [MutationTypes.INITIALIZE_TODO_ITEMS](store, todoItems: TodoItem[]) {
      store.todoItems = todoItems;
    }
  },
  actions: {
    async [ActionTypes.INITIALIZE_TODO_ITEMS]({ commit }) {
      const todoItems = await fetchAllTodoItems(); // TodoItemsを取得するAPIコール
      commit(ActionTypes.INITIALIZE_TODO_ITEMS, todoItems);
    }
  }
});
```

今回は todoItem の一覧を取得する`fetchAllTodoItems`を実効してその戻値を mutation を介して TodoItems の初期化に使っています。


### コンポーネントでの利用

コンポーネントから action を利用する場合は`store.dispatch`を使います。
以下はコンポーネントがマウントされるタイミングで dispatch を呼ぶ例です。dispatch は Promise を返すので async/await で処理を待つことができます。

```vue:src/components/TodoItems.vue
<!-- ... --> 

<script lang="ts">
import { defineComponent, computed } from "@vue/runtime-core";
import { useStore } from "@/store/store"; // store/store.tsのものを利用
import * as ActionTypes from "@/store/actionTypes"

export default defineComponent({
  async setup() {
    const store = useStore();
    const todoItems = computed(() => store.state.todoItems);

    onMounted(async () => {
      await store.dispatch(ActionTypes.INITIALIZE_TODO_ITEMS)
    })

    return {
      todoItems
    };
  }
});
</script>
```

ここでは dispatch に引数を渡していませんが、引数を受け取る Actions の場合は第 2 引数で Actions に渡す引数を指定出来ます。

## Getters
getter は state からの算出データを定義するものです。state に何らかの処理をした値を複数の場所で利用する場合は、 getter を定義しておくと便利です。


### ストアでの定義
`store.ts`の createStore の引数のオブジェクトに getters プロパティを追加して、完了済みの TodoItems を取得する getter を定義します。

```ts:src/store/store
export const store = createStore<State>({
  // ...
  getters: {
    completedTodoItems: store => {
      return store.todoItems.filter(todo => todo.completed);
    }
  }
}
```

:::message
Vue 3.0 の場合は Vue2 系と違い getter の結果がキャッシュされないという不具合があるようです。これは[こちらのPR](https://github.com/vuejs/vuex/pull/1883)で対応されいます。
Vue 3.1 で解消される見込みとのことです。
:::

### コンポーネントでの利用

コンポーネントでは`store.getters`で利用出来ます。
ただ、state と違い`store.getters`の戻値は any なので型推論が効きません。

```vue:src/components/completedTodoItems.vue
<!-- ... -->
<script lang="ts">
import { defineComponent, computed } from "@vue/runtime-core";
import { useStore } from "@/store/store"; // store/store.tsのものを利用

export default defineComponent({
  setup() {
    const store = useStore();
    const todoItems = computed(() => store.getters.completedTodoItems); // todoItemsはComputedRef<any>と推論される

    return {
      todoItems
    };
  }
});
</script>
```

# その他考慮点
使ってみてわかった Vuex4 採用に関する考慮点を書きます。

## TypeScriptの対応がまだ不完全

Vuex の課題として長らく上がっているのが TypeScript への対応だと思います。
commit、dispatch、getters をタイプセーフに実効することが出来ず、型補完を効かせるには [vuex-type-helper](https://github.com/ktsn/vuex-type-helper) 等のプラグインを別途使う必要がありました。

TypeScript 4.1 で template literal types が入ったことですし Vuex4 で何らかの改善があるのかなと期待してたところですが、まだ完全な対応は難しいようです。
Vuex4 でも Vuex3 と同様、commit、dispatch、getters で型補完は効きません。別 Plugin を利用するか、独自で型定義を設定して何らかの対応をする必要があるようです。

（独自で型定義する例）
https://dev.to/shubhadip/vue-3-vuex-4-modules-typescript-2i2o

## mapHelpersがComposition APIでは使えない
Vue4 + Composition API の場合の現状の問題点が、複数のストア操作を一括で定義できる mapHelpers（`mapState`、 `mapGetters`、`mapActions`、`mapMutations`）が使えないことです。
※ Vuex4 でも Option API を使う場合は mapHelpers を使えます。

Vuex を使っている場合はほぼ mapHelpers を使っていると思うので、Vuex4 + Composition API の書き換えの障害になると思われます。

一応以下の Issue が上がっています。早く実装されると良いですね。

https://github.com/vuejs/vuex/issues/1725

# 終わりに

以上、Vuex4 + Composition API + TypeScript の簡単な紹介でした。
TypeScript の対応や mapHelpers など今すぐの採用は迷うところですが、引き続き動向を追っていきたいです。
その他 Vuex の Moudule 機能が中々ややこしいのでまた別でまとめようと思っています。

# 参考

- [Vuex公式ドキュメント](https://next.vuex.vuejs.org/)
- [みんなのVue.js：書籍案内｜技術評論社](https://gihyo.jp/book/2021/978-4-297-11902-7)