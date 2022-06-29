---
title: "はじめてのyargs - Node.jsでコマンドライン引数をパース"
emoji: "🏴‍☠️"
type: "tech"
topics: ["yargs", "typescript", "cli", "nodejs"]
published: false
---

最近 [yargs](https://github.com/yargs/yargs) を使っていくつか CLI ツールを作ってみたので備忘録としてまとめます。

※ yargs: v17.5.1 時点の情報です。

# yargs とは？

Node.js にてコマンドライン引数をパースするライブラリです。
自作の CLI ツールの作成時などに便利です。

主な機能はこちらです。

- コマンドライン引数のパース
- ヘルプやバージョンの自動生成
- bash 補完スクリプトの生成

https://github.com/yargs/yargs

## Get started

コマンドラインで受け取った引数を出力するだけの簡単な CLI ツールを作ってみます。

はじめに、任意のディレクトリで以下コマンドを実行し、npm のプロジェクトを作成します。

```bash
$ npm init -y
```

次に、TypeScript の実行環境を作ります。

```bash
$ npm i -D typescript
```

```bash
$ mkdir src
$ echo "console.log('hello yargs!')" > src/index.ts
```

`ts-node` で実行して、`hello yargs` と表示されれば下準備は完了です。

```bash
$ npx ts-node src/index.ts
hello yargs!
```

yargs を npm からインストールします。TypeScript の型定義は DefinitelyTyped で提供さてています。

```bash
$ npm i yargs
```

```bash
$ npm i -D @types/yargs
```

index.js を以下のように修正します。

```ts:index.ts
import yargs from 'yargs';

const args = yargs
  .command("* <message>", "print a message received as an argument")
  .parseSync()

console.log(args.message)
```

ts-node で実行します。
標準出力に`hogehoge`と表示されるはずです。

```bash
$ npx ts-node src/index.ts hogehoge
```

また、`--help` option をつけて実行すると以下のようなヘルプが出力されます。

```bash
$ npx ts-node src/index.ts --help
index.ts <message>

print a message received as an argument

Options:
  --help     Show help                                                 [boolean]
  --version  Show version number                                       [boolean]
```

その他、バージョンも出力できます。ここで表示されるバージョンは package.json の`version`の値を参照しています。

```bash
$ npx ts-node src/index.ts --version
1.0.0
```

とても便利ですね。

# 逆引き yargs

yargs の使い方を逆引き風に紹介します。

## オプション引数をパースする

オプション引数は `.options()`で指定できます。
以下例では、必須の boolean 型のオプションとして`capitalize`を指定しています。

```ts
const args = yargs
  .command("* <message>", "print a message received as an argument")
  .options({
    capitalize: {
      type: "boolean",
      describe: "convert received messages to uppercase",
      demandOption: true,
      default: false,
    },
  })
  .parseSync();

console.log(args.capitalize); // true or false
```

http://yargs.js.org/docs/#api-reference-optionskey-opt

## 位置引数をパースする

位置引数は `.positional()`で指定できます。
以下例では、必須の string 型の位置引数を`message`として指定しています。

```ts
import yargs from "yargs";

const args = yargs
  .command("* <message>", "print a message received as an argument")
  .positional("message", {
    describe: "message for printing",
    type: "string",
    demandOption: true,
  })
  .parseSync();

console.log(args.message);
```

http://yargs.js.org/docs/#api-reference-positionalkey-opt

## エイリアスを指定する

`.alias()`またはオプション引数の`alias`でエイリアスも指定できます。
以下例では、`capitalize`のオプションに`c`を、version のオプションに`v`、ヘルプのオプションに`h`をエイリアスとして設定しています。

```ts
const args = yargs
  .command("* <message>", "print a message received as an argument")
  .options({
    capitalize: {
      type: "boolean",
      describe: "convert received messages to uppercase",
      demandOption: true,
      default: false,
      alias: "c",
    },
  })
  .alias({
    h: "help",
    v: "version",
  })
  .parseSync();
```

help の出力はこちらです。

```
index.ts <message>

print a message received as an argument

Options:
  -c, --capitalize  convert received messages to uppercase
                                      [boolean] [required] [default: false]
  -h, --help        Show help                                     [boolean]
  -v, --version     Show version number                           [boolean]
```

http://yargs.js.org/docs/#api-reference-aliaskey-alias

## 引数の値を検証する

`check()`を使うことで引数の値を検証することもできます。
以下は、偶数の数値が指定されることを想定したオプション`even-number`の検証例です。

```ts
const args = yargs
  .command("* <message>", "print a message received as an argument")
  .options({
    "even-number": {
      type: "number",
      describe: "set even number",
      demandOption: true,
    },
  })
  .check((args) => {
    if (args["even-number"] % 2 === 0) {
      return true;
    } else {
      throw new Error("Error: please specify an even number");
    }
  })
  .parseSync();
```

偶数を指定しなかった場合、以下のような出力となります。

```bash
$ npx ts-node src/index.ts --even-number 3 hoge      ✘ 1
index.ts <message>

print a message received as an argument

Options:
  --help         Show help                                        [boolean]
  --version      Show version number                              [boolean]
  --even-number  set even number                        [number] [required]

Error: please specify an even number
```

http://yargs.js.org/docs/#api-reference-checkfn-globaltrue

## 任意数の位置引数をパースする

パスの一覧など、任意の数の位置引数を指定したい場合は、command の第一引数で`[arg...]`を指定します。
positional のオプションで、`array`を指定しておくと TypeScript の型的にも配列になります。
どの型の配列になるかは`type`の指定により変わります。今回は`type: "string"`としているので、`args.paths`は文字列の配列として型推論されます。

```ts
import yargs from "yargs";

const args = yargs
  .command("* [paths...]", "print a paths received as arguments")
  .positional("paths", {
    describe: "paths for printing",
    type: "string",
    demandOption: true,
    array: true,
  })
  .parseSync();

console.log(args.paths);
```

# おわりに

yargs がないと Node.js の CLI ツールを作る気がおきないくらい必須のライブラリです。
ここで紹介した以外にも便利なオプションがさまざまあります。

自作の CLI ツールを作る際には是非利用してみてください。

http://yargs.js.org/docs/
