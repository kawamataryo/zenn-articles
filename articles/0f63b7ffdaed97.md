---
title: "ts-jestでcannot be compiled under '--isolatedModules'と出た時の対処法"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jest", "TypeScript"]
published: true
---

`ts-jest`で`isolateModules` のエラーでハマったのでメモ。

# 起こったこと

`npx create-next-app --typescript`で Next.js のプロジェクトを作り、その後 [ts-jest](https://github.com/kulshekhar/ts-jest) の README の通りテストのセットアップを行い、Jest の動作確認のために以下テストを作成しました。

```ts:sample.spec.ts
describe("membersPosts", () => {
  test('sample', () => {
    expect(1 + 2).toBe(3)
  })
})
```

1 + 2 は 3 になるという Jest の動作を確認するためだけの自明なテストです。
しかし、絶対通るだろうと思って実行したら型エラーで落ちました。

```
src/builder/__tests__/sample.spec.ts:1:1 - error TS1208: 'membersPosts.spec.ts' cannot be compiled under '--isolatedModules' because it is considered a global script file. Add an import, export, or an empty 'export {}' statement to make it a module.
```

当時の状況・自分の焦り用は以下 YouTube で全世界に公開されています😇

https://youtu.be/lI8CjHspGFU?t=1892

# 原因

まず Error の直接的な原因は、`create-next-app`で作られた`tsconfig.json`で`isolateModules: true`が設定されていることです。

```json:tsconfig.json
{
  "compilerOptions": {
    // ...
    "isolatedModules": true,
  },
}
```

`isolateModules` フラグは単一ファイルのトランスパイルで正しく型解釈できない特定のコードを書いた場合にコンパイルエラーを出すフラグです。
デフォルトは`false`となっています。

フラグを`true`にしている場合にコンパイルエラーとなるコードは以下のようなものがあります。

- 型のみの再エクスポートファイル
- export も import もしていないファイル
- const enum メンバーへの参照をしているファイル


https://www.typescriptlang.org/tsconfig#isolatedModules

# 対処法

対処法としては以下のようなものがあります。

## Bad 👎

最初に、微妙な対処法としては Error メッセージの`Add an import, export, or an empty 'export {}' statement to make it a module`というコメントの通り、テストファイルに`export {}`を追記するというものです。

```diff ts
describe("membersPosts", () => {
  test('sample', () => {
    expect(1 + 2).toBe(3)
  })
})

+ export {}
```

これでこのファイルは export を含むのでモジュールとして解釈され `isolateModules` のエラーは回避できます。
実際に前述の動画の配信では、暫定的にこちらで対処しました。ですが、実質不要な `export{}` がテストファイルに含まれるのは非常に微妙ですよね。。

## Good 👍

テストでは不要な `isolateModules` のフラグが ON になっていることが原因なので、あのテストを通すための正しい対処としては、テスト上では `isolateModules` を `false` に戻すことです。
それはテスト用の `tsconfig.json` を作成し、`ts-jest`に読み込ませることで実現できます。

まず最初にテスト用の `tsconfig.test.json` を作成します。初期作成された`tsconfig.json`を継承して、`isolateModules`のフラグのみを `false` で上書きしています。

```ts
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "isolatedModules": false
  }
}
```

そしてこれをテストで読み込まれるように`jest.config.js`で読み込まれるように修正します。

```ts:jest.config.js
/** @type {import('ts-jest/dist/types').InitialOptionsTsJest} */
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  globals: {
    "ts-jest": {
      tsconfig: './tsconfig.test.json'
    }
  }
};
```

これでテストでは`tsconfig.test.json`が読み込まれるようになるので、`isolateModules`のエラーは回避できます。

:::messages
配信時にもこれを試していたのですが、`"ts-jest"`と書くところを`tsJest`としてしまい反映されず焦ってました
:::

他にもテストのディレクトリが決まっていて、そこにテストファイルを配置していくというルールがあるのならば、テスト用の`tsconfig.json`をそのディレクトリに配置することでも回避可能です。


```
__tests__/
├── tsconfig.json
└── sample.spec.ts
```



## Very Good 🌟

実は、そもそもの本質的な回避方法は何もしないことです。

以下の Stack Over Flow のコメントの通り本来テストは別のモジュールを import して、そのモジュールをテストするはずなので、実は通常のテストファイルは自動的に`isolateModules`のエラーを回避できます。

> You do not have any import statements in your code. This basically means you are not testing anything outside of the test file.
> If you test something that is not in the test code (and therefore import something), the test file will become a module and the error will go away 🌹

https://stackoverflow.com/questions/57860261/isolatedmodules-error-on-jest-test-with-create-react-app-and-typescript

なので実際のプロダクト開発では、Good で上げた方法は対処する必要がありません。
Jest のテストを早まってしまったのが間違いでした。


# おわりに

以上、`cannot be compiled under '--isolatedModules'` の対処法でした。配信でエラーでると焦りますね。良い勉強になりました。
