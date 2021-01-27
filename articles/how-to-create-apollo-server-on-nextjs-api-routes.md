---
title: "Next.js の API Routes に Apollo Server を立てる"
emoji: "💠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "react", "apollo", "graphql"]
published: true
---

Next.js に Apollo Server を立てるまでの備忘録です。

# API Routes & 依存モジュールの追加

`npx create-next-app`で Next.js を作成して TypeScript の設定まで完了している前提で書きます。

まず `pages/api` に `graphql.ts` を作成します。
これが Apollo Server のエンドポイントとなります。

```bash
touch pages/api/graphql.ts
```

次に Apollo Server の [micro](https://www.npmjs.com/package/micro)（Vercel が開発している非同期 HTTP サーバー）インテグレーションを依存に追加します。

```bash
yarn add apollo-server-micro
```

これで準備は完了です。

# Apollo Server の作成

`graphql.ts`内に`typeDefs`を作ります。
これが、今から作る Apollo Server の GraphQL スキーマ定義となります。

```ts:graphql.ts
import { ApolloServer, gql, Config } from "apollo-server-micro";

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

次にサーバー処理の実態となるリゾルバを作ります。

サンプルなので外部 API や DB との接続周りは省略してオンメモリでデータを持つ構造とします。先程の`typeDefs`で定義した`getArticle`と`getArticles`のクエリーを設定しています。

```ts:graphql.ts
// オンメモリでデータを持つ
const DB = {
  articles: [
    { id: 1, title: "foo", content: "foooooooooooooo" },
    { id: 2, title: "bar", content: "baaaaaaaaaaaaaa" },
  ],
};

// リゾルバ
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

最後に Apollo Server のインスタンスを作成して export すれば完成です。
API Routes の [Custom Config](https://nextjs.org/docs/api-routes/api-middlewares#custom-config) でリクエスト body の parse を無効にする設定が必要なので注意。

```ts:graphql.ts
// Custom Config
export const config = {
  api: {
    bodyParser: false,
  },
};

const apolloServer = new ApolloServer({ typeDefs, resolvers });
export default apolloServer.createHandler({ path: "/api/graphql" });
```

http://localhost:3000/api/graphql にアクセスすると、GraphiQL のエディタが起動するはずです。完成 🎉

![](https://i.gyazo.com/4e5ffb87c53a1e7b7aed5858f3f5db02.png)

# 参考

- [next.js/examples/api-routes-apollo-server-and-client at canary · vercel/next.js](https://github.com/vercel/next.js/tree/canary/examples/api-routes-apollo-server-and-client)
- [How To Build A GraphQL Server Using Next.js API Routes — Smashing Magazine](https://www.smashingmagazine.com/2020/10/graphql-server-next-javascript-api-routes/)