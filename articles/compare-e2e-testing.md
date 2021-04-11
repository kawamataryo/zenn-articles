---
title: "Cypress 事始め 〜導入からはじめてのE2Eテストまで〜"
emoji: "🌲"
type: "tech"
topics: ["cypress", "typescript", "vue", "e2e"]
published: false
---

realworld Vue のプロジェクトに cypress を導入するまでのメモ。

# Cypressとは？

# サンプルプロジェクトの追加

https://github.com/gothinkster/vue-realworld-example-app

```bash
git clone git@github.com:gothinkster/vue-realworld-example-app.git
```
```bash
cd vue-realworld-example-app
```
```bash
yarn & yarn serve
```

# Cypressの導入


```bash
yarn add cypress
```

```bash
yarn run cypress open
```

`cypress`ディレクトリが追加される。`cypress/integration/examples/*`、`cypress/fixtures/example.js`を削除するとこんな感じに。
```bash
$ tree cypress
cypress
├── fixtures
├── integration
├── plugins
│   └── index.js
└── support
    ├── commands.js
    └── index.js
```

package.jsonにスクリプトを追加

```json:package.json
{
  //...
  "scripts": {
    //...
    "cypress:open": "cypress open",
  },
  //...
}
```

cypress.jsonにbaseUrlを追加

```bash
{
  "baseUrl": "http://localhost:8080"
}
```

# 参考サイト
- [Cypress をお供にE2E受け入れテスト駆動開発 〜そしてAutifyへ〜｜seya｜note](https://note.com/seyanote/n/n68825bf83138)
- [E2EテストツールAutifyを使うまでの話 - チームスピリットデベロッパーブログ](https://teamspirit.hatenablog.com/entry/2020/04/17/150000)
- [Autifyを試してみました(E2E自動テストツールの話) - GA technologies GROUP Tech Blog](https://tech.ga-tech.co.jp/entry/2019/09/autify)


# メモ
binaryキャッシュがありそう。実行速度の改善？
https://docs.cypress.io/guides/getting-started/installing-cypress#Binary-cache