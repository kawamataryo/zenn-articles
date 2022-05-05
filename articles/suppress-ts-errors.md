---
title: "TypeScriptで'strict: true'にしたい全ての人へ〜suppress-ts-errorsの紹介"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "compilerApi", "ts-morph", "cli"]
published: false
---

TypeScript の型チェックを厳格化したいが既存の型エラーが多すぎてやむなく緩い型チェックにしているプロジェクトにて、改善の助けになりそうなツールを作ったので紹介です。

# 🔧 作ったもの

[suppress-ts-errors](https://github.com/kawamataryo/suppress-ts-errors)というプロジェクトのコードベースを走査して、型エラーがあるコードすべてに自動的に、`@ts-expect-error`または`@ts-ignore`のコメントを追加する CLI ツールを作りました。

以下の Gif のように npx 経由で簡単に利用できます。

さらに、`.ts`だけでなく`.tsx`と`.vue`ファイルのスクリプト部分へのコメント追加にも対応しています。

コードもこちらで公開しています。

https://github.com/kawamataryo/suppress-ts-errors

## なぜ作った？

現職のプロダクトにて型チェックを厳格化できていない（`strict: true`にできない）という状況を改善したいと思ったからです。

一般的に型チェックを厳格化するには「既存の型エラーをすべて直し、その上で型チェックを厳格化する」だと思いますが、現職のプロダクトはもともと JS で書かれていたということもあり既存のエラーがあまりにも多く、すべてを解消するには多くの工数が必要でなかなか着手に踏み切れない状況でした。

その間にも、新機能はどんどん開発されていき、新たな型エラーが生まれやすい状況でした。それを解消するために、まず先に型チェックを厳格化して、新規追加のコードは型に守られた状態としたいと考えました。

そのためには、既存の膨大な型エラーすべてに型エラーを無効化するコメント（`@ts-expect-error`又は`@ts-ignore`）を追加する必要があり、それを自動化するためにこの CLI ツールを開発しました。

この CLI ツールを作るにあたっては以下記事から着想・実装のヒントを頂きました。感謝 🙏

https://mizchi.dev/202006232052-rewrite-to-ts-with-expect-error

https://blog.nnn.dev/entry/2022/03/10/110000

# 🚀 使い方

## `ts`、`tsx`ファイルへの適応

TypeScript のプロジェクトにて、tsconfig.json の型チェックのルールを強化し（`strict: true`にするなど）、型エラーが出ている状態で以下コマンドを実行するだけです。

```bash
$ npx suppress-ts-errors
```

これだけで、型エラーが出ている箇所に`@ts-expect-error`が追加されて、型エラーを無効化できます。

## `vue`ファイルへの適応

Vue の SFC の script 部分の型エラーについても、`@ts-expect-error`を追加できます。
その場合は、サブコマンドの`vue`にて、vue ファイルの glob パターンを指定して実行します。

```bash
$ npx suppress-ts-errors vue "src/**/*.vue"
```

## オプション

オプションでコメントの種類やエラーコードの追加有無も制御できます。

| option              | default           | description                                                                                                                                                                                                                                                                                                                   |
| ------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -t, --tsconfig-path | `./tsconfig.json` | tsconfig.json へのパスの指定                                                                                                                                                                                                                                                                                                  |
| -c, --comment-type  | `1`               | 追加するコメントの種類。 <br> `1` は [@ts-expect-error](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-9.html#-ts-expect-error-comments), `2` は [@ts-ignore](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-6.html#suppress-errors-in-ts-files-using--ts-ignore-comments). |
| -e, --error-code    | `true`            | コメント末尾にエラーコードを追加するかどうか（例: TS2345）                                                                                                                                                                                                                                                                    |

# 🦾 仕組み & 工夫したところ

## ts-morph での TypeScript Compiler API の操作

型エラーの判定やコメントの挿入は、TypeScript Compiler API で行っています。そのまま Compiler API を使うでも良かったのですが、のですが、[ts-morph](https://github.com/dsherret/ts-morph) という Compiler API を使いやすくする API を提供しているラッパーライブラリがあったので、そちらを使いました。

https://github.com/dsherret/ts-morph

コアのロジック部分では ts-morph の`getPreEmitDiagnostics`を使って、ファイル単位で型エラーをチェックしています。

https://github.com/kawamataryo/suppress-ts-errors/blob/main/src/lib/suppressTsErrors.ts#L4-L59

コメント付きコードへの書き換えも、ts-morph の`replaceWithText`と`saveSync`を利用しています。

https://github.com/kawamataryo/suppress-ts-errors/blob/main/src/handlers/tsHandler.ts#L5-L37

ts-morph を使ったおかげで、簡単に見通しよく Compiler API を使うことができたので良かったです。

## Vue の SFC への対応

現職のプロダクトが Vue の SFC を使って書かれており、その SFC 内の Script でも大量の型エラーが発生しているという状況があったので、そちらも改善するためにも Vue に対応する必要がありました。
TypeScript Compiler API に直接`.vue`ファイルの中身を追加することは出来ないので以下手順で対応しています。

1. 検査対象の Vue ファイル一覧を取得
2. 個々の Vue ファイルから`<script lang=“ts”>`の script 部分を抽出
3. script 部分を TypeScript Compiler API の Project にセット
4. 型チェックを行い、コメント挿入したコードを作る
5. コメント挿入したコードで、対象の Vue ファイルの script 部分を書き換え保存

以下が Vue ファイルの処理部分です。

https://github.com/kawamataryo/suppress-ts-errors/blob/main/src/handlers/vueHandler.ts#L9-L69

ポイントは一度すべての`vue`ファイル内部の TypeScript を Compiler API の Project に追加してから、個々のファイルの型チェックを行うことです。
そうしなければ、それぞれの`vue`ファイルの import を解決できず不要なエラーが認識されてしまいます。

## Vitest での Unit Test

TypeScript Compiler API はドキュメントもほぼなく、型定義から API を読み取り開発する必要があったため、効率よくコーディング・動作確認のサイクルを回す必要がありました。
そのため、まずテストを書きその後実装を書いていくという TDD 風に開発を行ったのですが、その際のテストツールとして、最近話題の[Vitest](https://vitest.dev/)を使ってみました。

https://vitest.dev/

Vitest は[Vite](https://ja.vitejs.dev/)以外のプロジェクトでなくても、単体で使うことができます。

以下実際のテストコードです。Jest 同様 Parameterize なテストも書けるので、効率的にパターンを網羅することができました。
TypeScript の設定等も不要で即利用できる点、高速に動作する点がとても良かったです。

https://github.com/kawamataryo/suppress-ts-errors/blob/main/src/lib/__tests__/suppressTsErrors.test.ts#L5-L182

# おわりに

以上、[suppress-ts-errors](https://github.com/kawamataryo/suppress-ts-errors) の紹介でした。TypeScript Compiler API へのチャレンジや、Vitest の導入などもできて作っていてとても楽しいツール開発でした。業務でも即生かすことができて満足です。

自分と同じように、TypeScript の型厳格化に悩む人の助けになれば嬉しいです。
