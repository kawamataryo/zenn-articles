---
title: "Vue SFC の script 部分のみを型チェックするCLIツールを作ってみた"
emoji: "🪴"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "typescript", "ts-morph", "cli", "yargs"]
published: false
---

最近作ってみた CLI ツールの紹介です。

# 作ったもの

[vue-script-type-check](https://github.com/kawamataryo/vue-script-type-check) という Vue SFC の`<script>` 部分のみを型チェックする CLI ツールを作りました。
以下 GIF のように、Vue SFC のパスを指定することで、型チェックを実行できます。
主に、CI での型チェックの際に tsc と併用して利用されることを想定しています。

![](https://i.gyazo.com/54ca662f85b6909bfd510da200968f53.gif)

コードはこちらで公開しています。

https://github.com/kawamataryo/vue-script-type-check

## 使い方

以下コマンドで実行できます。
`./src/**/*.vue` の部分は任意のファイルを個別で指定することも可能です。

```bash
$ npx vue-script-type-check ./src/**/*.vue
```

option はこちらです。CI 用の tsconfig.json を指定したい時などにご利用ください。

| option              | default           | description                  |
| ------------------- | ----------------- | ---------------------------- |
| -t, --tsconfig-path | `./tsconfig.json` | tsconfig.json へのパスの指定 |

## モチベーション

なぜこのようなツールを作ったかというと、Vue SFC 内の TypeScript の型チェックが実施されていないプロジェクトにて、漸進的に型チェックを適応していきたいと考えたからです。

本来であれば文末の類似ツールで紹介している [vue-tsc](https://github.com/johnsoncodehk/volar) を使って型チェックを行えばいいのですが、現状のプロジェクトで`<template>` 部分も含めて型チェクすると、型エラーの数が膨大になりすぎて、対処が現実的ではありませんでした。まず、型改善の効果が出やすく、対処のしやすい`<script>`部分に焦点をあてて、改善していくきたいと思いました。

[こちらの記事](https://zenn.dev/ryo_kawamata/articles/suppress-ts-errors) で紹介した[suppress-ts-errors](https://github.com/kawamataryo/suppress-ts-errors)と組み合わせ、一度既存のすべての型エラーを `@ts-expect-error` で抑制した上で、このツールの型チェックを CI で実行すれば、新しいファイルには型エラーが混入することを防ぐことができます。あとは、既存のファイルにある`@ts-expect-error`を順次潰していけば、漸進的な型チェックの強化が行えます。

https://github.com/kawamataryo/suppress-ts-errors

:::message
webpack の ts-loader で型チェックすれば良いのでは？という意見もあると思うのですが、webpack はあくまでビルドのみのシンプルな責務にとどめたい & ビルド速度の低下を防ぎたいという思いから今回は別にツールを作りました。
:::

# 実装のポイント

実装時の工夫ポイントを紹介します。

## Vue のスクリプト部分のみ型チェック

内部的には[suppress-ts-errors の記事](https://zenn.dev/ryo_kawamata/articles/suppress-ts-errors)で紹介した実装とほぼ同じです。[ts-morph](https://github.com/dsherret/ts-morph) で TypeScript Compiler API をにて型エラーを抽出しています。

https://github.com/dsherret/ts-morph

または、Vue ファイルの処理は以下流れで行っています。

```
1. 検査対象の Vue SFC ファイル一覧を取得
2. 個々の Vue SFC ファイルから`<script lang=“ts”>`の script 部分を抽出
3. script 部分を TypeScript Compiler API の Project にセット
4. 型チェックを行い結果を出力
```

スクリプト部分の抽出から型エラーの出力までのコードはこちらです。
コマンドライン引数で受けった Vue SFC のファイルパスをもとに、ファイルを走査して、`script` の抽出、型エラーのチェックを行っています。

https://github.com/kawamataryo/vue-script-type-check/blob/main/src/handler.ts#L9-L73

## 視認性を意識したカラフルな出力結果

小さいことですが、標準出力はカラフルなほうが楽しいかなと思い、色付けしています。

![](https://i.gyazo.com/f8a21faebcb1f48dd060b10d4c90ac6e.png)

ライブラリは[chalk](https://github.com/chalk/chalk) を使っています。出力したい文字列を色別のメソッドに渡すだけで OK です。

https://github.com/chalk/chalk

出力するエラーコードの整形はこちらで行っています。

https://github.com/kawamataryo/vue-script-type-check/blob/main/src/lib/collectTsErrors.ts#L13-L30

# 類似ツール

類似ツールの紹介も簡単に。

## vue-tsc

Vite の Vue プロジェクトで公式に採用されている型チェッカーです。VS Code の Vue プラグインの Volar の language-server を内部的に使っているようで、`<template>`部分含めて型チェックしてくれます。

今回は `<template>` 部分の型チェックは無視したかったので、選択肢には上がりませんでしたが、`<script>` 部分の型エラーが解消されたら、こちらの型チェックも通るように修正していきたいと思っています。

https://www.npmjs.com/package/vue-tsc

## vue-script-tsc

まさに今回の用途と同様に Vue SFC の`<script>`部分のみを型チェックする CLI ツールです。このツールの存在に気づいたのは、vue-script-type-check をほぼほぼ実装したあとでした。。
ただ、このツール自体に `lang="ts"` と TypeScript の指定がない SFC も型チェックの対象としてしまうバグや、出力が真っ赤で視認性が悪い（ただの言いがかり 😇）というのがあったので、今回自分で作ったツールのほうが使い勝手は良いかなと個人的に思っています。

https://github.com/policyfly/vue-script-tsc

# おわりに

以上、[vue-script-type-check](https://github.com/kawamataryo/vue-script-type-check) の紹介でした！
このツールを使ってVue SFC の型チェックの強化頑張っていくぞ。

※ 大分勢いで作ったので、もしかしたらバグあるかもです。使ってみて変なところがあったら[Issue](https://github.com/kawamataryo/vue-script-type-check/issues)を挙げてもらえると助かります🙏