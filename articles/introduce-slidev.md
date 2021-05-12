---
title: "開発者のための拡張性溢れるスライド作成ツール Slidev"
emoji: "👨‍💻"
type: "tech"
topics: ['slidev', "typescript", "mock", "jest"]
published: true
---

マークダウン形式のスライド作成ツール Slidev を試したら、とてもとても良かったので紹介です。

# Slidevとは？
Slidev は[Vue Use](https://github.com/vueuse/vueuse)や[Type Challenges](https://github.com/type-challenges/type-challenges)の作者であり、Vue.js のコアチームメンバーでもある[Anthony Fu](https://github.com/antfu)が開発しているマークダウン形式でのスライド作成ツールです。
[Vite](https://github.com/vitejs/vite)、[Vue3](https://github.com/vuejs/vue-next)、[WindiCSS](https://github.com/windicss/windicss)を用いて開発されています。

まだ Public Beta ですが既に完成度がかなり高く、**リリースからわずか数日で 7,000 を超える GitHub スター**を集めています。

https://github.com/slidevjs/slidev

実際にどのようなプレゼンが可能なのかは、Slidev のデモや、作者の Anthony が Slidev を用いてプレゼンした [VueDay2021](https://2021.vueday.it) の資料が分かりやすいです。

- デモ動画

https://www.youtube.com/watch?v=eW7v-2ZKZOU


- VueDay2021 でのプレゼン

https://www.youtube.com/watch?v=IMJjP6edHd0

# 使い方

任意のディレクトリで以下コマンドを実行するだけで環境は整います。

```bash
$ npm init slidev
```

コマンドを実行するとディレクトリ名の入力や、パッケージマネージャーの選択肢が表示されます。それらを入力した後しばらく待つとブラウザが開き、サンプルのスライドが表示されます。
あとは作成されたディレクトリに移動し`slidev.md`を編集すれば OK です。

# どんな機能があるの？
Slidev の特徴的な機能を紹介します。

## 柔軟なマークダウン記法によるスライド作成

[reveal.js](https://revealjs.com/)や[remark](https://github.com/remarkjs/remark)などのマークダウン形式でのスライド作成ツールとほぼ同様の記法で、スライドを作成できます。大抵のツールでは、ある程度決まったレイアウトになるのですが、Slidev の場合は拡張性がかなり高いです。Vue Component や、[WindiCSS](https://github.com/windicss/windicss) でスタイリングすることでページごとに独自のスタイルを当てて自由なレイアウトで構築することも出来きます。

- [WindiCSS](https://github.com/windicss/windicss) によるスタイリング（もちろん通常の CSS も書けます）
- Twitter・YouTube の埋込み
- [vite-plugin-icon](https://github.com/antfu/vite-plugin-icons)、[Icontify](https://github.com/iconify/iconify) によるアイコンサポート
- LaTeX 記法のサポート
- Vue、CSS トランジションによるアニメーション

https://sli.dev/guide/syntax.html


## 視認性の高いコードスニペット & ライブコーディング
開発者が PowerPoint や Keynote でスライドを作るときに大抵苦労するのがコードの埋込みです。シンタックスハイライトを効かせることだったり、Font の幅の調整、可視性を上げるためのアニメーションなど地味に苦労します。Slidev はその部分をとても強力にサポートしています。
いつものマークダウン記法を使えば、**シンタックスハイライトが効くことはもちろん、可視性を上げるために行ごとのハイライトも可能**です。

さらにブラウザベースのコードエディタである[monaco-editor](https://github.com/Microsoft/monaco-editor)をサポートしているため、**スライド画面を移しながらライブコーディング的にコードを編集することも可能**です。TypeScript の型補完なども表示されるので、コードの説明がとてもやりやすそうです。


![](https://i.gyazo.com/f56ea3abc1402e74806e3480970f5127.gif =650x)


https://sli.dev/guide/syntax.html#line-highlighting

## 多彩な表示モード
ただ画面にスライドを移すだけでなく、Keynote や PowerPoint のようにプレゼンテーションのツールバーに様々な表示モードがあります。
特に、インカメラの表示はリモート環境での LT を考えるとかなり良いですね。

- ダークモードの切り替え
- スライド一覧表示
- プレゼンターモード
- インカメラの表示
- メモ機能

![](https://i.gyazo.com/7702cd5573225807ae7250868c879813.gif)


https://sli.dev/guide/presenter-mode.html

## テーマ機能のサポート

Slidev ではテーマ機能をサポートしているため、デザインの統一されたスライドを簡単に作成できます。まだテーマの数は少ないですが、ユーザー側での作成も容易にできそうなので今後どんどん増えていきそうですね。

![](https://i.gyazo.com/ef8ea0d97dbdab9155bf75971ae9932e.png)


https://sli.dev/themes/gallery.html
## プレゼンテーションの録画機能

事前録画の発表の際にとても便利だと思うのですが、プレゼンテーションの録画機能も Slidev 自体でサポートしています。
ただただ凄い。.!

![](https://i.gyazo.com/0e90c81a250bf673973b1c8f6515d827.png)

https://sli.dev/guide/recording.html

## 豊富な出力形式

Slidev は PDF、PNGs、SPA と多数の出力形式を持っています。
SPA として GitHub Pages などに公開も出来るので SNS での共有もとても楽そうです。

https://sli.dev/guide/exporting.html

# おわりに
以上、とても簡単ですが Slidev の紹介でした。
正直プレゼン資料作成はいつも苦痛なのですが、そんな私でも次のプレゼンが楽しみになるくらいワクワクするプロダクトです。
しかも、この豊富な機能でまだ Beta 版です。今後の進化も楽しみですね。