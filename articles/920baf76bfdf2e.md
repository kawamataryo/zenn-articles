---
title: "Deno で簡易レンダリングエンジンを作ってみた"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "TypeScript", "browser"]
published: false
---

# 作ったもの

Web ブラウザの仕組みを基礎から勉強したいと思い、「[Let's build a browser engine!](https://limpet.net/mbrubeck/2014/08/08/toy-layout-engine-1.html)」の記事を参考に Deno で簡易的な HTML レンダリングエンジンを作ってみました。

簡易的という言葉の通り、実用性はないです。

以下 GIF のように、HTML と CSS を入力として受け取り、Canvas にボックスを描画するだけです。

![](https://i.gyazo.com/0ff561a6683205a130cf4f444cb8f19f.gif)

https://deno-toy-rendering-engine.deno.dev/

描画に対応しているものは、ブロック要素のレイアウトのみで、使える CSS もごくわずか。サイズ・位置指定（width、 height、padding、margin、border-width）と装飾（background-color、border-color）のみ。

テキストの描画もできません。

ただ、ひとつひとつの過程を自分で実装していくので、レンダリングエンジンの仕組みを勉強するにはとても良いものでした。

![](https://i.gyazo.com/2d8cbf42388d833d54f10f7c17eb77d3.png)
_本記事で実装するレンダリングエンジンの流れ_

参考記事では、Rust で作られていたのですが、~~Rust のコンパイラが怖いので~~ TypeScript で実装してみました。また、ランタイムは Node.js ではなく [Deno](https://deno.land/) を選択しています。 実装コードはすべてこちらのリポジトリで公開しています。

https://github.com/kawamataryo/deno-toy-browser-engine

:::message
「これでレンダリングエンジンと言えるのか!!😡 」というマサカリが怖いですが、あくまで簡易的ですし、参考記事もレンダリングエンジンと言っているのでお許しを。。ちなみに、参考記事の筆者[Matt Brubeck](https://github.com/mbrubeck) さんはもともと Mozilla で Firefox の開発をしていた方のようです。
https://hacks.mozilla.org/author/mbrubeckmozilla-com/
:::

# 章ごとの所感

## Part 1: Getting started, Part 2: HTML

![](https://i.gyazo.com/43006f9b63e958494409e643d00cee4d.png)
<https://limpet.net/mbrubeck/2014/08/08/toy-layout-engine-1.html>
<https://limpet.net/mbrubeck/2014/08/11/toy-layout-engine-2.html>

この記事で作るレンダリングエンジンの全体の流れを説明した上で、HTML の文字列を受け取り、DOM ノードツリーを構築するのパーサーを作ります。

この章で作る HTML パーサーが対応している HTML 構文は以下のみです。

- `<p>`</p>``などのオープンとクローズが対になったタグ
- タグに付与する id と class の属性
- `hello world`などのテキストノード

`<!-- -->`でのコメントや`<br />`の自己完結タグなどはすべて未対応です。

`consumeChar`等のメソッドをつくり、再帰やループを駆使して 1 文字ずつ解析していく過程で、パーサーの実装が徐々に理解できました。

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/html_parser.ts#L1-L141

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/html_parser.test.ts#L1-L116

## Part 3: CSS

![](https://i.gyazo.com/20fb75e12e49e1fd730d052632261f49.png)

<https://limpet.net/mbrubeck/2014/08/13/toy-layout-engine-3-css.html>

次に CSS も HTML と同じようにパーサーを作りパースします。CSS の場合は階層構造とはなっていないので、パース結果は、ツリーではなくオブジェクトの配列になります。

ここで作るパーサーでも、すべてのプロパティと値をサポートすると莫大なコード量が必要なので、最低限のものしかサポートしていません。

プロパティは

- ボックスのサイズ・レイアウト指定（width、 height、 padding、margin、border-width）
- ボックスのカラーリング（background-color, border-color)

値は

- サイズは数値 + `px`
- カラーは HEX（`#00ff00`など）

のみです。

:::message
記事中で、HTML パーサーの説明は懇切丁寧に書かれていたのですが、CSS パーサーの実装説明は最低限必要なところのみでした。ちょっと不親切かもと思いつつ、ちょうど HTML パーサーでパーサーの基本は理解したあとなので、サンプルコードを見ずともなんとか実装できました。
:::

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/css_parser.ts#L1-L189

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/css_parser.test.ts

## Part 4: Style

![](https://i.gyazo.com/f6ce1ea7f32c618575d1440d4a3f1fcb.png)

<https://limpet.net/mbrubeck/2014/08/23/toy-layout-engine-4-style.html>

DOM ノードツリーと CSS のパース結果を入力として受けとり、各ノードに CSS のスタイル定義が割り当てられた Style ツリーというものを作ります。

DOM ノードツリーに新しいフィールドを追加することでも同じような出力結果を得ることはできますが、Style の指定によっては、後続の処理に渡さない（たとえば`display: none`など）こともあるので、新しいノードを作るほうが効率的とのことです。これは今後の処理でも同じで、都度新しい構造体を作っていきます。

`id`、`class`、`tag`
などセレクターでの詳細度を考慮して、ノードのスタイルを決める（同名のプロパティの場合は、詳細度の高いものを優先する）処理など勉強になりました。

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/styled_node.ts#L1-L130

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/styled_node.test.ts#L1-L197

## Part 5: Boxes, Part 6: Block layout

![](https://i.gyazo.com/494404977dad6e6657d894bdc6593510.png)

<https://limpet.net/mbrubeck/2014/09/08/toy-layout-engine-5-boxes.html>
<https://limpet.net/mbrubeck/2014/09/17/toy-layout-engine-6-block.html>

Style ツリーを入力として受けとり、[CSS のボックスモデル](https://developer.mozilla.org/ja/docs/Learn/CSS/Building_blocks/The_box_model)に対応したレイアウト情報を持つボックスの集合（レイアウトツリー）を作ります。

:::message ボックスモデルについての説明は MDN の記事がわかりやすかったです。
https://developer.mozilla.org/ja/docs/Learn/CSS/Building_blocks/The_box_model
:::

Style ツリーの情報から、各ノードに対応するボックスの幅、高さ、ポジションをそれぞれ計算します。

height が指定されていないときのボックスの高さは、そのボックスが内包する子要素の高さによって決まるので子要素から再帰的に高さを計算し、最後に親要素の高さを決める。インライン要素を配置するときは匿名ブロックで覆うなどのロジックが強になりました。

この 5 章、6 章がなかなか複雑で個人的に 1 番の難所だった気がします（今も理解できているか怪しい 😅 ）。

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/layout_box.ts#L1-L295

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/layout_box.test.ts

## Part 7: Painting 101

![](https://i.gyazo.com/389707613bc8f18b75bb19a9b236fa1b.png)
<https://limpet.net/mbrubeck/2014/11/05/toy-layout-engine-7-painting.html>

レイアウトツリーを受け取り描画処理に使うピクセルの配列を出力するモジュールを使っています、実際に画像に出力するまでを行います。

流れとしては、レイアウトツリーを最初にディスプレイコマンドという描画処理を内包するクラスの配列に変換し、そのコマンドを順に実行することでピクセル情報を取得、そのピクセル情報から画像へ出力というものでした。
画像出力部分は、deno-canvas というライブラリを使って実装しています。

https://github.com/DjDeveloperr/deno-canvas

レイアウトツリーをそのままピクセル化するのではなく、一度ディスプレイコマンドに変換するこで、ムダな描画を排除したいり、処理の再利用ができるそうです。

最終的に、この Tweet のように HTML と CSS から画像を出力することができました。
結果としては画像を出すだけですが、パーサーからひとつひとつ作ってきたので達成感はすごかったです。

https://twitter.com/KawamataRyo/status/1557148537745702912

**実装コード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/painting.ts#L1-L164

**テストコード**
https://github.com/kawamataryo/deno-toy-rendering-engine/blob/main/src/lib/%5F%5Ftests%5F%5F/painting.test.ts#L1-L151

## Appendix

記事のすべての実装をやり終えた後、「せっかく作ったので画像出力だけでなく、これを Web 上で動かせたらおもしろいかも」と思いました。そこで、最近メジャーバージョンがリリースされた Deno の Web フレームワークである[Fresh](https://fresh.deno.dev/)で実装しみました。

https://deno-toy-rendering-engine.deno.dev/

実装時は、Fresh の設計思想の[islands architecture](https://jasonformat.com/islands-architecture/)
に最初戸惑ったのですが、なれてみれば責務が明確で良かったです。

レンダリングエンジンの組み込みも、Painting の章で行った画像出力部分を、HTML の Canvas に置き換えるだけで動いたので、やりやすかったです。

Deploy 先も Deno Deploy を試してみました。サクッと公開できて良いですね。総じて開発体験が良かったです。
fresh でのデモサイトの実装は以下にあります。

https://github.com/kawamataryo/deno-toy-rendering-engine/tree/main/web

# 全体を通して

## 多言語で再実装するという学習方法

今回はじめてサンプルコードとは別の言語で再実装するという学習方法を試してみたのですが、これが思いのほか良かったです。

そのまま写経する場合は内容を理解してなくとも動いてしまうのですが、言語構造の違う言語で再実装しようとすると、ある程度内容を理解していないと動かすことが難しいです。時間はかかったのですが、再実装する過程で大分理解が深まりました。

その他、この学習方法を行う上で大切なのは、テストを書くことだと思いました。やはり実行しないとわからない部分がとても多く、そのときに手軽に実行結果を試せるテストを書いておくことはテストコードを書く時間を上回る、効率化に繋がったと思います。

## Deno の使いやすさ

ちゃんと Deno をランタイムとして使ったのは今回がはじめてなのですが、とても開発体験が良かったです。
素で TS が使える点、formatter と linter が同梱されている点など、機能としてはどれも Node.js でもできることなのですが、外部ライブラリに依存せず設定不要で使えることは、開発体験に大きく寄与すると感じました。

また TDD で開発していたので、テストを何度も実行していたのですが、そのテストの実行速度の速さにも驚きました。

最近速いと話題の [Vitest](https://vitest.dev/)と使い比べて見たのですが、Vitest よりもさらに数段早かったです。

## Rust の文法の表現の豊かさ

単純に Rust 良いなーと思いました w
参考記事では、Rust の Enum のデータ添付やパターンマッチでの処理の分岐がよく使われていたのですが、これを TS で再現すると、Rust と比べかなり冗長なコードになってしまいました。

とくにパターンマッチは強力だと感じたので、是非 ECMAScript にも入って欲しいなと思っています。

:::message
ECMAScript にもパターンマッチのプロポーザルは出ているようです。ただまだ stage 1 なのでかなり先になりそう。
https://github.com/tc39/proposal-pattern-matching
:::

_とくに再現に苦労したコード_ ([Part 6: Block layout より](https://limpet.net/mbrubeck/2014/09/17/toy-layout-engine-6-block.html))

```rust
match (width == auto, margin_left == auto, margin_right == auto) {
    // If the values are overconstrained, calculate margin_right.
    (false, false, false) => {
        margin_right = Length(margin_right.to_px() + underflow, Px);
    }

    // If exactly one size is auto, its used value follows from the equality.
    (false, false, true) => { margin_right = Length(underflow, Px); }
    (false, true, false) => { margin_left  = Length(underflow, Px); }

    // If width is set to auto, any other auto values become 0.
    (true, _, _) => {
        if margin_left == auto { margin_left = Length(0.0, Px); }
        if margin_right == auto { margin_right = Length(0.0, Px); }

        if underflow >= 0.0 {
            // Expand width to fill the underflow.
            width = Length(underflow, Px);
        } else {
            // Width can't be negative. Adjust the right margin instead.
            width = Length(0.0, Px);
            margin_right = Length(margin_right.to_px() + underflow, Px);
        }
    }

    // If margin-left and margin-right are both auto, their used values are equal.
    (false, true, true) => {
        margin_left = Length(underflow / 2.0, Px);
        margin_right = Length(underflow / 2.0, Px);
    }
}
```

# 参考

https://developer.chrome.com/blog/inside-browser-part1/

https://dackdive.hateblo.jp/entry/2021/02/23/113522
