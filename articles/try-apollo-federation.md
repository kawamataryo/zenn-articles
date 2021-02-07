---
title: "Next.jsのAPI RoutesでApollo Federationを試す"
emoji: "🇫🇲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["apollo", "graphql", "typescript"]
published: false
---

# Apollo Federationとは？

最近、Netflix の GraphQL アーキテクチャ構成が話題になりました。その構成を支えているのが Apollo Federation です。

https://netflixtechblog.com/how-netflix-scales-its-api-with-graphql-federation-part-1-ae3557c187e2

Apollo Federation は、複数の GraphQL マイクロサービスをゲートウェイでまとめて、1 つの GraphQL サーバーとして提供するためのアーキテクチャです。
これだけだとただのゲートウェイでの集約のように見えるのですが、Apollo Federation を使うことでそれぞれのマイクロサービスで依存する処理も良い感じに統合してくれます。

例えば、投稿情報は Post サービスにあり、その投稿の投稿者であるユーザーの情報は User サービスにあるときに、以下のようなクエリで投稿情報と投稿者情報を取得したいとします。

```graphql
query {
  posts {
    id
    title
    content
    user {
      id
      name
    }
  }
}
```

この時に Apollo Federation を使わない場合は、以下のように投稿情報を処理する Post サーバーからユーザー情報を処理する User サーバーにクエリを投げる必要があり、マイクロサービスごとが密結合になってしまいます。

