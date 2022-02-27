---
title: "Cypress + Serve で Chrome拡張機能のE2Eテストを実装する"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cypress", "test", "serve", "typescript", "chromeExtension"]
published: false
---

Chrome拡張機能のE2Eテストが書けたので手法をまとめます。

# やってみたこと
Chikamichiという閲覧履歴やタブ、ブックマークを横断検索できるChrome拡張機能を作っているのですが、その拡張機能のリファクタリングの前準備として、E2Eテストを実装してみました。
この拡張機能はPopupページで動作するので、Popupページを対象にテストしています。

https://twitter.com/KawamataRyo/status/1496457270401826819

本記事ではサンプルのChrome拡張のPopupページを対象にCypressのE2Eテストを実装していきます。

実際のテストコードについてはChikamichiのリポジトリにあるので、そちらを参照してください。

https://github.com/kawamataryo/chikamichi

# サンプルのChrome拡張機能の作成
Viteese-webextというChrome拡張のtemplateリポジトリを使ってサンプルの拡張機能を作成します。

https://github.com/antfu/vitesse-webext

任意のディレクトリで以下コマンドを実行してください。

```
$ npx degit antfu/vitesse-webext e2e-sample-webext
$ cd e2e-sample-webext
$ pnpm i
$ pnpm dev
```

これでローカルサーバーが起動します。
この状態で、[Chromeの拡張機能の設定](chrome://extensions/)から、`Load unpacked` を選択して、`e2e-sample-webext/extensions`を選択します。

![](https://i.gyazo.com/170bfc6c5f19aa1a81b48c54165435c7.png)

![](https://i.gyazo.com/83d1512b27d61b1a696ffb6a1e404eea.png)

`Viteese WebExt`というサンプルの拡張機能がインストールされるはずです。

![](https://i.gyazo.com/16d6ea76c2078060ac7364c9618c9483.png)

拡張機能のアイコンをクリックして、Popupページが表示されれば準備完了です。

![](https://i.gyazo.com/f44dbe3a678a016f37a218de653b8a9e.png)

# Cypressのセットアップ
続いてテストで利用するCypressのセットアップを行います。
まずパッケージを追加します。

```
pnpm i -D cypress
```

続いて以下コマンドでCypressが起動するか試します。

```
pnpm cypress open
```

CypressのダッシュボードがChromeで起動すればOKです。

![](https://i.gyazo.com/2352b9664aa3c19f1ba2c5cf33eb1d74.png)

最後にViteのdev serverとの接続設定を行います。これはViteese-webextがViteを使っているからです。

以下を参考に設定します。

https://docs.cypress.io/guides/component-testing/framework-configuration#Vite-Based-Projects-Vue-React

```js:cypress/plugins/index.js
const path = require('path')
const { startDevServer } = require('@cypress/vite-dev-server')

module.exports = (on, config) => {
  on('dev-server:start', (options) => {
    return startDevServer({
      options,
      viteConfig: {
        configFile: path.resolve(__dirname, '..', '..', 'vite.config.js'),
      },
    })
  })
}
```

```json:cypress.json
{
  "baseUrl": "http://localhost:3000"
}
```

baseUrlのlocalhost:3000は、次項で追加するserveの起動portの指定です。

# Serveのセットアップ
Cypressでテストを実行するためには何らかの方法でテスト対象のページをブラウザからアクセスする必要があります。今回はPopupページを対象にテストするので、Popupページをレンダリングする必要があります。そのために、静的ファイルを元にコマンド一つで開発サーバーを起動できる[serve](https://github.com/vercel/serve#readme)を利用します。

https://github.com/vercel/serve




# テストケースの作成

**ポイント**
Chrome拡張機能用のモックの追加

# テストケースを書く

# 終わりに

以上、CypressとServeを使ったChrome拡張機能のE2Eテストの実装方法の紹介でした。

実際にこの方法でE2Eテストが書けたおかげで、安心して大規模なリファクタリングが行えました。

（実際のリファクタコミット）
https://github.com/kawamataryo/chikamichi/commit/f96e02d13da026a977afc84c299d324f4b11e763

このリファクタリングもE2Eテストがない状態だと、着手に躊躇していたと思います。やはり、E2Eテスト大事ですね。

Cypressでとても柔軟にテストケースをかけるということもわかったので今後の業務でも生かしていきたいです。
