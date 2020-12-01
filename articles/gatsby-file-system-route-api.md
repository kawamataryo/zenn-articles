---
title: "Gatsby.jsの新機能「File System Route API」を試してみる"
emoji: "🗃"
type: "tech"
topics: ["Gatsby", "JavaScript", "React"]
published: false
---

[Jamstack Advent Calendar 2020](https://qiita.com/advent-calendar/2020/jamstack) の 1 日目の記事です。


最近 [Next.js](https://nextjs.org/) が凄い勢いで進化していますが、同じ React フレームワークの [Gatsby.js](https://www.gatsbyjs.com/) も負けず劣らず新しい機能が追加されています。
今回は、先月公開された Gatsby.js の新しい API、 [File System Route API](https://www.gatsbyjs.com/docs/file-system-route-api/) について紹介します。

この記事は以下バージョン時点の情報です。

- Gatsby.js: `2.25.4`

# File System Route APIとは？

File System Route API は、ブログの個別ページなど同レイアウトでデータが異なるページを作る際に、ファイル名を特定の表記とすることで動的にページを生成する API です。今までは `gatsby-node.js` で `createPages` で行ってたことを代替できます。

https://www.gatsbyjs.com/docs/file-system-route-api/


# 旧来の方式とFile System Route APIの比較
旧来の API と比較をしながら File System Route API の機能を解説していきます。

## 旧来の方式（createPages）の場合
最初に旧来の方式で、動的にページを生成する方法を確認します。

以下 Wordpress を HeadlessCMS としてリソースを取得する場合のコード例なのですが、
`gatsby-node.js`の`createPages`のフックにて、全ての詳細ページを取得する GraphQL クエリを発行してページデータを取得、foreach でページデータの個数だけ、`actions.createPage` を実行してページを作成するという処理を行っています。

```js:gatsby-node.js
const path = require("path");
const slash = require("slash");

exports.createPages = async ({ graphql, actions }) => {
  const result = await graphql(`
    {
      allWpPost {
        nodes {
          id
        }
      }
    }
  `);

  if (result.errors) {
    throw new Error(result.errors);
  }
  const { allWpPost } = result.data;

  const postTemplate = path.resolve("./src/templates/post.jsx");
  allWpPost.nodes.forEach(edge => {
    actions.createPage({
      path: `/${edge.id}/`,
      component: slash(postTemplate),
      context: {
        id: edge.id
      }
    });
  });
};
```

そして、`actions.createPage`でデータを挿入するテンプレートページでは、以下のように`createPage`時に渡す`context`から ID を受け取り、描画に必要なデータを取得して表示します。

```jsx:src/templates/post.jsx
import React from "react";
import {graphql} from "gatsby";

const BlogSinglePage = ({data}) => (
  <main>
    <h1>{ data.wpPost.title }</h1>
  </main>
);

export const query = graphql`
  query ($id: String) {
    wpPost(id: { eq: $id}) {
      title
    }
  }
`

export default BlogSinglePage;
```

以上が旧来の動的なページの生成方法です。

ただ動的にページを作りたいだけなのに`gatsby-node.js`に生成処理を書く必要があることのは地味に面倒ですよね。

# File System Route APIの場合

次は File System Route API の場合です。

File System Route API では `{ }`でファイル名を囲み、ファイル名に取得したいリソースを書くことで、今まで`gatsby-node.js`で行ってきた動的なページの生成処理を自動で行うことができます。

例えば、先程と同様 Wordpress の投稿データからページを作る場合は、

`{WpPost.id}.jsx`

というファイル名で`pages`ディレクトリにファイルを配置します。

```
src
└── pages
   └── {WpPost.slug}.tsx
```

こうすることで Gatsby.js のビルド時、内部的には旧来の`createPages`で読んでいたクエリと同様の以下クエリが発行されます。
そしてその結果でページが生成されます。

```graphql
allWpPost {
  nodes {
    id
  }
}
```

今まで `actions.createPage` で `context` に渡していた値と同様に、詳細ページのテンプレート（`{WpPost.id}.jsx`）でページ生成時の GraphQL の戻り値を使うことができます。
今回の例だと`{WpPost.id}.jsx`は以下のようになります。

```js:src/pages/{WpPost.id}.jsx
import React from 'react';
import {graphql, PageProps} from 'gatsby';

const BlogSinglePage: React.FC<PageProps> = ({data}) => (
  <main>
    <h1>{ data.wpPost.title }</h1>
  </main>
);

export const query = graphql`
  query ($id: String) {
    wpPost(id: { eq: $id}) {
      title
    }
  }
`

export default BlogSinglePage;
```

ファイルの配置場所が`templates`から`pages`に移動し、ファイル名が変わっただけでファイルの内容は変わりません。

ファイル名でリソースを指定するだけで、`gatsby-node.js`の処理が丸ごと不要になるため、動的にページを生成する手間がだいぶ省けますね。

# ファイル名のSyntax

File System Route API でリソース取得に使うファイル名のSyntaxのメモです。

- ファイル名の全体を`{}`で囲む
- リソース名は lowercase または uppercase とする
- `.`でつないで取得したいフィールド名を記載する
- 第一階層以外のフィールドを指定したい場合は、`__`でフィール名を繋ぐ
- [GraphQLのユニオンタイプ](https://graphql.org/learn/schema/#union-types)を使いたい場合は`()`でユニオンタイプ名を囲む

例えば、投稿の `author` の `id` を基準としてページを生成したい場合は、以下のようなファイル名とします。

`{WpPost.author__id}.jsx`

これで生成されるクエリは以下です。

```graphql
allWpPost {
  nodes {
    id # idはデフォルトで必ず取得される
    author {
      id
    }
  }
}
```

さらに深い階層にあるフィールドでも`__`で繋いでいくことで取得できます。


# File System Route API の内部実装

最後に、 File System Route API の内部実装を読んでみたのでその紹介です。
File System Route API が導入された PR はこちらです。

https://github.com/gatsbyjs/gatsby/pull/25204

ここからコードを追ってみます。

最初にみるべきファイルは`packages/gatsby-plugin-page-creator/src/create-page-wrapper.ts`です。
createPages の処理の中で、ファイル名に`{`を含むかの判定処理を追加して、含む場合は File System Route API の処理（`createPagesFromCollectionBuilder`）を実行しているようです。

https://github.com/gatsbyjs/gatsby/pull/25204/files#diff-b3684b0053156d0efb3de913e8e58a2fd32aa31972d5ecfe8475a0b082d6e250R36-R46

```ts:packages/gatsby-plugin-page-creator/src/create-page-wrapper.ts
// ...

function pathIsClientOnlyRoute(path: string): boolean {
  return path.includes(`[`)
}

// ...

export function createPage(
  filePath: string,
  pagesDirectory: string,
  actions: Actions,
  ignore: string[],
  graphql: CreatePagesArgs["graphql"]
): void {
  //...

  // If the page includes a `{}` in it, then we create it as a collection builder
  if (pathIsCollectionBuilder(absolutePath)) {
    trackFeatureIsUsed(`UnifiedRoutes:collection-page-builder`)
    if (!process.env.GATSBY_EXPERIMENTAL_ROUTING_APIS) {
      throw new Error(
        `PageCreator: Found a collection route, but the proper env was not set to enable this experimental feature. Please run again with \`GATSBY_EXPERIMENTAL_ROUTING_APIS=1\` to enable.`
      )
    }
    createPagesFromCollectionBuilder(filePath, absolutePath, actions, graphql)
    return
  }

  //...
}
```

`createPagesFromCollectionBuilder` は、`create-pages-from-collection-builder.ts`にあり、以下処理を行ってるようです。

1. `collectionExtractQueryString`でパスから GraphQL のクエリー文字列を作成
2. 作成したクエリー文字列から GraphQL リクエストを実行
3. レスポンスの node を検証
4. node からループで `actions.createPage` を実行してページを作成

https://github.com/gatsbyjs/gatsby/pull/25204/files#diff-cfc298fefff4ec4b5edda6360c015d8a74a5113a3c0fabdc2b7a7ce75e76d93bR14-R58

```ts:packages/gatsby-plugin-page-creator/src/create-pages-from-collection-builder.ts
export async function createPagesFromCollectionBuilder(
  filePath: string,
  absolutePath: string,
  actions: Actions,
  graphql: CreatePagesArgs["graphql"]
): Promise<void> {

  // ...

  // 1. Query for the data for the collection to generate pages
  const queryString = collectionExtractQueryString(absolutePath)

  // ...

  const { data, errors } = await graphql<{ nodes: Record<string, unknown> }>(
    queryString
  )

  // ...

  // 2. Get the nodes out of the data. We very much expect data to come back in a known shape:
  //    data = { [key: string]: { nodes: Array<ACTUAL_DATA> } }
  const nodes = (Object.values(Object.values(data)[0])[0] as any) as Array<
    Record<string, object>
  >

  if (nodes) {
    reporter.info(
      `   Creating ${nodes.length} page${
        nodes.length > 1 ? `s` : ``
      } from ${filePath}`
    )
  }

  // 3. Loop through each node and create the page, also save the path it creates to pass to the watcher
  //    the watcher will use this data to delete the pages if the query changes significantly.
  const paths = nodes.map((node: Record<string, object>) => {
    // URL path for the component and node
    const path = createPath(derivePath(filePath, node))
    // Params is supplied to the FE component on props.params
    const params = getCollectionRouteParams(createPath(filePath), path)
    // nodeParams is fed to the graphql query for the component
    const nodeParams = reverseLookupParams(node, absolutePath)
    // matchPath is an optional value. It's used if someone does a path like `{foo}/[bar].js`
    const matchPath = getMatchPath(path)

    actions.createPage({
      path: path,
      matchPath,
      component: absolutePath,
      context: {
        ...nodeParams,
        __params: params,
      },
    })

    return path
  })

  // ...
```

実際の GraphQL クエリーの文字列を組み立ては`collectionExtractQueryString`から呼ばれている`generateQueryFromString` で行われます。
`generateQueryFromString`のテストをみることで、どのようなファイル名から、どのようなクエリー文字列が生成されるのか分かりそうです。

https://github.com/gatsbyjs/gatsby/blob/d305ee57a58c9d8bdf44e2084ea3e972925b9cb5/packages/gatsby-plugin-page-creator/src/__tests__/extract-query.ts

```ts:packages/gatsby-plugin-page-creator/src/__tests__/extract-query.ts
// ...
describe(`extract query`, () => {
  describe(`root query string`, () => {
    it(`basic example`, () => {
      expect(
        generateQueryFromString(
          `Thing`,
          compatiblePath(`/foo/bar/{Thing.id}.js`)
        )
      ).toBe(`{allThing{nodes{id}}}`)
    })

    it(`works with different file extsnions`, () => {
      expect(
        generateQueryFromString(
          `Thing`,
          compatiblePath(`/foo/bar/{Thing.id}.tsx`)
        )
      ).toBe(`{allThing{nodes{id}}}`)
    })
  // ...
  })
})
// ...
```

# 終わりに

以上、「Gatsby.js の新機能 File System Route API を試してみた」でした。
Nuxt.js や、Next.js と同じようにファイル名を変えることで動的なページの生成が出来るのは便利ですね。

今回紹介できなかったのですが、Gatsby.js を クライアント再度でページを組み立てる Client-only-route のページ生成にも`File System Route API`は対応しています。

https://www.gatsbyjs.com/docs/file-system-route-api/#creating-client-only-routes

Gatsby.js の今後の進化に期待です。

# 参考
- [File System Route API | Gatsby](https://www.gatsbyjs.com/docs/file-system-route-api/)
