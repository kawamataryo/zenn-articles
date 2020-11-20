---
title: "Gatsby.jsの新しいFile System Route APIを試してみた"
emoji: "👾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Gatsby.js", "JavaScript", "React"]
published: false
---


プロジェクトの作成

```
$ yarn add gatsby@latest
```


```
gatsby new my-gatsby-project https://github.com/gatsbyjs/gatsby-starter-blog
```

通常の場合

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

# 新しい方式



```js:{WpPost.id}.jsxf
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