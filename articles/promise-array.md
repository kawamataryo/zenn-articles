---
title: "JavaScriptで配列を元に非同期処理をループで回す際の方法色々"
emoji: "➰"
type: "tech"
topics: ["javascript"]
published: true
---

何かと使うけど、よく忘れるのでメモ。

# 並行処理

配列の各要素をもとに、並行で非同期処理を実行する場合は`Promise.all`, `map`を使います。

```ts:index.ts
class ApiClient {
  get(query: string) {
    return new Promise((resolve, _) => setTimeout(
      () => {
        console.log(`run: ${query} query`)
        resolve("ok")
      }
      , Math.random() * 1000)
    )
  }
}

const main = async () => {
  const client = new ApiClient()
  const queries = ['apple', 'banana', 'lemon', 'grape']

  // 並行に非同期処理を実効
  await Promise.all(queries.map(async (query) => client.get(query)))
}

main()
```

結果はこちら。出力は`queries`で定義した配列の並び順とは異なる場合もあります。

```bash
$ ts-node index.ts
run: grape query
run: banana query
run: lemon query
run: apple query
```

# 逐次処理

配列の各要素をもとに、逐次非同期処理を実行する場合は`for 〜 of`を使います。

```ts:index.ts
class ApiClient {
  get(query: string) {
    return new Promise((resolve, _) => setTimeout(
      () => { console.log(`run: ${query} query`) resolve("ok") } , Math.random() * 1000))
  }
}

const main = async () => {
  const client = new ApiClient()
  const queries = ['apple', 'banana', 'lemon', 'grape']

  // 直列に非同期処理を実効
  for(const query of queries) {
    await client.get(query)
  }
}

main()
```

結果はこちら。出力は`queries`で定義した配列の並び順と同様になります。

```bash
$ ts-node index.ts
run: apple query
run: banana query
run: lemon query
run: grape query
```
