---
title: "Deno で簡易レンダリングエンジンを作ってみた"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "TypeScript", "browser"]
published: false
---

# 作ったもの

Web ブラウザの仕組みを基礎から勉強したいと思い、「[Let's build a browser engine!](https://limpet.net/mbrubeck/2014/08/08/toy-layout-engine-1.html)」記事を参考に Deno で簡易的な HTML レンダリングエンジンを作ってみました。

簡易的という言葉の通り、実用性はないです。以下 GIF のように、HTML と CSS を入力として受け取り、Canvas にボックスを描画するだけです。


![](https://i.gyazo.com/0ff561a6683205a130cf4f444cb8f19f.gif)

https://deno-toy-rendering-engine.deno.dev/

描画に対応しているものは、ブロック要素のレイアウトのみで、使える CSS もごくわずか。サイズ・位置指定（width、 height、 padding、margin、border-width）と装飾（background-color、border-color）のみ。

テキストの描画もできません。

ただ、ひとつひとつの過程を自分で実装していくので、レンダリングエンジンの仕組みを勉強するにはとても良いものでした。

![](https://i.gyazo.com/2d8cbf42388d833d54f10f7c17eb77d3.png)
*本記事で実装するレンダリングエンジンの流れ*

参考記事では、Rust で作られていたのですが、~~Rust のコンパイラが怖いので~~ TypeScript で実装してみました。また、ランタイムは Node.js ではなく [Deno](https://deno.land/) を選択しています。
実装コードはすべてこちらのリポジトリで公開しています。

https://github.com/kawamataryo/deno-toy-browser-engine

:::message
「これでレンダリングエンジンと言えるのか!!😡 」というマサカリが怖いですが、あくまで簡易的ですし、参考記事もレンダリングエンジンと言っているのでお許しを。。ちなみに、参考記事の筆者 [Matt Brubeck](https://github.com/mbrubeck) さんはもともと Mozilla で Firefox の開発をしていた方のようです。

https://hacks.mozilla.org/author/mbrubeckmozilla-com/
:::

# 章ごとの所感

## Part 1: Getting started, Part 2: HTML

![](https://i.gyazo.com/43006f9b63e958494409e643d00cee4d.png)
<https://limpet.net/mbrubeck/2014/08/08/toy-layout-engine-1.html>
<https://limpet.net/mbrubeck/2014/08/11/toy-layout-engine-2.html>

この記事で作るレンダリングエンジンの全体の流れを説明した上で、HTMLのパーサーを作ります。そして、そのパーサーを使って、HTMLの文字列を受け取り、DOMノードツリーを構築します。

この章で作るHTMLパーサーが対応しているHTML構文は以下のみです。

* `<p>``</p>`などのオープンとクローズが対になったタグ
* タグに付与するidとclassの属性
* `hello world`などのテキストノード

それ以外の`<!-- -->`でのコメントや`<br />`の自己完結タグなどはすべて未対応です。

`consumeChar`等のメソッドをつくり、再帰やループを駆使して1文字ずつ解析していく過程で、パーサーの実装が徐々に理解できました。閉じタグ忘れががどのように評価されるのか、身を持って体験することもできます。

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/html_parser.ts#L1-L141

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/html_parser.test.ts#L1-L116


## Part 3: CSS

![](https://i.gyazo.com/20fb75e12e49e1fd730d052632261f49.png)

<https://limpet.net/mbrubeck/2014/08/13/toy-layout-engine-3-css.html>

次にCSSもHTMLと同じようにパーサーを作りパースします。CSSの場合は階層構造とはなっていないので、パース結果は、ツリーではなくオブジェクトの配列になります。

ここで作るパーサーでも、すべてのプロパティと値をサポートすると莫大なコード量が必要なので、最低限のものしかサポートしていません。

プロパティは

* ボックスのサイズ・レイアウト指定（width、 height、 padding、margin、border-width）
* ボックスのカラーリング（background-color, border-color)

値は

* サイズは数値 + `px`
* カラーはHEX（`#00ff00`など）

のみです。


:::message
記事中で、HTMLパーサーの説明は懇切丁寧に書かれていたのですが、CSSパーサーの実装説明は最低限必要なところのみでした。ちょっと不親切かもと思いつつ、ちょうどHTMLパーサーでパーサーの基本は理解したあとなので、サンプルコードを見ずともある程度実装できました。
:::

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/css_parser.ts#L1-L189

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/css_parser.test.ts


## Part 4: Style

![](https://i.gyazo.com/f6ce1ea7f32c618575d1440d4a3f1fcb.png)

<https://limpet.net/mbrubeck/2014/08/23/toy-layout-engine-4-style.html>

DOMノードツリーとCSSのパース結果を入力として受けとり、各ノードにCSSのスタイル定義が割り当てられたStyleツリーというものを作ります。

DOMノードツリーに新しいフィールドを追加することでも同じような出力結果を得ることはできますが、Styleの指定によっては、後続の処理に渡さない（たとえば`display: none`など）こともあるので、新しいノードを作るほうが効率的によいとのことです。これは今後の処理でも同じで、都度新しい構造体を作っていきます。

`id`、`class`、`tag` などセレクターでの詳細度を考慮して、ノードのスタイルを決める（同名のプロパティの場合は、詳細度の高いものを優先する）処理など勉強になりました。


**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/styled_node.ts#L1-L130

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/styled_node.test.ts#L1-L197


## Part 5: Boxes, Part 6: Block layout

![](https://i.gyazo.com/494404977dad6e6657d894bdc6593510.png)

<https://limpet.net/mbrubeck/2014/09/08/toy-layout-engine-5-boxes.html>
<https://limpet.net/mbrubeck/2014/09/17/toy-layout-engine-6-block.html>

Styleツリーを入力として受けとり、[CSSのボックスモデル](https://developer.mozilla.org/ja/docs/Learn/CSS/Building_blocks/The_box_model)に対応したレイアウト情報を持つボックスの集合、レイアウトツリーを作ります。

:::message
ボックスモデルについての説明はMDNの記事がわかりやすかったです。
https://developer.mozilla.org/ja/docs/Learn/CSS/Building_blocks/The_box_model
:::

Styleツリーの情報から、各ノードに対応するボックスの幅、高さ、ポジションをそれぞれ計算します。
heightが指定されていないとき、ボックスの高さは、そのボックスが内包する子要素の高さによって決まるので、子要素から再帰的に高さを計算して、最後に親要素の高さを決めるというロジックなど勉強になりました。

この5章、6章がなかなか複雑で個人的に1番の難所だった気がします（今も理解できているか怪しい）。

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/layout_box.ts#L1-L295

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/layout_box.test.ts


## Part 7: Painting 101

![](https://i.gyazo.com/389707613bc8f18b75bb19a9b236fa1b.png)


**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/painting.ts#L1-L164

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/painting.test.ts#L1-L151

## Appendix

記事のすべての実装をやり終えたときに、せっかく作ったので画像出力だけでなく、これをWeb上で動かせたらおもしろいかもと思いました。そこで、最近メジャーバージョンがリリースされたDenoのWebフレームワークである[Fresh](https://fresh.deno.dev/)で実装しみました。

https://deno-toy-rendering-engine.deno.dev/

Freshの設計思想の、[islands architecture](https://jasonformat.com/islands-architecture/) に最初戸惑ったのですが、なれてみれば責務が明確で良いなと思いました。

レンダリングエンジンの組み込みも、Paintingの章で行った画像出力部分を、HTMLのCanvasに置き換えるだけで動いたので、やりやすかったです。

Deploy先も Deno Deploy を試してみましたが、サクッと公開できて良いですね。総じて開発体験が良かったです。
freshでのデモサイト実装はこちらにあります。

https://github.com/kawamataryo/deno-toy-rendering-engine/tree/main/web

# 全体を通して

## テストの大切さ

## Deno の使いやすさ

## Rust の文法の表現の豊かさ

## ブラウザの動作理解への助け

# 参考

https://developer.chrome.com/blog/inside-browser-part1/

https://dackdive.hateblo.jp/entry/2021/02/23/113522
