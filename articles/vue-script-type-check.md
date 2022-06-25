---
title: "Vue SFC の script 部分のみを型チェックするCLIツールを作ってみた"
emoji: "🥗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "typescript", "ts-morph", "cli", "yargs"]
published: false
---

最近作ってみた CLI ツールの紹介です。

# 作ったもの

[vue-script-type-check](https://github.com/kawamataryo/vue-script-type-check) という Vue SFC の`<script>`部分のみを型チェックする CLI ツールを作りました。
以下 GIF のように、Vue SFC のパスを指定することで、型チェックを実行できます。
主に、CI での型チェックの際に tsc と併用して利用されることを想定しています。

![](https://i.gyazo.com/54ca662f85b6909bfd510da200968f53.gif)


## 使い方

以下コマンドで実行できます。
`./src/**/*.vue` の部分は任意のファイルを個別で指定することも可能です。

```bash
$ npx vue-script-type-check ./src/**/*.vue
```

option はこちらです。CI 用の tsconfig.json を指定したい時などは、こちらを利用ください。

| option              | default           | description                  |
| ------------------- | ----------------- | ---------------------------- |
| -t, --tsconfig-path | `./tsconfig.json` | tsconfig.json へのパスの指定 |

## モチベーション

なぜこのようなツールを作ったかというと、Vue SFC 内のTypeScriptの型チェックが実施されていないプロジェクトにて、漸進的に型チェックを強化していきたいと考えたからです。
本来であれば文末の類似ツールで紹介している [vue-tsc](https://github.com/johnsoncodehk/volar) を使って型チェックを実施すればいいのですが、現状のプロジェクトで`<template>` 部分も含めて型チェクすると、型エラーの数が膨大になりすぎて、対処が現実的ではありませんでした。まず、型改善の効果が出やすく、対処のしやすい`<script>`部分に焦点をあてて、改善していくためにこちらを作りました。

以前の以下の記事で紹介した[suppress-ts-errors](https://github.com/kawamataryo/suppress-ts-errors)と組み合わせ、一度既存のすべての型エラーを `@ts-expect-error` で抑制した上で、このツールの型チェックをCIで実行すれば、新しいファイルには型エラーが混入することを防ぐことができます。あとは、既存のファイルにある`@ts-expect-error`を順次潰していけば、漸進的に型チェックの強化が行えます。

https://github.com/kawamataryo/suppress-ts-errors

:::message
webpackのts-loaderで型チェックすれば良いのでは？という声もあると思うのですが、ビルドツールはあくまでビルドのみのシンプルな責務にとどめたい、ビルド速度の低下を防ぎたいという思いから今回は採用を見送りました。
:::


# 実装のポイント

個人的な実装時の工夫ポイントを紹介します

## Vue のスクリプト部分のみ型チェック

基本的には[suppress-ts-errorsの記事](https://zenn.dev/ryo_kawamata/articles/suppress-ts-errors)で紹介した実装と同じく以下流れで処理を行っています。

```
1. 検査対象の Vue ファイル一覧を取得
2. 個々の Vue ファイルから`<script lang=“ts”>`の script 部分を抽出
3. script 部分を TypeScript Compiler API の Project にセット
4. 型チェックを行い結果を出力
```

スクリプト部分の抽出から型エラーの出力までのコードはこちらです。
コマンドライン引数で受けったVue SFCのファイルパスをもとに、Vue SFCを走査して、`script` の抽出、型エラーのチェックを行っています。

https://github.com/kawamataryo/vue-script-type-check/blob/main/src/handler.ts#L9-L73

## エラー発生行の表示

小さなこだわりですが、標準出力をカラフルにしたほうが楽しいかなと思い、[chalk]()

## 視認性を意識したカラフルな出力結果

# 類似ツール

## vue-tsc

Vite の Vue プロジェクトで公式に採用されている型チェッカーです。VS Code の Vue プラグインの Volar の language-server を内部的に使っているようで、`<template>`部分含めて型チェックしてくれます。

今回は `<template>` 部分の型チェックは無視したかったので、選択肢には上がりませんでしたが、`<script>` 部分の型エラーが解消されたら、こちらの型チェックも通るように修正していきたいと思っています。

https://www.npmjs.com/package/vue-tsc

## vue-script-tsc

まさに今回の用途と同様に Vue SFC の`<script>`部分のみを型チェックする CLI ツールです。このツールの存在に気づいたのは、vue-script-type-check をほぼほぼ実装したあとでした。。
ただ、このツール自体に `lang="ts"` と TypeScript の指定がない SFC も型チェックの対象としてしまうバグや、出力が真っ赤で視認性があまりよくない（ただの言いがかり 😇）があったので、今回自分で作ったツールのほうが使い勝手は良いかなと個人的に思っています。

https://github.com/policyfly/vue-script-tsc

# おわりに

以上、[vue-script-type-check](https://github.com/kawamataryo/vue-script-type-check) の紹介でした！
このツールを使って徐々により良いプロジェクトを作っていきたいと思います。