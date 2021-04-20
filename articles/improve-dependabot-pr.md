---
title: "GitHub Actions で Dependabot プルリクエストの滞留を防ぐ"
emoji: "🌊"
type: "tech"
topics: ["githubactions", "github", "bash"]
published: false
---

自動的にライブラリのアップデートのプルリクエストを作ってくれる[Dependabot](https://dependabot.com/)はとても便利です。
ただ、今のチームでは何かと通常の開発タスクに追われ、Dpendabot のプルリクエストが放置されがちでした。何とかそれを解決するための仕組みはないかなと思い、試行錯誤してみたので書きます。

# DependabotのPR作成時に自動的にレビュアーをアサイン

私のチームのプルリクエスト作成からマージまでの流れは以下です。

1. プルリクエスト作成時に任意のチームメンバーを 1 人選びレビュアーにアサイン
2. レビュアーがレビュー後マージ

基本的には、レビュアーにアサインされたものをレビューするという運用なので、レビュアーのアサインがない Dependabot のプルリクエストは滞留しがちでした。

それを改善するために、Dependabot のプルリクエストのみ自動的にレビュアーをアサインする仕組みを GitHub Actions を使って作りました。
以下がコードです。

```yml
name: Reviewer assign action

on:
  pull_request:
    types: [opened]
    branches:
     - 'dependabot/**'

jobs:
  reviewer-assign:
    runs-on: ubuntu-18.04
    steps:
      # ランダムでレビュアーをアサイン
      - name: Assign reviewer
        uses: hkusu/review-assign-action@v1
        with:
          reviewers: Chanmoro, denzow, p-baleine, rockymanobi, yktakaha4, yuichikun
          max-num-ofreviewers: 1
      # ライブラリアップデートロールをアサイン
      - if: contains(env.GITHUB_REF, 'npm_and_yarn')
        run: echo ROLL_USER=kawamataryo >> $GITHUB_ENV
      - if: contains(env.GITHUB_REF, 'pip')
        run: echo ROLL_USER=showwin >> $GITHUB_ENV
      - name: Assign roll user
        uses: hkusu/review-assign-action@v1
        with:
          reviewers: ${{ env.ROLL_USER }}
          assignees: ${{ env.ROLL_USER }}

```

プルリクエストの自動レビュアー登録には、[Review Assign Action](https://github.com/hkusu/review-assign-action) という便利な GitHub Actions が公開されていたのでそちらを利用しています。

https://github.com/marketplace/actions/review-assign-action

上記のコードだと、最初の `name: Assign reviewer` のステップで `reviewers` に指定されているユーザーから、ランダムに 1 人がレビュアーにアサインされます。
そしてその後の `name: Assign roll user` の方で、ライブラリの種類（ここでは npm か pip）によって、専任の担当者を決めています。これはライブラリアップデートという役割を持つメンバーがチームにいて、その者をランダムなレビュアーとは別に必ずアサインするためです。このように GitHub Actions の `if` 構文を使うことで条件によって指定するものを変えることも可能です。


また、Dependabot のプルリクエストのみを対象にするために、`on.pull_request.branches`で`dependabot/**`を指定しています。これで、通常のプルリクエストは対象にならず、Dependabot のプルリクエストのみこの GitHub Actions が実行されます。


:::message
Dependabot の標準の設定でも、レビュアーやアサイナーの設定はできるのですが、複数人の候補からランダムに 1 人を選ぶというのはできないので GitHub Actions で対応しています。
もし必ず固定メンバーをアサインということだったら、以下で設定可能です。
https://docs.github.com/en/code-security/supply-chain-security/configuration-options-for-dependency-updates#reviewers
:::

# 静的アセットのビルド差分からレビューの必要性を判断

今のチームではバックエンドは Python、フロントエンドが Vue.js という構成なので、Node.js はランタイムで利用せず、あくまで静的アセット（JS, CSS, Image）のビルドのみです。
そのため、npm モジュールのライブラリアップデートブランチでビルドされた静的アセットが、master ブランチでビルドされた静的アセットと差分がなければ実行状態は変わららないはずです。
なので、そのようなモジュールのアップデートプルリクエストは特にレビュー不要でマージして良いはずです。

:::message
この考えは前職の開発チームで[@mugi_uno](https://twitter.com/mugi_uno)が作ってくれたスクリプトをそのまま参考にしています。ありがとうございます🙏
:::

その差分比較を毎回手動で行うのは面倒なので、GitHub Actions で自動実行できるようにしました。

下準備として、比較用のビルドスクリプトを組みます。
[webpack-merge](https://www.npmjs.com/package/webpack-merge) を利用して、Production の webpack config から、output のパス のみ環境変数で指定できるように書き換えた webpack config を作っています。

※ ビルド環境によって`output`以外にも手を加える必要があります（例： SentryWebpackPlugin, MiniCssExtractPlugin の挙動など）

```js:src/webpack/webpack.comparison.conf.js
const merge = require('webpack-merge')
const prodConf = require('./webpack.prod.conf')

module.exports = merge(prodCoNF, {
  output: {
    path: process.env.OUT_DIR,
    filename: 'js/[name].js',
    sourceMapFilename: 'js/[name].js.map',
  }
})
```

そしてそれを、実行する npm スクリプトを package.json に追加します。

```json:package.json
{
  // ...
  "scripts": {
    // ...
    "build:comp": "NODE_ENV=production OUT_DIR=$OUT_DIR webpack --config webpack.comparison.conf.js"
  }
}
```

後は、このスクリプト利用して、Dependabot の npm モジュールのプルリクエストの場合に静的アセットので差分を取る GitHub Actions を追加するだけです。

```yaml:.github/workflows/compare-static-assets.yml
name: Compare static assets

on:
  pull_request:
    types: [opened]
    branches:
     - 'dependabot/**'

jobs:
  compare-static-assets-job:
    timeout-minutes: 10
    runs-on: ubuntu-latest
    steps:
      # 前処理
      - name: Checkout current branch
        uses: actions/checkout@v2
      - name: Setup Node.js
        uses: actions/setup-node@v1
        with:
          node-version: '14.x'
      # プルリクエストのブランチでビルド
      - name: Install dependencies
        run: yarn
      - name: Build on current branch
        run: export OUT_DIR=/tmp/current; yarn build:comp
      # masterブランチでビルド
      - name: Checkout master branch
        uses: actions/checkout@v2
        with:
          ref: 10848/feature/add_comparison_build
      - name: einstall dependencies
        run: yarn
      - name: Build on master branch
        run: export OUT_DIR=/tmp/master; yarn build:comp
      # 静的アセットの比較
      - name: Compare static assets
        run: |
          export RESULT_FILE=/tmp/result.txt;
          git diff --compact-summary /tmp/current /tmp/master > $RESULT_FILE;
          if [ ! -s $RESULT_FILE ]; then
            sed -e '1i 静的アセットのビルド結果に差分はありません🎉' $RESULT_FILE;
          else
            sed -e '11,$d' $RESULT_FILE;
            sed -e '1i 静的アセットのビルド結果に差分があります👀' $RESULT_FILE;
            sed -e '$a ...' $RESULT_FILE;
          fi;
      # 結果をPRにコメント
      - name: Comment on PR
        uses: machine-learning-apps/pr-comment@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          path: /tmp/results.txt
```

ちょっと長いのですが、このコードでは以下を行っています。

1. プルリクエストのブランチでビルドを実行。生成物を `/tmp/current` に格納
2. master ブランチでビルドを実行。生成物を `/tmp/master` に格納
3. `git diff` で `/tmp/current`と`/tmp/master`を比較。結果を `/tmp/result.txt` に記録
4. `/tmp/result.txt` をフォーマット
5. `/tmp/result.txt` をもとに結果をプルリクエストにコメント

差分がないとコメントされた場合は、現在の master と動きに差は無いはずなので、気軽にマージできます。

# レビュー不要のプルリクエストの自動 merge の設定

Lint やテストなどに使われるモジュールに関するアップデートは基本的に、CI が通ればマージして良いはずです。
そのプルリクエストに時間を取られるのも煩わしいので、自動マージの設定をします。

以前は、Dependabot の設定で出来たのですが、v2 からは利用できなくなったので、GitHub Actions で実現します。

https://github.com/marketplace/actions/dependabot-auto-merge

```yaml
name: auto-merge

on:
  pull_request:

jobs:
  auto-merge:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: ahmadnassri/action-dependabot-auto-merge@v2
        with:
          target: minor
          github-token: ${{ secrets.mytoken }}
```

# おわりに
以上、「GitHub Actions で Dependabot のプルリクエスト滞留問題を解決する仕組み作り」でした。
まだ、運用を始めたばかりで道半ばというところですが、この仕組を使って良い感じにバージョンアップを進められればと思っています。