![](https://i.gyazo.com/98d841301f82c3f4092733be8545ba77.png)

それを Apollo Federation を使うとゲートウェイ側良い感じにハンドリングしてくれて、マイクロサービス間の結合度を下げられます。

![](https://i.gyazo.com/eaf71bde17a4d27692129656b8e6a503.png)
# Demo

サンプルアプリを作ります。
Next.js の環境構築と TypeScript 化が終わっている前提で書きます。

構成はこちらです。Next.js の API Routes にゲートウェイサーバーを立て、その後ろに 2 つの GraphQL サーバーを作ります。

## マイクロサービス側の実装

まずモックのマイクロサービス用ディレクトリを作ります。

```bash
mkdir -p servers
mkdir servers/posts # 投稿者情報の処理サーバー
mkdir servers/users # ユーザー情報の処理サーバー
```

そして、必要なライブラリをインストールします。

```bash
yarn add ts-node apollo-server @apollo/federation
```

### Usersサーバー
ユーザー情報を管理する Users サーバーを作ります。

```
users
├── index.ts
├── resolvers.ts
└── type-defs.ts
```

:::details servers/users/type-defs.ts
```ts:servers/users/type-defs
import { gql } from 'apollo-server';

export const typeDefs = gql`
  type User @key(fields: "id") {
    id: Int!
    name: String!
  }

  type Query {
    user(id: Int): User
    users: [User]
  }

  type Mutation {
    addUser(name: String): User
  }
`;
```
:::

:::details servers/users/resolvers.ts
```ts:
export const DB = {
  users: [
    { id: 1, name: 'taro' },
    { id: 2, name: 'jiro' },
  ],
};

export const resolvers = {
  Query: {
    users: () => DB.users,
    user: (_, { id }) => DB.users.find((u) => u.id === id),
  },

  Mutation: {
    addUser: (_, { name }) => {
      const user = { id: DB.users.length + 1, name };
      DB.users.push(user);
      return user;
    },
  },

  User: {
    __resolveReference(user) {
      return DB.users.find((u) => u.id === user.id);
    },
  },
};
```
:::

:::details servers/users/index.ts
```ts:servers/users/index.ts
import { ApolloServer } from 'apollo-server';
import { typeDefs } from './type-defs';
import { resolvers } from './resolvers';
import { buildFederatedSchema } from '@apollo/federation';

const server = new ApolloServer({
  schema: buildFederatedSchema([{ typeDefs, resolvers }]),
});

server.listen({ port: 4002 }).then(({ url }) => {
  console.log(`🚀  Server ready at ${url}`);
});
```
:::

Federation のポイントは、type-defs.ts のスキーマ定義にある下記の部分です。

```graphql
type User @key(fields: "id") {
  id: Int!
  name: String!
}
```

オブジェクト型のスキーマ定義に`@key`ディレクティブを追加することで、エンティティとして指定できます。
エンティティはゲートウェイを通して、他の GraphQL サーバーから参照できる型です。

`(fields: "id")`の部分はエンティティの主キーの定義です。ゲートウェイサーバーではこの主キーを使って、識別をします。
今回は User 型の id フィールドを指定しています。


resolvers.ts では下記のコードで別 GraphQL サーバーからエンティティである User 型が参照された際の処理を書いています。

```ts
export const resolvers = {
  //...

  User: {
    __resolveReference(user) {
      return DB.users.find((u) => u.id === user.id);
    },
  },
}
```

この例では別 GraphQL サーバーから User が参照されたときに、こちらのリゾルバが呼ばれインメモリのオブジェクトから対象の User を探して返しています。

### Postsサーバー
投稿情報を管理する Posts サーバーを作ります。

```
posts
├── index.ts
├── resolvers.ts
└── type-defs.ts
```

:::details servers/posts/type-defs.ts
```ts:servers/posts/type-defs.ts
import { gql } from 'apollo-server';

export const typeDefs = gql`
  type Post {
    id: Int
    title: String
    content: String
    user: User
  }

  type DeleteResponse {
    result: Boolean
  }

  type Query {
    post(id: Int): Post
    posts: [Post]
  }

  type Mutation {
    addPost(title: String, content: String, userId: Int): Post
  }

  extend type User @key(fields: "id") {
    id: Int! @external
  }
`;
```
:::

:::details servers/posts/resolvers.ts
```ts:servers/posts/resolvers.ts
const DB = {
  posts: [
    { id: 1, title: 'foo', content: 'foooo', user: { id: 1 } },
    { id: 2, title: 'bar', content: 'baaaa', user: { id: 2 } },
    { id: 3, title: 'baz', content: 'bazzz', user: { id: 1 } },
  ],
};

export const resolvers = {
  Query: {
    post: (_, { id }) => DB.posts.find((u) => u.id === id),
    posts: () => DB.posts,
  },

  Mutation: {
    addPost: (_, { title, content, userId }) => {
      const post = { id: DB.posts.length + 1, title, content, user: { id: userId } };
      DB.posts.push(post);
      return post;
    },
  },

  Post: {
    user(post) {
      return { __typename: 'User', id: post.user.id };
    },
  },
};
```
:::

:::details servers/posts/index.ts
```ts:servers/posts/index.ts
import { ApolloServer } from 'apollo-server';
import { typeDefs } from './type-defs';
import { resolvers } from './resolvers';
import { buildFederatedSchema } from '@apollo/federation';

const server = new ApolloServer({
  schema: buildFederatedSchema([{ typeDefs, resolvers }]),
});

server.listen({ port: 4001 }).then(({ url }) => {
  console.log(`🚀  Server ready at ${url}`);
});
```
:::

ポイントは、`type-defs.ts`の GraphQL 定義の下記の部分です。

```graphql
  type Post {
    id: Int
    title: String
    content: String
    user: User
  }

  #...

  extend type User @key(fields: "id") {
    id: Int! @external
  }
```

Post サーバーにはない User の情報を Post の型定義として使っています。そしてその参照先はに`extend type User`になっています。
`extends` をつけるとその型は外部の GraphQL サーバーのエンティティに対する参照型となります。
`@key(fields: "id")`ディレクティブがその型を参照する際の主キーです。そして、id に対して`@external`ディレクティブを付与して、このフィールドが外部の GraphQL サーバーにオリジナルがあることを示しています。

resolvers.ts 下記を定義しています。

```ts
export const resolvers = {
  //...

  Post: {
    user(post) {
      return { __typename: 'User', id: post.user.id };
    },
  },
}
```

この関数で return するのは User 型を解決するために必要な情報です。`__typename`は User がある GraphQL サーバーの識別子、そして id は参照の主キーです。

### マイクロサービスの起動

Posts サーバーと Users サーバーを起動させるためのスクリプトを package.json に追記します。

```json
  "scripts": {
    // ...
    "dev:server:posts": "ts-node -O '{\"module\":\"commonjs\"}' servers/posts/index.ts",
    "dev:server:users": "ts-node -O '{\"module\":\"commonjs\"}' servers/users/index.ts"
  },
```

これで、`yarn dev:server:posts`, ``yarn dev:server:users`コマンドでマイクロサービスの GraphQL サーバーが起動します。

![](https://i.gyazo.com/853d69698467c56270f4d318d7aa9db8.png)

以上でマイクロサービス側の実装は完了です。

### ゲートウェイ側の実装（API Routes）
続いて Next.js の API Routes に立てる GraphQL サーバー（ゲートウェイ）を作ります。

まず、Apollo Server の [micro](https://www.npmjs.com/package/micro) インテグレーションである[apollo-server-micro](https://github.com/apollographql/apollo-server/tree/main/packages/apollo-server-micro)と、Apollo Federation のゲートウェイ側ライブラリである[@apollo/gateway](https://www.npmjs.com/package/@apollo/gateway)を依存に追加します。

```bash
yarn add @apollo/gateway apollo-server-micro
```

次に Next.js の pages/api 配下に`graphql.ts`を追加します。

```
src/pages/api
└── graphql.ts
```

```ts:pages/api/graphql.ts
import { ApolloServer } from 'apollo-server-micro';
import { ApolloGateway } from '@apollo/gateway';

export const config = {
  api: {
    bodyParser: false,
  },
};

const gateway = new ApolloGateway({
  serviceList: [
    { name: 'posts', url: 'http://localhost:4001' },
    { name: 'users', url: 'http://localhost:4002' },
  ],
});

export default new ApolloServer({
  gateway,
  subscriptions: false,
}).createHandler({
  path: '/api/graphql',
});
```

`ApolloGateway`の初期化時に、先程作成したモックのモックのマイクロサービスのエンドポイント情報を指定しています。
そして`ApolloServer`の初期化時に ApolloGateway のインスタンスを渡します。
また、この時に`subscriptions: false`を設定しています。これは 2021/02/06 現在 Apollo Gateway が GraphQL の Subscription と互換性がないためです（[参考](https://github.com/apollographql/federation/issues/426)）。

この状態で、`yarn dev`で Next.js を起動し、http://localhost:3000/api/graphql にアクセスすれば GraphiQL が起動するはずです。

![](https://i.gyazo.com/a5d50361de6896f0bcc0f15143b00b93.png)

しっかりそれぞれおのマイクロサービス側のクエリとミューテーションがスキーマに存在し、Post の User のように依存するクエリにも対応しています。

```graphql
query {
  posts {
    id
    title
    content
    user {
      id
      name
    }
  }
}
```

これで Apollo Federation の完成です。


# 参考
- [Entities - Apollo Federation - Apollo GraphQL Docs](https://www.apollographql.com/docs/federation/entities/)
- [Apollo Federationのすゝめ -GraphQLとマイクロサービス- | Web系エンジニアのアウトプット練習場](https://blog.h-sakano.dev/posts/ixgnj6xg8)
- [GraphQLでマイクロサービスアーキテクチャを構築する際に有効なApollo Federationを採用する際に注意すべきこと | Web系エンジニアのアウトプット練習場](https://blog.h-sakano.dev/posts/ybaupsbu8)
- [GraphQLとマイクロサービスは相性が良さそうな件 〜Apollo Federationを用いたスキーママージについて〜 | スペースマーケットブログ](https://blog.spacemarket.com/code/graphql-apollo-federation/)