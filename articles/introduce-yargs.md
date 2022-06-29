---
title: "はじめてのyargs - Node.jsでコマンドライン引数をパースする -"
emoji: "🏴‍☠️"
type: "tech"
topics: ["yargs", "typescript", "cli", "nodejs"]
published: false
---

最近yargsでいくつかCLIツールを作ったので備忘録としてまとめます。

※ yargs: v17.5.1 時点の情報です。

# yargsとは？

Node.jsにてコマンドライン引数をパースするライブラリです。
自作のCLIツールの作成時などに便利です。

主な機能はこちらです。

* コマンドライン引数のパース
* ヘルプやバージョンの自動生成
* bash補完スクリプトの生成

https://github.com/yargs/yargs

## Get started

コマンドラインで受け取った引数を出力するだけの簡単なCLIツールを作ってみます。

はじめに、任意のディレクトリで以下コマンドを実行し、npmのプロジェクトを作成します。

```bash
$ npm init -y
```

次に、TypeScriptの実行環境を作ります。

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

yargsをnpmからインストールします。TypeScriptの型定義はDefinitelyTypedで提供さてています。

```bash
$ npm i yargs
```

```bash
$ npm i -D @types/yargs
```

index.jsを以下のように修正します。

```ts:index.ts
import yargs from 'yargs';

const args = yargs
  .command("* <message>", "print a message received as an argument")
  .parseSync()

console.log(args.message)
```

ts-nodeで実行します。
標準出力に`hogehoge`と表示されるはずです。

```bash
$ npx ts-node src/index.ts hogehoge
```

また、`--help` optionをつけて実行すると以下のようなヘルプが出力されます。

```bash
$ npx ts-node src/index.ts --help
index.ts <message>

print a message received as an argument

Options:
  --help     Show help                                                 [boolean]
  --version  Show version number                                       [boolean]
```

その他、バージョンも出力できます。ここで表示されるバージョンはpackage.jsonの`version`の値を参照しています。

```bash
$ npx ts-node src/index.ts --version
1.0.0
```

とても便利ですね。

# 逆引きyargs

yargsの使い方を逆引き風に紹介します。


## オプション引数をパースする

オプション引数は `.options()`で指定できます。
以下例では、必須のboolean型のオプションとして`capitalize`を指定しています。

```ts
const args = yargs
  .command("* <message>", "print a message received as an argument")
  .options({
    "capitalize": {
      type: "boolean",
      describe: "convert received messages to uppercase",
      demandOption: true,
      default: false
    }
  })
  .parseSync()

console.log(args.capitalize) // true or false
```

http://yargs.js.org/docs/#api-reference-optionskey-opt


## 位置引数をパースする

位置引数は `.positional()`で指定できます。
以下例では、必須のstring型の位置引数を`message`として指定しています。

```ts
import yargs from 'yargs';

const args = yargs
  .command("* <message>", "print a message received as an argument")
  .positional("message", {
    describe: "message for printing",
    type: "string",
    demandOption: true,
  })
  .parseSync()

console.log(args.message)
```

http://yargs.js.org/docs/#api-reference-positionalkey-opt

## エイリアスを指定する

`.alias()`またはオプション引数の`alias`でエイリアスも指定できます。
以下例では、`capitalize`のオプションに`c`を、versionのオプションに`v`、ヘルプのオプションに`h`をエイリアスとして設定しています。

```ts
const args = yargs
  .command("* <message>", "print a message received as an argument")
  .options({
    "capitalize": {
      type: "boolean",
      describe: "convert received messages to uppercase",
      demandOption: true,
      default: false,
      alias: 'c'
    }
  })
  .alias({
    'h': 'help',
    'v': 'version'
  })
  .parseSync()
```

helpの出力はこちらです。

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
    }
  })
  .check((args) => {
    if(args['even-number'] % 2 === 0) {
      return true
    } else {
      throw new Error('Error: please specify an even number')
    }
  })
  .parseSync()
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

## 任意の引数をパースする
パスの一覧など、任意の数の位置引数を指定したい場合は、commandの第一引数で`[arg...]`を指定します。
positionalのオプションで、`array`を指定しておくとTypeScriptの型的にも配列になります。
どの型の配列になるかは`type`の指定により変わります。今回は`type: "string"`としているので、`args.paths`は文字列の配列として型推論されます。

```ts
import yargs from 'yargs';

const args = yargs
  .command("* [paths...]", "print a paths received as arguments")
  .positional("paths", {
    describe: "paths for printing",
    type: "string",
    demandOption: true,
    array: true
  })
  .parseSync()

console.log(args.paths)
```

# おわりに

yargsがないと Node.jsのCLIツールを作る気がおきないくらい必須のライブラリです。
ここで紹介した以外にも便利なオプションがさまざまあります。

自作のCLIツールを作る際には是非利用してみてください。

http://yargs.js.org/contributing/