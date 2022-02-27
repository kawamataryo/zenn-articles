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
$ pnpm i -D cypress
```

続いて以下コマンドでCypressが起動するか試します。

```
$ npx cypress open
```

CypressのダッシュボードがChromeで起動すればOKです。

![](https://i.gyazo.com/2352b9664aa3c19f1ba2c5cf33eb1d74.png)

最後にViteのdev serverとの接続設定を行います。これはViteese-webextがViteを使っているからです。

以下を参考に設定します。

https://docs.cypress.io/guides/component-testing/framework-configuration#Vite-Based-Projects-Vue-React

```
$ pnpm i -D @cypress/vite-dev-server
```

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
Cypressでテストを実行するためには何らかの方法でテスト対象のページをブラウザからアクセスする必要があります。今回はPopupページを対象にテストするので、Popupページをレンダリングします。そのために、静的ファイルを元にコマンド一つで開発サーバーを起動できる[serve](https://github.com/vercel/serve#readme)を利用します。

https://github.com/vercel/serve

まずパッケージを追加します。

```
$ pnpm i -D serve
```

次に、サーバー起動のスクリプトをpackage.jsonに追記します。

```json:package.json
{
  "scripts": {
    //...
    "serve": "serve extension -l 3000"
  },
}
```

そして起動します。

```
$ pnpm serve
```

起動後、http://localhost:3000 にアクセスすると以下画面が出るはずです。

![](https://i.gyazo.com/4830b9522054010d0075be21b6a3cc35.png)

# テストケースの作成

次に実際にテストケースを書いていきます。
`cypress/integration` 配下に`sample.ts`を追加します。

```
$ touch cypress/integration/smaple.spec.js
```

ここが一番のポイントなのですが、Chrome拡張機能のAPIを利用するためには、}**Cypressのテスト実行時にChromeのランタイムをモックする**必要があります。

以下、visitの`onBeforeLoad`のフックで行っている処理が、chormeのランタイムのモックになります。

```js:cypress/integration/sample.spec.js
describe('App', () => {
  before(() => {
    // popup.htmlへのパスを指定
    cy.visit('/dist/popup/index.html', {
      onBeforeLoad(win) {
        win.chrome = win.chrome || {}
        win.chrome.runtime = {
          id: '12345',
          // ボタンクリックで実行されるAPIをモック
          openOptionsPage: cy.stub().as('openOptionsPage'),
        }
      },
    })
  })

  it('Popupが表示される', () => {
    cy.get('#app').should('include.text', 'This is the popup page')
  })

  it('Open Optionsのボタンクリックでchrome runtimeのopenOptionPageが実行される', () => {
    cy.contains('Open Options').click()
    cy.window().its('chrome.runtime.openOptionsPage').should('be.called')
  })
})
```

これで`npx cypress open`からsample.spec.tsのテストを実行すると無事Popupの表示の確認と、ボタンクリックの動作の検証が行えるはずです。

![](https://i.gyazo.com/43b11ebef8605722d6b7f5e7ac95794e.png)

# 終わりに

以上、CypressとServeを使ったChrome拡張機能のE2Eテストの実装方法の紹介でした。

実際にこの方法でE2Eテストが書けたおかげで、安心して大規模なリファクタリングが行えました。

（実際のリファクタコミット）
https://github.com/kawamataryo/chikamichi/commit/f96e02d13da026a977afc84c299d324f4b11e763

このリファクタリングもE2Eテストがない状態だと、着手に躊躇していたと思います。やはり、E2Eテスト大事ですね。

Cypressでとても柔軟にテストケースをかけるということもわかったので今後の業務でも生かしていきたいです。
