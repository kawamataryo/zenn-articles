---
title: "Python の dotted path をコピーする VS Code 拡張機能を作ってみた"
emoji: "🫐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "python", "typescript"]
published: false
---

# 🔧 作ったもの

[Copy Python Path](https://marketplace.visualstudio.com/items?itemName=kawamataryo.copy-python-dotted-path) という Python の dotted path （例 `foo.hoge.class`）をコピーする拡張機能を作りました。

Django で unittest を実行する際に、dotted path を使うのですが、それを毎回手動で組み立てるのが面倒で作りました。

機能はこちらです。

- クラス or メソッドの定義行でコマンドを実行すると、ルートからそのクラス or メソッドまでの dotted path をクリップボードにコピーする
- クラス or メソッドの定義行以外でコマンドを実行すると、ルートからそのファイルまでの dotted path をクリップボードにコピーする

![](https://i.gyazo.com/fe88befdaea034eff0adfd4caacd028f.gif)

:::message
既存でも dotted path をコピーする拡張機能はあるのですが、どれもファイル単位のパスコピーまでで、クラス・メソッド単位までのコピーには対応していませんでした。
:::



こちらからインストールできます。レビューをもらえると泣いて喜びます。

https://marketplace.visualstudio.com/items?itemName=kawamataryo.copy-python-dotted-path

コードもこちらで公開しています。
https://github.com/kawamataryo/copy-python-path

# 🦾 仕組み & 工夫したところ

## JS で Python コードをパースしてメソッド・クラス定義を解析

ファイルまでの相対パスだけならば VS Code の API から取得できるのですが、クラス・メソッドの定義までのパスとなると、VS Code の API から取得することはできません。
正確にクラス・メソッド名、定義位置（階層構造）を取得するためには Python コードをパースする必要がありました。

そこで今回は、 [ANTLR4](https://github.com/antlr/antlr4) ベースのパーサージェネレーターである [dt-python-parser]() を使ってみました。

https://www.npmjs.com/package/dt-python-parser

このライブラリに Python コードを与えると AST（抽象構文木）にパースしてくれ、さらに ANTLR4 ベースのメソッドで、AST をトラバースできます。

dt-python-parser を使った copy-python-path の解析部分のコードはこちらです。

https://github.com/kawamataryo/copy-python-path/blob/main/src/utils/getRelatedDefinedSymbols.ts

```ts:src/utils/getDefinedParentSymbols.ts
// ...

/**
 * Get defined symbols related to the selected rows from a python file. e.g. Class name, function name
 * @param  {string} text - python code
 * @param  {number} lineNumber - current cursor line number
 * @return {array} defined symbols
 */
export const getRelatedDefinedSymbols = (text: string, lineNumber: number): string[] => {
  const parser = new Python3Parser();
  const tree = parser.parse(text);

  const symbols: DefinedSymbol[] = [];
  class MyListener extends Python3Listener {
    enterClassdef(ctx: any): void {
      symbols.push({
        name: ctx.children[1].getText(),
        line: ctx.children[0].getSymbol().line,
        column: ctx.children[0].getSymbol().column,
      });
    }
    enterFuncdef(ctx: any): void {
      symbols.push({
        name: ctx.children[1].getText(),
        line: ctx.children[0].getSymbol().line,
        column: ctx.children[0].getSymbol().column,
      });
    }
  }
  const listenTableName = new MyListener();
  parser.listen(listenTableName, tree);

  const symbol = symbols.filter(s => s.line === lineNumber)[0];
  if (!symbol) {
    return [];
  }
  if (symbol.column === 0) {
    return [symbol.name];
  }

  const parentSymbolNames = getDefinedParentSymbols(symbol, symbols).map(s => s.name);
  return [...parentSymbolNames, symbol.name];
};
```

dt-parser でクラス名・メソッド名と、その定義位置を取得しコードの構造に合わせて並び替えた配列を作っています。
コード構造に合わせた並び替えは、`getDefinedParentSymbols`内にて再帰処理で行っています。

## E2E テストで動作を担保

仕組みは簡単な拡張機能ですが、継続してメンテナンスしやすいように E2E テストを追加してみました。
以下の画像のようにテストを実行すると、VS Code が別プロセスで立ち上がり実際コマンドを実行してくれます。

![](https://i.gyazo.com/83a568023b703b8d9fbff5e93d5fa5b2.gif)

テストコードはこちらです。

```ts:src/test/suite/extension.test.ts
// ...

const executeCommandWithWait = async (command: string): Promise<any> => {
  await sleep(500);
  await vscode.commands.executeCommand(COMMAND_NAME);
  await sleep(1000);
};

const COMMAND_NAME = 'copy-python-path.copy-python-path';

// example.py is following code
/*
class ClassA:
    def class_a_method_a():
        pass

    class ClassB:
       def class_b_method_a():
           pass

class ClassD:
    def class_d_method_a():
       pass
*/
const testFileLocation = '/pythonApp/example.py';

suite('Extension Test Suite', () => {
  vscode.window.showInformationMessage('Start all tests.');
  let editor: vscode.TextEditor;

  setup(async () => {
    // open folder
    const fileUri = vscode.Uri.file(vscode.workspace.workspaceFolders![0].uri.fsPath + testFileLocation);
    const document = await vscode.workspace.openTextDocument(fileUri);
    editor = await vscode.window.showTextDocument(document);
  });

  test('selected class lines', async () => {
    editor.selection = new vscode.Selection(new vscode.Position(0, 0), new vscode.Position(0, 0));

    await executeCommandWithWait(COMMAND_NAME);

    assert.strictEqual(await vscode.env.clipboard.readText(), 'pythonApp.example.ClassA');
  });

  test('selected method lines', async () => {
    editor.selection = new vscode.Selection(new vscode.Position(1, 0), new vscode.Position(1, 0));

    await executeCommandWithWait(COMMAND_NAME);

    assert.strictEqual(await vscode.env.clipboard.readText(), 'pythonApp.example.ClassA.class_a_method_a');
  });

  test('selected nested class lines', async () => {
    editor.selection = new vscode.Selection(new vscode.Position(4, 0), new vscode.Position(4, 0));

    await executeCommandWithWait(COMMAND_NAME);

    assert.strictEqual(await vscode.env.clipboard.readText(), 'pythonApp.example.ClassA.ClassB');
  });

  // ...
});

```

カーソル位置を移動しながら、コマンドを実行し、クリップボードへのコピー結果を検証しています。
適宜、setTimeoutを使い実行感覚を開けている所がポイントです。

実は VS Code 拡張機能のテストの方法は、ほぼほぼ情報がなくとても苦労しました。VS Code のドキュメントにも Hello World レベルの簡単なテスト記載しかなく、結局、Microsoft の出している拡張機能の実際のテストコードをみながらテストを書きました。

※ 参考になったリポジトリ
https://github.com/microsoft/vscode/tree/main/extensions/typescript-language-features/src/test

## GitHub Actions での CI 環境の整備

VS Code のドキュメントに CI の情報があったので、そちらを参考に E2E テストと拡張機能のリリースの CI を GitHub Actions で組んでみました。
GitHub Actions のイメージに VS Code の起動に必要なパッケージは入っているようで、ほぼ考慮点はなく実行することができました。

以下が E2E テストの GitHub Actions です。
Mac と Linux（ubuntu）と Windows のイメージで並列してテストを実しています。

https://github.com/kawamataryo/copy-python-path/blob/main/.github/workflows/e2e-test.yml

```yml:.github/workflows/e2e-test.yml
name: E2E

on:
  push:
    branches:
      - main

jobs:
  build:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '16'
        cache: 'yarn'
    - name: Install dependencies
      run: yarn install
    # xvfb is required to run vscode on linux
    - name: Run test on linux
      run: xvfb-run -a yarn test
      if: runner.os == 'Linux'
    - name: Run test on Mac and Windows
      run: yarn test
      if: runner.os != 'Linux'
```

:::message
実際に自分の作っていた拡張機能を GitHub Actions で動かした際に、Windows のビルドだけテストが落ちて、プラットフォーム依存のバグがあることに気付けました。テストはやはり大事。
:::

リリースはこちらです。

```yml:.github/workflows/release.yml
name: release

on:
  workflow_dispatch:
    inputs:
      release_type:
        description: 'select release type'
        type: choice
        options:
         - patch
         - minor
         - major
        required: true

jobs:
  build:
    strategy:
      matrix:
        os: [macos-latest, ubuntu-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
    - name: Checkout
      uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '16'
        cache: 'yarn'
    - name: Install dependencies
      run: yarn install
    # xvfb is required to run vscode on linux
    - name: Run test on linux
      run: xvfb-run -a yarn test
      if: runner.os == 'Linux'
    - name: Run test on Mac and Windows
      run: yarn test
      if: runner.os != 'Linux'
    - name: Publish
      if: success() && matrix.os == 'ubuntu-latest'
      run: |
        git config --global user.name "${GITHUB_ACTOR}"
        git config --global user.email "${GITHUB_ACTOR}@users.noreply.github.com"
        yarn run deploy -- ${{ github.event.inputs.release_type }}
      env:
        VSCE_PAT: ${{ secrets.VSCE_PAT }}
```

リリースバージョンを管理するために、リリース種別をオプションで受け取る手動ワークフローとしています。
E2E テストの CI と同じ処理の実行後、リリースのスクリプトを実行しています。
リリース自体は[vsce](https://github.com/microsoft/vscode-vsce)を利用しているのですが、その内部でバージョンのコミットが走るので、それ用に git config も設定しています。

# おわりに

開発を思い立って数日で公開できたので、かなりスピード感をもって開発できてよかったなと思いました。今回で VS Code 拡張機能開発の基礎は学べたので、今後も色々便利な拡張機能の開発や既存の拡張機能への PR など積極的に行っていきたいです。
