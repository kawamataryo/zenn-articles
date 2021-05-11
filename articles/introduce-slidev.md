---
title: "開発者のための拡張性溢れるスライド作成ツール Slidev を使ってみた"
emoji: "📘"
type: "tech"
topics: ['slidev', "typescript", "mock", "jest"]
published: false
---

マークダウン形式のスライド作成ツール Slidev を試したらとても良かったので紹介です。


# Slidevとは？
Slidev は[Vue Use](https://github.com/vueuse/vueuse)や[Type Challenges](https://github.com/type-challenges/type-challenges)の作者であり、Vue.js のコアチームメンバーでもある[Anthony Fu](https://github.com/antfu)が開発しているマークダウン形式でのスライド作成ツールです。
まだ Public Beta ですが、すでに完成度がとても高く、**Public Beta のリリースからわずか 4 日で 2,700 を超える GitHub スター**を集めています。

https://github.com/slidevjs/slidev

実際にどのようなプレゼンが可能なのかは、Slidev のデモや、作者の Anthony が実際に Slidev でプレゼンした [VueDay2021](https://2021.vueday.it) の資料が分かりやすいです。

- デモ動画

https://www.youtube.com/watch?v=eW7v-2ZKZOU


- VueDay2021 でのプレゼン

https://www.youtube.com/watch?v=IMJjP6edHd0

# どんな機能がある？
Slidev の特徴的な機能を紹介します。

## マークダウン記法によるスライド作成

[reveal.js](https://revealjs.com/)や[remark](https://github.com/remarkjs/remark)などのマークダウン形式でのスライド作成ツールとほぼ同様の記法で、スライドを作成できます。大抵のツールでは、ある程度決まったレイアウトになるのですが、Slidev の場合は拡張性がかなり高く、ページごとに独自のスタイルを当てて自由なレイアウトを構築することも出来きます。

```md
# Slidevの紹介

Hello, World!

---

# ページ2

コードスニペット

```ts
console.log('Hello, World!')
```


---

# ページ3

ツイート の埋め込み

<div class="p-3">
  <Tweet :id="20" />
<div>
```

## コードスニペットの強力なサポート
開発者が PowerPoint や Keynote でスライドを作るときに大抵苦労するのがコードの埋込みです。シンタックスハイライトを効かせることだったり、Font の幅の調整、可視性を上げるためのアニメーションなど地味に苦労します。Slidev はその部分をとても強力にサポートしています。
いつものマークダウン記法を使えば、シンタックスハイライトが効くことはもちろん、可視性を上げるために行ごとのハイライトも可能です。

```md

```ts {all|2-3|5|all}
function add(
  a: Ref<number> | number,
  b: Ref<number> | number
) {
  return computed(() => unref(a) + unref(b))
}
```

```

ほかにもブラウザベースのコードエディタである[monaco-editor](https://github.com/Microsoft/monaco-editor)をサポートしているため、スライド画面を移しながらライブコーディング的にコードを編集することも可能です。TypeScriptの型補完なども表示されるので、コードの説明がとてもしやすくなりそうです。

```
```js {monaco}
const count = ref(1)
const plusOne = computed(() => count.value + 1)

console.log(plusOne.value) // 2

plusOne.value++ // error
```
```

## プレゼンテーションの録画機能

## 豊富な出力形式


## 自由な拡張


# 使い方

任意のディレクトリで以下コマンドを実行するだけで環境は整います。

```bash
$ npm init slidev
```

コマンドを実行するとディレクトリ名の入力や、パッケージマネージャーの選択肢が表示されます。それらを入力し、しばらく待つとブラウザが開き、サンプルのスライドが表示されます。
あとは作成されたディレクトリに移動し`slidev.md`を編集すれば OK です。

# 終わりに
クライアントのノート画面や、
次回のプレゼン作成では、是非使ってみたいと思いました。
