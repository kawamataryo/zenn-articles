---
title: "GitHub API v4 でページネーションを考慮したクエリの実装"
emoji: "📖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "TypeScript", "Node.js", "GraphQL"]
published: true
---

GitHub API v4 でページネーションを考慮してクエリを投げる機会があったので、その作業メモです。

# GitHub API v4のページネーション

GitHub API v4 で nodes や edges を持つリソースには、[PageInfo](https://docs.github.com/ja/graphql/reference/objects#pageinfo)というオブジェクトフィールドがあり、以下 4 つのフィールドを持ちます。

- endCursor
- hasNextPage
- hasPreviousPage
- startCursor

ページングでは、この`endCursor`と`hasNextPage`のペア又は`startCursor`と`hasPreviousPage`のペアを使います。
endCursor は、nodes 内の最後のノードのカーソルです（`cursor:xxx`を base64 でエンコードした文字列が入ります）。hasNextPage はそのまま次のページがあるかどうかの boolean です。

nodes を持つクエリには、`after`、`before`、`first`、`last`のページング用の引数があります。
first、last は必須の引数です。スタート位置からどれくらいの件数を取得するかを指定します。after、before はスタート位置を決めるものです。ここに先程の PageInfo で取得した endCursor、startCursor の値を指定することでページングが実現できます。

https://docs.github.com/en/graphql/reference/queries

# 実例

あるオーナーのすべてのリポジトリを取得するクエリを Node.js で書いてみました。


```ts
import fetch from 'node-fetch'

type Repository = {
  name: string;
  url: string;
};

type APIResponse = {
  data: {
    repositoryOwner: {
      repositories: {
        pageInfo: {
          endCursor: string,
          hasNextPage: boolean
        }
        nodes: Repository[]
      }
    }
  }
};

const GITHUB_ACCESS_TOKEN = process.env.GITHUB_ACCESS_TOKEN
const OWNER = 'kawamataryo'

const fetchRepositories = async  (endCursor?: string): Promise<APIResponse> => {
  const res = await fetch("https://api.github.com/graphql", {
    method: "post" as const,
    headers: {
      Authorization: `Bearer ${GITHUB_ACCESS_TOKEN}`,
    },
    body: JSON.stringify({query: `
      query {
        repositoryOwner(login: ${OWNER}) {
          repositories (
            first: 30
            ${endCursor ? `after: "${endCursor}"` : ''}
          ) {
            pageInfo {
              hasNextPage
              endCursor
            }
            nodes {
              name
              url
            }
          }
        }
      }
    `}),
  });
  return await res.json() as APIResponse
}

export const getAllRepositories = async (Repositories: Repository[], endCursor?: string): Promise<never | Repository[]> => {
  const res = await fetchRepositories(endCursor)
  const pageInfo = res.data.repositoryOwner.repositories.pageInfo
  const nodes = res.data.repositoryOwner.repositories.nodes

  // 再帰の終了条件
  if (!pageInfo.hasNextPage) {
    return  Repositories
  }

  return getAllRepositories([...Repositories, ...nodes], pageInfo.endCursor)
}
```

ポイントは、GraphQL リクエストを行う`fetchRepositories`で endCursor を受け取り、クエリ枚に after に異なる値を設定していること。`getAllRepositories`で、pageInfo から endCursor を取得して 毎回異なる endCursor で fetchRepositories を実行していることです。
`getAllRepositories`を再帰的に実行しているので、終了条件の hasNextPage が false のときに全てのリポジトリ情報が取得出来ます。
