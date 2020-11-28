---
title: "Gatsby.jsの新機能「File System Route API」を試してみた"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatsby", "JavaScript", "React"]
published: false
---

[Jamstack Advent Calendar 2020 - Qiita](https://qiita.com/advent-calendar/2020/jamstack) の 1 日目の記事です。

https://qiita.com/advent-calendar/2020/jamstack

最近 [Next.js](https://nextjs.org/) が凄い勢いで進化していますが、同じ React エコシステムの[Gatsby.js](https://www.gatsbyjs.com/) も負けず劣らず新しい機能や API が公開されています。

今回は先月公開された Gatsby.js の新しい API、 [File System Route API](https://www.gatsbyjs.com/docs/file-system-route-api/) について紹介します。

# File System Route APIとは？

File System Route API は、ブログの個別ページなど同レイアウトだけどデータが異なるページを作る際に、ファイルの名前に特定の表記をつけることで動的にページを生成する API です。今までは `gatsby-node.js` で `createPages` で行ってたことを代替できます。

https://www.gatsbyjs.com/docs/file-system-route-api/


# 旧来の方式とFile System Route APIの比較
旧来の API と比較をしながら File System Route API の機能を解説していきます。

## 旧来の方式（createPages）の場合
最初に旧来の方式で、動的にページを生成する方法を確認します。

以下 Wordpress を HeadlessCMS として使う場合のコード例なのですが、
`gatsby-node.js`の`createPages`のフックにて、全ての詳細ページを取得する GraphQL クエリを発行してページデータを取得、foreach でページデータの個数だけ、 `createPage` を実行するという処理が必要です。

```js:gatsby-node.js
const path = require("path");
const slash = require("slash");

exports.createPages = async ({ graphql, actions }) => {
  const { createPage } = actions;

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
    createPage({
      path: `/${edge.id}/`,
      component: slash(postTemplate),
      context: {
        id: edge.id
      }
    });
  });
};
```

そして、`createPage`でデータを挿入するテンプレートページでは、以下のように`createPage`時に渡す`context`から ID を受け取り、描画に必要なデータを取得して表示します。

```jsx:templates/post.jsx
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

例えば、先程と同様 Wordpress の投稿データからページを作る場合は

`{WpPost.id}.jsx`

というファイル名で`pages`ディレクトリにファイルを配置します。
こうすることで Gatsby.js のビルド時に内部的には旧来の`createPages`で読んでいたクエリと同様の以下クエリが発行されます。

```graphql
allWpPost {
  nodes {
    id
  }
}
```

そしてその結果で動的にページを生成し、GraphQL クエリの戻り値を今まで `createPage` の `context` で渡していた値と同様に、詳細ページのテンプレート（`{WpPost.id}.jsx`）で使うことができます。

今回の例だと`{WpPost.id}.jsx`は以下のようなコードとなります。
ファイルの配置場所が、`templates`から`pages`に移動しただけでファイルの内容は変わりませんね。

```js:{WpPost.id}.jsx
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

ファイル名でリソースを指定するだけで、`gatsby-node.js`の処理が丸ごと不要になるため、動的にページを生成する手間がだいぶ省けますね。

# ファイル名のSyntax

File System Route API でリソース取得に使うファイル名のシンタックスのメモです。

- ファイル名の全体を`{}`で囲む
- リソース名は lowercase または uppercase とする
- `.`でつないで取得したいフィールド名を記載する
- 第一階層以外のフィールドを指定したい場合は、`__`でフィール名を繋ぐ
- [GraphQLのユニオンタイプ](https://graphql.org/learn/schema/#union-types)を使いたい場合は`()`ユニオンタイプ名を囲む

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


# 諸注意

File System Route API では、取得したリソースの分だけページを生成してしまいます。
例えば、「特定の ID のページだけ作りたい」「published が false のページは作りたくない」などのように生成するページをフィルタリングすることはできません。

その場合は、まだ `gatsby-node.js` で `createPages` を使ってページを生成する必要があります。

# 終わりに

以上、簡単ですが「Gatsby.js の新しい File System Route API を試してみた」でした。
Nuxt.js や、Next.js と同じようにファイル名を変えることで動的なページの生成が出来るのは便利ですね。

今回紹介できなかったのですが、Gatsby.js を SPA 的に使う Client-only-route のページ生成にも`File System Route API`は対応しています。

https://www.gatsbyjs.com/docs/file-system-route-api/#creating-client-only-routes

Gatsby.js が今後もより進化していくのに期待です。

# 参考
- [File System Route API | Gatsby](https://www.gatsbyjs.com/docs/file-system-route-api/)