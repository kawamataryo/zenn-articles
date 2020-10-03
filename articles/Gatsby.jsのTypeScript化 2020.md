---
title: "Gatsby.jsのTypeScript化 2020"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatsby.js", "TypeScript", "JavaScript", "React"]
published: false
---

ちょっと個人でGatsby.jsを使い始めたのですが、Gatsby.jsのTypeScript化についての記事の情報が少し古かったので、自分で書き直してみました。
スターターパッケージをTypeScript化していきます。

なお今回のコードは全て下記リポジトリにあります。
もし、何かエラーになった際は参照してみてください。

# 0. プロジェクトの作成

gatsbyコマンドを使うめに`gatsby-cli`の追加。

```bash
npm install -g gatsby-cli
```

一番標準のblogのスターターでプロジェクトを作成します。

```bash
gatsby new gatsby-typscript-sample https://github.com/gatsbyjs/gatsby-starter-blog
cd gatsby-typscript-sample
```

ここからこのプロジェクトをTypeScript化していきます。

# 1. tsconfig.jsonの追加

最初に、`tsconfig.json`を追加します。

```
npx tsc --init
```

`tsconfig.json`は以下のようにして設定します。

```
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

これで以降、エディタ上でTSの型エラーを適切に認識できるようになります。

# GraphQLリクエストのTS化

Gatsbyはresourcesに対してGraphQLでリクエストを送り、データを取得します。
そのGraphQLリクエストのレスポンスの型を、[gatsby-plugin-typegen](https://github.com/cometkim/gatsby-plugin-typegen)を使い生成します。

:::message
GraphQLの型の生成には[gatsby-plugin-graphql-codegen](https://github.com/d4rekanguok/gatsby-typescript)を使っている記事が多いのですが、gatsby-plugin-graphql-codegenのリポジトリにてgatsby-plugin-typegenを使ってねと紹介されていたので、今回はtypegenの方を使っています。
:::

```bash
yarn add gatsby-plugin-typegen
```

gatsby-config.jsのpluginsに`gatsby-plugin-typegen`を追記します。

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

次に、各コンポーネントのqueryにクエリ名を追加していきます。
この変更をすることでそのクエリ専用の型が生成されます。

（例: src/components/index.js)

```js:src/components/index.js
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
ここにGraphQLリクエストの型定義があります。
先ほど追加したBlogIndexクエリの型を見てみると、、

```ts:src/__generated__/gatsby-types.ts
//...
type BlogIndexQueryVariables = Exact<{ [key: string]: never; }>;


type BlogIndexQuery = { readonly site: Maybe<{ readonly siteMetadata: Maybe<Pick<SiteSiteMetadata, 'title'>> }>, readonly allMarkdownRemark: { readonly nodes: ReadonlyArray<(
      Pick<MarkdownRemark, 'excerpt'>
      & { readonly fields: Maybe<Pick<Fields, 'slug'>>, readonly frontmatter: Maybe<Pick<Frontmatter, 'date' | 'title' | 'description'>> }
    )> } };
//...
```

ちゃんと生成されてますね!最高便利。

# 各コンポーネントファイルのTS化

さてこれで準備ができたので、各ファイルをTS化していきます。
ここで他の記事では、[gatsby-plugin-typescript](https://github.com/gatsbyjs/gatsby/tree/master/packages/gatsby-plugin-typescript)の追加から入る記事が多いのですが、2020年10月現在、Gatsbyには`gatsby-plugin-typescript`が既に組み込まれているので、何もせずで大丈夫です。


:::message
何かTypeScriptのビルド関連で追加の設定をしたい場合は、gatsby-config.jsのpluginsで`gatsby-plugin-typescript`を追加して、optionを設定してください。
:::

各コンポーネントのファイル拡張子を`.js`から`.tsx`に書き換えましょう。
そして、StaticQueryの戻り値など型エラーとなっている箇所に型をつけていきます。

例えば、`src/pages/index.ts`の型付けは以下のようになります。

```ts:src/pages/index.ts
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
あとは、適宜Optional Chainingや、Non null Assertionを使って型エラーを解決しましょう。


# gatsby-config.jsのTS化

# gatsby-node.jsのTS化

# 参考

- [Gatsby.js を完全TypeScript化する - Qiita](https://qiita.com/Takepepe/items/144209f860fbe4d5e9bb)
- [TypeScript + Gatsby config and node API](https://gist.github.com/JohnAlbin/2fc05966624dffb20f4b06b4305280f9)