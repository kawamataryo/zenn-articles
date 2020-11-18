---
title: "Tailwind CSS の Purge でビルドサイズを縮小する"
emoji: "🍏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Tailwind", "css", "PostCSS"]
published: true
---

Tailwind CSS はユーティリティファーストでとても使いやすいのですが、無数のクラスが提供されているので、そのままだとビルドサイズがとても巨大ですよね。

それを解消するために、 Tailwind CSS に標準で搭載されている Purge オプションを使ってビルドサイズの縮小を試してみました。

# テスト環境の構築
最初に Tailwind CSS の環境を構築します。

まず npm プロジェクトを作成して、Tailwind CSS を依存に追加します。

```bash
$ yarn init -y
$ yarn add tailwindcss
```

次に `tailwind.config.js` を作成します。

```bash
$ npx tailwindcss init
```

コマンドを実行するとこんな感じのファイルがルートに作成されるはずです。

```js:tailwind.config.js
module.exports = {
  future: {},
  purge: [],
  theme: {
    extend: {},
  },
  variants: {},
  plugins: [],
}
```

次に、以下内容で`src`配下に`style.css`を追加します。

```css:src/style.css
@tailwind base;

@tailwind components;

@tailwind utilities;
```

最後に表示用の html を追加します。

```html:index.html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Tailwind purge sample</title>
  <link rel="stylesheet" href="dist/style.css">
</head>
<body>
  <div class="text-base">Hello World</div>
  <div id="sample">foo</div>
</body>
</html>
```

この状態で以下のようなディレクトリ構成になっているはずです。

```bash
├── package.json
├── src
│     └── style.css
├── index.html
├── tailwind.config.js
└── yarn.lock
```

これで準備は完了です。


# Purgeの実行

最初に Purge しない場合を試してみます。

前項で、準備した環境そのままでビルドを実行します。

```bash
 $ NODE_ENV=production npx tailwindcss build src/style.css -o dist/style.css
 
✅ Finished in 1.41 s
📦 Size: 2.31MB
💾 Saved to dist/style.css

```
**2.3MB**と中々巨大な style.css が生成されます。
これではとてもプロダクションで使うことはできませんね…


ということで、Purge を実行してみます。

Purge の実行は、`tailwind.config.js`の`purge`オプションにプロジェクト内でクラスを参照しているファイルを指定するだけです。

今回は以下のように設定しました。

```js:tailwind.config.js
module.exports = {
  future: {},
  purge: ['./index.html'],
  theme: {
    extend: {},
  },
  variants: {},
  plugins: [],
}
```

:::message
purge オプションの配列には`src/**/*.js`, `src/**/*.vue`などのように[Globパターン](https://github.com/isaacs/node-glob/blob/master/README.md#glob-primer)での指定が可能です。
:::

これで再度ビルドを実行します。

```bash
 NODE_ENV=production npx tailwindcss build src/style.css -o dist/style.css

✅ Finished in 1.33 s
📦 Size: 11.33KB
💾 Saved to dist/styles.css
```

**11.3KB**と一気に減りました！

実際の生成物を確認しても、`index.html`で参照している`text-base`のクラスは残りつつ、他は全体に影響するリセット CSS 等のみになっています。
良い！

# Purge する際の注意点

purge オプションに動的にクラスの付け替えを行っているスクリプトファイルを指定しても、そこで参照しているクラスは Purge の対象からは外れます。
これは JSX でも、Vue でも同じです。

例えば以下のような場合です。

`index.js`があるとしてそのファイルパスを`tailwind.config.js`の`purge`の配列に追加すればとクラス検索の対象になり、`text-blue-500`は Purge 対象から外れ、ビルド生成物にもクラスが残ります。

```js:index.js // text-blue-500はpurgeされない const text_blue_500 = 'text-blue-500'
document.querySelector('#sample').classList.add(text_blue_500)
```

ただし、**動的にクラス名の文字列を組み立てる場合は注意が必要**です。

以下のようにカラーレベルを動的に組み立て指定をしている場合は、`color-green-500`は Purge されてしまいます。

```js:index.js
// text-green-500はpurgeされる
const color_level = 500
document.querySelector('#sample').classList.add(`text-green-${color_level}`)
```

これは Tailwind CSS が Purge の際に内部的に利用する、[PurgeCSS](https://purgecss.com) が JavaScript を動的に実行した結果でパージ対象を選別をしているのではなく、コードに対して以下正規表現でクラス名を抽出して Purge 対象の選別を行っているからです。

```
/[^<>"'`\s]*[^<>"'`\s:]/g
```

スクリプトでクラス名の文字列を動的に組み立てている処理がある場合は注意してください。

なお、どうしても動的にクラス名の文字列を生成したい場合は、生成するクラス名を PurgeCSS の`whitelist`オプションを指定して Purge 対象から除外することで対応が可能です。

```js:tailwind.config.js
module.exports = {
  future: {},
  purge: {
    content: ['./index.html', './index.js']
    options: {
      whitelist: ['text-green-500', 'text-green-400', 'text-green-400']
    }
  }
  theme: {
    extend: {},
  },
  variants: {},
  plugins: [],
}
```

# 終わりに

以上、「Tailwind CSS の Purge でビルドサイズを圧縮する」でした。
これを調べる前は「ユーティリティクラスとかどれだけ巨大な CSS を読み込む必要があるんだ…？」と思ってたのですが、簡単な指定をするだけで Purge CSS を使って良い感じビルドサイズの制御を行ってくれるので、良いですね。

Purge CSS 自体は別で Tailwind CSS 関係ないので、他プロダクトでも積極的に利用していきたいと思いました。

# 参考

- [Controlling File Size - Tailwind CSS](https://tailwindcss.com/docs/controlling-file-size)