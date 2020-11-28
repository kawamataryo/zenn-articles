---
title: "Gatsby.jsの新しいFile System Route APIを試してみた"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatsby.js", "JavaScript", "React"]
published: false
---

最近は Next.js が凄い勢いで進化していますが、Gatsby.js も負けず劣らず新しい機能や API が公開されています。

今回は先月公開された新しい File System Route API について紹介します。

# File System Route APIとは？

File System Route API は、ブログの個別ページなど同レイアウトだけどデータが異なるページを作る際に、ファイルの名前に特定の表記をつけることで動的にページを生成する API です。
今までは `gatsby-node.js` の `createPages` で行ってたことを代替するものですね。

次項で旧来の API と比較をしながら解説していきます。

# 旧来の方式とFile System Route APIの比較

## 旧来の方式（createPages）の場合
最初に旧来の方式で、ブログ詳細ページを作る方法を確認します。

以下 Wordpress を HeadlessCMS として使う場合のコード例なのですが、
`gatsby-node.js`の`createPages`のフックで全ての詳細ページを取得する GraphQL クエリを発行して、ページデータを取得、foreach でページデータの個数だけ、 `createPage` を実行するという流れになっています。

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

そして、`createPage`でデータを挿入する templatePage では以下のように、`createPage`時に渡す`context`から ID を受け取り、描画に必要なデータを取得して表示する必要があります。

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

ただ動的なページを作りたいだけに、`gatsby-node.js`、テンプレートのページ両方を書く必要があるのが地味に複雑で面倒ですよね。

# File System Route APIの場合

次はFile System Route APIの場合です。
File System Route API では `{ }`でファイル名を囲み、ファイル名として取得したいリソースを書くことで、今まで`gatsby-node.js`で行ってきたページの生成処理を自動的に行うことができます。

例えば、先ほどと同様Wordpressの投稿データからページを作る場合は

`{WpPost.id}.jsx`

というファイル名で`pages`ディレクトリにファイルを配置します。
こうすることで内部的には旧来の`createPages`で読んでいたクエリと同様の以下クエリが発行されます。

```graphql
allWpPost {
  nodes {
    id
  }
}
```

そしてその戻り値を今まで `createPage` の `context` で渡していた値と同様に、詳細ページのテンプレート（`{WpPost.id}.jsx`）で使うことができます。

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

ファイル名でリソースを指定するだけで、`gatsby-node.js`の処理を丸ごと書かなくてよくなるため、動的にページを生成する手間がだいぶ減ると思います。

# Syntax

先ほど説明した File System Route API でリソース取得に使うファイル名のシンタックスのメモです。

- ファイル名の全体を`{}`で囲む
- リソース名は lowercase または uppercase とする
- `.`でつないで取得したいフィールド名を記載する
- 第一階層以降のフィールドを指定したい場合は、`__`でフィール名を繋ぐ

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

# 終わりに
以上、「Gatsby.jsの新しいFile System Route APIを試してみた」でした。
Nuxt.jsや、Next.jsと同じようにファイル名を変えることで動的なページの生成が出来るのは便利ですね。

今後もより進化していくのに期待です。

# 参考
- [File System Route API | Gatsby](https://www.gatsbyjs.com/docs/file-system-route-api/)