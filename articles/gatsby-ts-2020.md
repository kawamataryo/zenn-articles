---
title: "Gatsby.jsのTypeScript化 2020"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatsby", "TypeScript", "JavaScript", "React"]
published: true
---
![](https://storage.googleapis.com/zenn-user-upload/r219yhkkbumg7bc434l33daob216)

個人で Gatsby.js を使い始めたのですが、検索して見つかる Gatsby.js の TypeScript 化についての記事の情報が少し古かったので、自分で書き直してみました。
順を追ってスターターパッケージを TypeScript 化していきます。

なお今回のコードは全て下記リポジトリにあります。
もし、何かエラーになった際は参照してみてください。

https://github.com/kawamataryo/gatsby-typescript-sample

この記事は以下のバージョン時点の情報です。

* `Gatsby.js`: 2.24.66
* `gatsby-plugin-typegen`: 2.2.1


# 0. プロジェクトの作成

gatsby コマンドを使うめに`gatsby-cli`の追加。

```bash
npm install -g gatsby-cli
```

一番標準の blog のスターターでプロジェクトを作成します。

```bash
gatsby new gatsby-typscript-sample https://github.com/gatsbyjs/gatsby-starter-blog
```

ここからこのプロジェクトを TypeScript 化していきます。

# 1. tsconfig.jsonの追加

最初に、`tsconfig.json`の追加と`typescript`のインストールをします。

```
npx tsc --init
yarn add typescript -D
```

`tsconfig.json`は以下のようにして設定します。

```json
{
  "include": ["./src/**/*"],
  "compilerOptions": {
    "target": "esnext",
    "module": "esnext",
    "lib": ["dom", "es2017"],
    "jsx": "react",
    "strict": true,
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "noEmit": true,
    "skipLibCheck": true,
    "moduleResolution": "node"
  }
}
```

そして、`packege.json`に TypeScript の型チェック用の npm スクリプトを追加します。
すでに`tsconfig.json`側で`noEmit: true`を設定しているので`tsc`コマンドで JS コードは生成されません。

```json
{
  // ...
  "scripts": {
    // ...
    "typecheck": "tsc",
  }
}
```

これで以降、`yarn typecheck`で TS の型チェックを行えるようになります。

# 2. GraphQL Schema, リクエストの型生成

Gatsby はリソースに対して GraphQL でリクエストを送り、データを取得します。
その GraphQL リクエストのレスポンスの型を、[gatsby-plugin-typegen](https://github.com/cometkim/gatsby-plugin-typegen)を使い生成します。

:::message
GraphQL の型の生成には[gatsby-plugin-graphql-codegen](https://github.com/d4rekanguok/gatsby-typescript)を使っている記事が多いのですが、gatsby-plugin-graphql-codegen のリポジトリにて gatsby-plugin-typegen を使ってねと紹介されていたので、今回は typegen の方を使っています。
:::

```bash
yarn add gatsby-plugin-typegen
```

`gatsby-config.js`の plugins に`gatsby-plugin-typegen`を追記します。

[🔗 src/components/index.ts](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/src/src/gatsby-config.js)

```js
module.exports = {
  siteMetadata: {
    // ...
  },
  plugins: [
    // ...
    `gatsby-plugin-typegen`
  ],
}
```

次に、各コンポーネントの query にクエリ名を追加していきます。
この変更をすることでそのクエリ専用の型が生成されます。

（例： src/components/index.js `query BlogIndex`の部分を追記している）

[🔗 src/components/index.ts](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/src/src/components/index.ts)

```js
//...
export const pageQuery = graphql`
  query BlogIndex {
    site {
      siteMetadata {
        title
      }
    }
    allMarkdownRemark(sort: { fields: [frontmatter___date], order: DESC }) {
      nodes {
        excerpt
        fields {
          slug
        }
        frontmatter {
          date(formatString: "MMMM DD, YYYY")
          title
          description
        }
      }
    }
  }
`
```

最後に`yarn build`を実行すると、`src/__generated__/gatsby-types.ts`が生成されているはずです。
ここに GraphQL リクエストの型定義があります。
先ほど追加した BlogIndex クエリの型を見てみると、、

[🔗 src/pages/index.ts](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/src/src/__generated__/gatsby-types.ts)

```ts
//...
type BlogIndexQueryVariables = Exact<{ [key: string]: never; }>;


type BlogIndexQuery = { readonly site: Maybe<{ readonly siteMetadata: Maybe<Pick<SiteSiteMetadata, 'title'>> }>, readonly allMarkdownRemark: { readonly nodes: ReadonlyArray<(
      Pick<MarkdownRemark, 'excerpt'>
      & { readonly fields: Maybe<Pick<Fields, 'slug'>>, readonly frontmatter: Maybe<Pick<Frontmatter, 'date' | 'title' | 'description'>> }
    )> } };
//...
```

ちゃんと生成されてますね！　最高便利。

# 3. 各コンポーネントファイルのTypeScript化

これで準備ができたので、各ファイルを TypeScript 化していきます。
[gatsby-plugin-typescript](https://github.com/gatsbyjs/gatsby/tree/master/packages/gatsby-plugin-typescript)の追加から入る記事が多いのですが、2020 年 10 月現在、Gatsby には`gatsby-plugin-typescript`がすでに組み込まれているので、何もせずで大丈夫です。


:::message
何か TypeScript のビルド関連で追加の設定をしたい場合は、gatsby-config.js の plugins で`gatsby-plugin-typescript`を追加して、option を設定してください。
:::

各コンポーネントのファイル拡張子を`.js`から`.tsx`に書き換えましょう。
そして、StaticQuery の戻り値など型エラーとなっている箇所に型をつけていきます。

例えば、`src/pages/index.ts`の型付けは以下のようになります。

[🔗 src/pages/index.ts](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/src/pages/index.ts)

```ts
import React from "react"
import { Link, graphql } from "gatsby"
import { PageProps } from "gatsby"

import Bio from "../components/bio"
import Layout from "../components/layout"
import SEO from "../components/seo"

const BlogIndex:React.FC<PageProps<GatsbyTypes.BlogIndexQuery>> = ({ data, location }) => {
  const siteTitle = data.site?.siteMetadata?.title || `Title`
  const posts = data.allMarkdownRemark.nodes

  // ... 以下略
}
```

ポイントは以下のように`React.FC`、`PageProps`などのジェネリクス型を使うことと、`gatsby-plugin-typegen`で生成した型を使うことです。

```ts
const BlogIndex:React.FC<PageProps<GatsbyTypes.BlogIndexQuery>> = ({ data, location }) => { /* -- */ }
```

これで`data`の型が`BlogIndexQuery`の型で推論されます。
あとは、適宜 Optional Chaining や、Non null Assertion を使って型エラーを解決しましょう。


# 4. gatsby-Node.jsのTypeScript化

`gatsby-node.js`でも TypeScrip で書けるようにしていきます。ここでは[ts-node][https://github.com/TypeStrong/ts-node]を追加ます。

ここの書き方は[@Takepepe](https://twitter.com/takepepe?lang=en)さんの以下の記事を参考にさせていただきました。良記事ありがとうございます🙏
[Gatsby.js を完全TypeScript化する - Qiita](https://qiita.com/Takepepe/items/144209f860fbe4d5e9bb)

```
yarn add -D ts-node
```

そして、`gatsby-config.js`を以下のように変更します。

[🔗 gatsby-config.js](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/gatsby-config.js)

```js
"use strict"

require("ts-node").register({
  compilerOptions: {
    module: "commonjs",
    target: "esnext",
  },
})

require("./src/__generated__/gatsby-types")

const {
  createPages,
  onCreateNode,
  createSchemaCustomization,
} = require("./src/gatsby-node/index")

exports.createPages = createPages
exports.onCreateNode = onCreateNode
exports.createSchemaCustomization = createSchemaCustomization
```


そして、今まで`gatsby-node.js`に記述していた内容を`src/gatsby-node/index.ts`に移動して、型を設定します。
基本的に note の API は`GatsbyNode`から型を取得できます。

:::message
本当は、`allMarkdownRemark`のクエリ部分の方も`gatsby-plugin-typegen`で生成したかったのですが、上手く認識してくれませでした。やり方わかる方いたら教えてください🙏
（ドキュメントの` Provides utility types for gatsby-node.js.`はまだチェックがついていないので、まだ未対応なのかな？）。
:::


[🔗 src/gatsb-node/index.ts](https://github.com/kawamataryo/gatsby-typescript-sample/blob/master/src/gatsby-node/index.ts)

```ts
import path from "path"
import { GatsbyNode, Actions } from "gatsby"
import { createFilePath } from "gatsby-source-filesystem"

export const createPages: GatsbyNode["createPages"] = async ({ graphql, actions, reporter }) => {
  const { createPage } = actions

  const blogPost = path.resolve(`./src/templates/blog-post.js`)

  const result = await graphql<{ allMarkdownRemark: Pick<GatsbyTypes.Query["allMarkdownRemark"], 'nodes'> }>(
    `
      {
        allMarkdownRemark(
          sort: { fields: [frontmatter___date], order: DESC }
          limit: 1000
        ) {
          nodes {
            fields {
              slug
            }
            frontmatter {
              title
            }
          }
        }
      }
    `
  )

  //...

  }
}

export const onCreateNode: GatsbyNode["onCreateNode"] = ({ node, actions, getNode }) => {
  const { createNodeField } = actions
  //...
}

export const createSchemaCustomization: GatsbyNode["createSchemaCustomization"] = async ({ actions }: { actions: Actions}) => {
  const { createTypes } = actions
  // ...
}

```

これで`gatsby-Node.js`の TypeScript 化も完了です🎉

# 終わりに

以上、「Gatsby.js の TypeScript 化 2020」でした。
まだまだ Gatsby.js も React も使い始めたばかりなので、もっと良いやり方があれば気軽にコメントを頂けると嬉しいです！


# 参考

- [Gatsby.js を完全TypeScript化する - Qiita](https://qiita.com/Takepepe/items/144209f860fbe4d5e9bb)
- [TypeScript + Gatsby config and node API](https://gist.github.com/JohnAlbin/2fc05966624dffb20f4b06b4305280f9)