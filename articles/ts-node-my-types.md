---
title: "ts-nodeで@typesに配置した独自の型定義ファイルを読み込んでくれない時は.."
emoji: "🚸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "node"]
published: true
---

詰まったのでメモ。

# 起こったこと

ちょっとしたスクリプトを TypeScript で書いていて、それを実効するために`ts-node`を使ったのですが、`src/@types`においた独自の型定義ファイルをどうしても読み込んでくれず、 `Could not find a declaration file for module ...`のエラーが発生しました。

試してみたこととしては以下のとおりです。

- `tsconfig.json` の `baseUrl`、`path` で型定義ファイルのパス指定
- `tsconfig.json` の `typeRoots` への型定義ファイルのパス追加（[非推奨らしい](https://qiita.com/tetradice/items/b89a5dd41fcebf96379e)）

どれを試しても駄目でした。

# 解決策

答えはやはりドキュメントに（最初から読もう😇）。

https://github.com/TypeStrong/ts-node#help-my-types-are-missing

ts-node のデフォルトでは、指定ファイルと依存関係にないファイルは読み込んでくれないので、起動時に`--files`オプションを付与する必要がありました。

これで`src/@types`に指定した独自の型定義ファイルも読み込んでくれるようになります。解決!


```json:package.json
{
  //...
  "scripts": {
    "start": "ts-node --files src/index.ts"
  }
  //...
}
```