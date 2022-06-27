---
title: "はじめてのyargs - Node.jsでコマンドライン引数をパースする -"
emoji: "🤠"
type: "tech"
topics: ["yargs", "typescript", "cli"]
published: false
---

最近yargsでいくつかCLIツールを作ったので備忘録としてまとめます。

# yargsとは？

Node.jsにてコマンドライン引数をパースするライブラリです。
自作のCLIツールの作成時などに利用されます。

主な機能はこちらです。

* コマンドライン引数のパース
* ヘルプやバージョンの生成
* bash補完スクリプトの生成

https://github.com/yargs/yargs

## インストール

npmからインストールできます。TypeScriptの型定義はDefinitelyTypedで提供さてています。

```bash
npm i yargs
```

```bash
npm i -D @types/yargs
```

## Get started

# 逆引きyargs

yargsの使い方を逆引き風に紹介します。


## オプション引数をパースする

オプション引数は `yargs`

## ポジショナル引数をパースする
## 必須引数にする
## 引数の値をチェックする
## ヘルプ・バージョンを表示する
## 任意の引数をパースする
## サブコマンドを作る