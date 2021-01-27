---
title: "Next.js の API Routes に Apollo Server を立てる"
emoji: "💠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs","apollo", "graphql","react"]
published: false
---

Next.js に Apollo Server を立てるまでの備忘録です。
`npx create-next-app`で Next.js を作成して TypeScript の設定まで完了している前提で書きます。

# 依存モジュールの追加

Apollo Server の [micro](https://www.npmjs.com/package/micro)（Vercel が開発している非同期 HTTP サーバー）インテグレーションである[apollo-server-micro](https://github.com/apollographql/apollo-server/tree/main/packages/apollo-server-micro)を依存に追加します。

```bash
yarn add apollo-server-micro
```

# 型定義の作成

Apollo の関連ファイルを置く`apollo`ディレクトリを作成し、`type-defs.ts`を追加します。
これが、今から作る Apollo Server の GraphQL スキーマの型定義となります。

```bash
mkdir apollo
touch apollo/type-defs.ts
```

Article という記事のオブジェクトと、その Article を単一取得する getArticle というクエリー、複数取得する getArticles というクエリーを定義しています。

```ts:apollo/type-defs.ts
import { gql, Config } from "apollo-server-micro";

export const typeDefs: Config["typeDefs"] = gql`
  type Article {
    id: Int
    title: String
    content: String
  }

  type Query {
    getArticle(id: Int): Article
    getArticles: [Article]
  }
`;
```

# リゾルバの作成

`apollo`ディレクトリ内に、`resolvers.ts`を作り、サーバー処理の実態となるリゾルバを実装します。
先程の`typeDefs`で定義した getArticle と getArticles の処理を定義しています。サンプルなので外部 API や DB との接続周りは省略してオンメモリでデータを持つ構造とします。

```bash
touch apollo/resolvers.ts
```

```ts:graphql.ts
import { Config } from "apollo-server-micro";

// サンプルなのでオンメモリでデータを持つ
const DB = {
  articles: [
    { id: 1, title: "foo", content: "foooooooooooooo" },
    { id: 2, title: "bar", content: "baaaaaaaaaaaaaa" },
  ],
};

export const resolvers: Config["resolvers"] = {
  Query: {
    getArticle: (_, { id }: { id: number }) => {
      const articles = DB.articles?.filter((a) => a.id === id);
      return articles ? articles[0] : [];
    },
    getArticles: () => DB.articles,
  },
};
```

# Apollo Serverの作成

最後に `pages/api` に `graphql.ts` を作成し、Apollo Server を実装します。

```bash
touch pages/api/graphql.ts
```

API Routes の [Custom Config](https://nextjs.org/docs/api-routes/api-middlewares#custom-config) で request body の parse を無効にする設定が必要なので注意です。

```ts:graphql.ts
import { ApolloServer } from "apollo-server-micro";
import { typeDefs } from "../../apollo/type-defs";
import { resolvers } from "../../apollo/resolvers";

export const config = {
  api: {
    bodyParser: false,
  },
};

const apolloServer = new ApolloServer({ typeDefs, resolvers });

export default apolloServer.createHandler({ path: "/api/graphql" });

```

この状態で、`yarn dev`で Next.js を起動し、http://localhost:3000/api/graphql にアクセスすると、GraphiQL のエディタが起動するはずです。完成 🎉

![](https://i.gyazo.com/4e5ffb87c53a1e7b7aed5858f3f5db02.png)

# 参考

- [next.js/examples/api-routes-apollo-server-and-client at canary · vercel/next.js](https://github.com/vercel/next.js/tree/canary/examples/api-routes-apollo-server-and-client)
- [How To Build A GraphQL Server Using Next.js API Routes — Smashing Magazine](https://www.smashingmagazine.com/2020/10/graphql-server-next-javascript-api-routes/)