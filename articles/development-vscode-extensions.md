---
title: "Python の dotted path をコピーする VS Code 拡張機能を作ってみた"
emoji: "🫐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "python", "typescript"]
published: false
---

はじめて VS Code 拡張機能を公開してみたのでまとめました。

# 作ったもの

Copy Python Path というPythonのdotted pathをコピーする拡張機能を作りました。

業務で Django を使っていて、メソッド単位で unittest を実行する際に、dotted path を利用するのですが、それを毎回手動で組み立てるのが面倒でした。既存でもdotted pathをコピーする拡張機能はあるのですが、どれもファイル単位のパスコピーでイマイチかゆいところに手が届かなかったので、作って見ました。

機能はこちらです。

* クラス or メソッドの定義行でコマンドランチャー又はコンテキストメニューからコマンドを実行すると、そのクラス or メソッドまでの dotted path をクリップボードにコピーする
* クラス or メソッドの定義行以外でコマンドランチャー又はコンテキストメニューからコマンドを実行すると、そのファイルまでの dotted path をクリップボードにコピーする

![](https://i.gyazo.com/fe88befdaea034eff0adfd4caacd028f.gif)

こちらからインストールできます。レビューを頂けると泣いて喜びます。

https://marketplace.visualstudio.com/items?itemName=kawamataryo.copy-python-dotted-path

またコードもこちらで公開しています。
https://github.com/kawamataryo/copy-python-path

# 仕組み & 工夫したところ

## JS で Python コードを AST にパースしてメソッド・クラス定義を解析
ファイルまでの相対パスだけならば VS Code のAPIから取得できるのですが、クラス・メソッドの定義までのパスとなると、VS Code のAPIから取得することはできません。

正確にクラス・メソッド名、定義位置（階層構造）を取得するためには Python コードをパースする必要がありました。

そこで今回は、 [ANTLR4](https://github.com/antlr/antlr4) [dt-python-parser] ベースのパーサージェネレーターであるdt-python-parser を使ってみました。

https://www.npmjs.com/package/dt-python-parser

このライブラリにPythonコードを与えると AST（抽象構文木）にパースしてくれ、さらに ANTLR4 ベースのメソッドで、AST をトラバースできます。

dt-python-parser を使った copy-python-path の解析部分のコードはこちらです。

https://github.com/kawamataryo/copy-python-path/blob/main/src/utils/getRelatedDefinedSymbols.ts
```ts:src/utils/getDefinedParentSymbols.ts
import { Python3Parser, Python3Listener } from 'dt-python-parser';

const getDefinedParentSymbols = (symbol: DefinedSymbol, symbols: DefinedSymbol[], result: DefinedSymbol[] = []): DefinedSymbol[] => {
	const parentSymbol = symbols.filter(s => s.column < symbol.column).sort((l, r) => {
		const lDistance = symbol.line - l.line;
		const rDistance = symbol.line - r.line;
		if (lDistance > 0 && lDistance < rDistance) {
			return -1;
		} else {
			return 1;
		}
	})[0];

	if (parentSymbol.column === 0) {
		return [parentSymbol, ...result];
	}

	return getDefinedParentSymbols(parentSymbol, symbols, [parentSymbol, ...result]);
};

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

dt-parserでクラス名・メソッド名と、その定義位置を取得し、再帰処理でコードの構造に合わせて並び替えた配列を作っています。

## E2Eテストで動作を担保
仕組みは簡単な拡張機能ですが、継続してメンテナンスしやすいようにE2Eテストを追加してみました。

以下の画像のようにテストを実行すると、VS Code が別プロセスで立ち上がり実際コマンドを実行してくれます。

実は VS Code 拡張機能のテストの方法は、ほぼほぼ情報がなくとても苦労しました。

実際のテスト APIの使い方などは、ほぼほぼ情報がありませんでした。VS Codeのドキュメントにも詳細な記載はなしです。
Microsoftの出している拡張機能の実際のテストコードをみることが一番良いと思います。

## JS で Python コードを AST にパースしてメソッド・クラス定義を解析

# CI

CI 

:::message
実際に自分の作っていた拡張機能をGitHub Actionsで動かした際に、Windowsのビルドだけテストが落ちて、プラットフォーム依存のバグがあることに気付けました。テストはやはり大事。
:::

# 公開