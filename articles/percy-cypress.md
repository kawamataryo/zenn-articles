---
title: "Percy + Cypress + GitHub Actionsで手軽にビジュアルリグレッションテストを導入する"
emoji: "🦔"
type: "tech"
topics: ["percy", "cypress", "githubactions"]
published: true
---

ビジュアルリグレッションテストのプラットフォーム  [Percy](https://percy.io/) を使ってみたらなかなか良かったので紹介です。

# Percyとは？

https://percy.io/

Percy はビジュアルリグレッションテスト（ページのスクリーンショットをピクセル単位で比較をすることで意図せぬ UI の変更を検知するテスト手法）のプラットフォームです。

Percy を使うことでビジュアルリグレッションテストに必要な画像の保存や、画像変化を検出するページのレンダリング、変更の通知までとても手軽に実現できます。

## 料金体系
Percy の料金体型はアップロードするスクリーンショットの枚数によって変わります。
月あたり 5,000 枚までは無料で利用できます。それ以降は 25,000 枚で 99$（年払い）, 100,000 枚で 399$（年払い）と変わっていきます。

リリース頻度やページ数によりますがリリースブランチのプルリクエストのみを検証対象とすれば、無料枠でも十分間に合うと思います。

https://www.browserstack.com/accounts/subscriptions?product=percy&ref=percy

![](https://storage.googleapis.com/zenn-user-upload/8j1eool95613d74wubdn9m91mtne)

## UI

スクリーンショットの比較画面はこちらです。
ピクセル単位での差分の検出が可能です。さらに GitHub Integration を使うことで差分についての Approve も可能です。

![](https://storage.googleapis.com/zenn-user-upload/yxahfq1vvjglfh7o7z5i3neiajs2)

:::message
正直なところ差分比較画面の細かい使い勝手は、OSS のビジュアルリグレッションツールである REG-SUIT の方が良い気がします。
https://github.com/reg-viz/reg-suit
:::

# Demo

Percy + Cypress + GitHub Actions で ビジュアルリグレッションテストの CI 環境を整えるまでのデモです。

## Percyのプロジェクト作成
Percy を利用するためにはアカウント登録が必要です。以下ページからサインアップしましょう。サインアップは[Browserstack](https://www.browserstack.com/)のアカウントと GitHub のアカウントに対応しています。

https://percy.io/signup

アカウントに登録したら、`Create a new project` で新しいプロジェクトを作成します。任意のプロジェクト名を入力してください。また GitHub 認証をしていると、リポジトリとのリンクを設定できるので、サンプルプロジェクトで利用するリポジトリを指定してください。

![](https://storage.googleapis.com/zenn-user-upload/r557f9kw983tvpz30g8hmdf9n5q8)

入力したら`create project`でプロジェクトが作成されます。API トークンが表示されるのでそちらをメモしておきます。

![](https://storage.googleapis.com/zenn-user-upload/04x97j6yd2l7i30b4yed1d3u4ygj)

## サンプルアプリの作成

[Vite](https://github.com/vitejs/vite) でサンプルアプリを作成します。

```bash
$ yarn create @vitejs/app percy-demo --template vue
```

プロジェクトに移動して、アプリが起動すれば OK です。

```bash
$ cd percy-demo
$ yarn
$ yarn dev
```

## Cypressでのスクリーンショットの撮影

Percy で差分比較に使うスクリーンショット撮影のために E2E テストフレームワークの[Cypress](https://www.cypress.io/)を導入します。

:::message
スクリーンショットが取れれば良いので、[Capybara](https://github.com/teamcapybara/capybara) や [Puppeteer](https://github.com/puppeteer/puppeteer) などその他のライブラリでも、もちろん大丈夫です。
:::

まず Cypress と Percy の依存モジュールをインストールします。

```
yarn add -D cypress @percy/cli @percy/cypress
```

そして Cypress の初期化を行います。

```
yarn run cypress open
```

コマンドを実行すると Cypress のブラウザが開き、プロジェクト直下に`cypress`ディレクトリと、`cypress.json`が作成されます。

次に Percy 専用の拡張コマンドでスクリーンショットを撮影するため、`cypress/support/index.js`に`@percy/cypress`のインポートを追記します。

```js:cypress/support/index.js
// ...
import '@percy/cypress';
```

最後に、`cypress/integration`配下にスクリーンショット撮影用のテストを追加します。

```bash
$ touch cypress/integration/index.spec.js
```

```js:cypress/integration/index.spec.js
describe('Index', () => {
  it('Capture a index page', () => {
    cy.visit('http://localhost:3000')

    cy.percySnapshot();
  })
})
```

これで準備は完了です。
あとは、アプリを起動し先程メモした Percy の API トークンを環境変数に設定したうえで、`percy exec`コマンドで Cypress を実行すればスクリーンショットが Percy のプロジェクトに送られます。

```bash
$ yarn dev
```

```bash
$ export PERCY_TOKEN=<your token>
$ yarn run percy exec -- cypress run
```

コマンド実行後に Percy のプロジェクトへのリンクが表示されます。アクセスすると以下のように差分比較画面が表示されるはずです。

![](https://storage.googleapis.com/zenn-user-upload/cs4ud2d9644pp83fcl5ekfc1zf1b)

※ 最初の実行の場合は、前の画像がないので差分は表示されません。

## GitHub Actionsの追加

GitHub Actions を使い CI でビジュアルリグレッションテストを実行できるようにします。
プロジェクトに`.github/workflows`ディレクトリを作成して、`visual-regression-test.yml`を追加します。

```
$ mkdir -p .github/workflows
$ touch .github/workflows/visual-regression-test.yml
```

```yml:visual-regression-test.yml
name: Visual Regression Test

on:
  pull_request:
    branches:
      - master

jobs:
  visual-regression-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Setup node
        uses: actions/setup-node@v2
        with:
          node-version: '14.x'
      - name: Cypress run
        uses: cypress-io/github-action@v2
        with:
          start: yarn dev
          wait-on: 'http://localhost:3000/'
          command-prefix: 'percy exec -- yarn'
        env:
          PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}
```

このコードでは Cypress 公式の GitHub Actions を使って Cypress を実行しています。

https://github.com/cypress-io/github-action

ポイントは、`with.command-prefix`で`percy exec -- yarn`を指定している点です。
これで先程ローカルで実行したときと同様、Percy にスクリーンショットが送られます。

また、Percy のトークンについては、`PERCY_TOKEN: ${{ secrets.PERCY_TOKEN }}`で環境変数に渡しています。
これは、GitHub リポジトリ画面の settings > secrets より指定した環境変数を参照しています。

![](https://storage.googleapis.com/zenn-user-upload/5fpnjsc5mbwm07awsda0d8czckpg)

あとは master に変更を反映して、その後別ブランチでプルリクエストを作成すれば、GitHub Actions が起動しプルリクエストごとに Percy で画像差分が確認できます。

以下プルリクストの CI のチェック部分です。Percy で差分がある場合はレッドになります。

![](https://storage.googleapis.com/zenn-user-upload/heqqk0rggvmgt5nc6evhqw8o6rp8)

Detail リンクをクリックすると Percy の差分比較画面に飛びます。

![](https://storage.googleapis.com/zenn-user-upload/c5spa2ee0vnfih95v0q4cx4ndjkr)

差分比較画面で左上の `Approve Build` を押せば、GitHub 上の CI のチェックもグリーンになります。これを利用して、Percy で差分が Approve されるまでマージをブロックするなどの運用も手軽に実現できそうです。

![](https://storage.googleapis.com/zenn-user-upload/yo7fwwpjmyy5zyp5ieny4555roxv)

# 終わりに

以上「Percy + Cypress + GitHub Actions で手軽にビジュアルリグレッションテストを導入する」でした。
チーム機能にも対応しているので本当に手軽にプロジェクトに組み込めそうです。まだそれほど使い込んだわけではないので詳細は分からないですが、実プロジェクトでも機会があれば導入してみたいです。