---
title: "GitHub Actions で Dependabot のプルリクエストの滞留を防ぐ仕組みづくり"
emoji: "🛤"
type: "tech"
topics: ["githubactions", "github", "dependabot"]
published: true
---

自動的にライブラリのアップデートのプルリクエストを作ってくれる[Dependabot](https://dependabot.com/)はとても便利です。ただ、何かと通常の開発タスクに追われライブラリアップデートのプルリクエストは滞留しがちです。それを解決するための仕組みはないかなと思い、試行錯誤してみたので書きます。
# 静的アセットのビルド差分からレビューの必要性を判断

今のチームのプロダクトでは静的アセット（JS, CSS, Image）のビルドにのみ Node.js を利用しています。
そのため、npm モジュールのライブラリアップデート時に**プルリクエストのブランチでビルドされた静的アセットが、master ブランチでビルドされた静的アセットと差分がなければプロダクトの動きは変わららないはず**です。
なので、そのビルド差分の有無をみれば詳細なレビューが必要かどうか判断できます。差分もなく CI も通っていればほぼ動作確認は不要で、Change log の確認だけでマージしてもよいでしょう。

※ 差分が出ない場合の例： Test 系、Lint 系、ビルド系のライブラリ、Tree Shaking で除去される部分のコードの変更など

:::message
この考え・仕組みは前職の開発チームで[@mugi_uno](https://twitter.com/mugi_uno)が作ってくた仕組みを参考にしています。感謝🙏
:::

その差分比較を毎回手動で行うのは面倒なので、GitHub Actions で自動実行できるようにしました。


下準備として、任意のパスに静的アセットを出力する比較用のビルドスクリプトが必要です。
[webpack-merge](https://www.npmjs.com/package/webpack-merge) を利用して、Production の webpack config から、output のパス のみ環境変数で指定できるように書き換えた webpack config を作ります。

※ ビルド環境によって`output`以外にも手を加える必要があります（例： SentryWebpackPlugin, MiniCssExtractPlugin の挙動など）

```js:webpack.comparison.conf.js
const merge = require('webpack-merge')
const prodConf = require('./webpack.prod.conf')

module.exports = merge(prodConF, {
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
  pull_request_target:
    types: [opened, ready_for_review, reopened]

jobs:
  compare-static-assets-job:
    timeout-minutes: 10
    if: contains(github.head_ref, 'dependabot/npm_and_yarn')
    runs-on: ubuntu-18.04
    steps:
      # 前処理
      - name: Setup Node.js
        uses: actions/setup-node@v1
        with:
          node-version: '14.x'
      # プルリクエストのブランチでビルド
      - name: Checkout current branch
        uses: actions/checkout@v2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - name: Install dependencies
        run: yarn
      - name: Build on current branch
        run: export OUT_DIR=/tmp/current; yarn build:comp
      # masterブランチでビルド
      - name: Checkout master branch
        uses: actions/checkout@v2
        with:
          ref: ${{ github.event.pull_request.base.sha }}
      - name: Reinstall dependencies
        run: yarn
      - name: Build on master branch
        run: export OUT_DIR=/tmp/master; yarn build:comp
      # 静的アセットの比較
      - name: Compare static assets
        run: git diff --compact-summary /tmp/current /tmp/master > /tmp/result.txt || true
      # 結果をPRにコメント
      - name: Comment to PR
        uses: actions/github-script@v3
        with:
          github-token: ${{secrets.GITHUB_TOKEN}}
          script: |
            const fs = require('fs')
            const result = fs.readFileSync('/tmp/result.txt', 'utf8')
            const commentBody = result ?
              `静的アセットのビルド結果に差分があります👀<details><summary>詳細</summary><pre>${result}</pre></details>`
              : '静的アセットのビルド結果に差分はありません🎉'

            await github.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: commentBody
            })
```

ちょっと長いのですが、このコードでは以下を行っています。

1. プルリクエストのブランチでビルドを実行。静的アセットを `/tmp/current` に格納
2. マージ先ブランチでビルドを実行。静的アセットを `/tmp/master` に格納
3. `git diff` で `/tmp/current`と`/tmp/master`を比較。結果を `/tmp/result.txt` に記録
4. `/tmp/result.txt` をもとにプルリクエストへコメントを投稿

こちらが実行結果です。

**差分がない場合**
![](https://storage.googleapis.com/zenn-user-upload/1ft5n4j30866cze0ddak9wo3kxs8)

**差分がある場合**
![](https://storage.googleapis.com/zenn-user-upload/co3e550t6fzpaym7belne7s3q60r)

差分がないとコメントされた場合は、気軽にマージできます。

# Dependabotのプルリクエスト作成時にランダムにレビュアーをアサイン

私のチームのプルリクエスト作成からマージまでの流れは以下です。

1. プルリクエスト作成時に任意のチームメンバーを 1 人選びレビュアーにアサイン
2. レビュアーがレビュー後マージ

基本的には、レビュアーにアサインされたものをレビューするという運用なので、レビュアーのアサインがない Dependabot のプルリクエストは後回しになりがちです。

それを改善するために、Dependabot のプルリクエストのみ自動的にレビュアーをアサインする仕組みを GitHub Actions を使って作りました。
以下がコードです。

```yml
name: Reviewer assign action

on:
  pull_request_target:
    types: [opened]

jobs:
  reviewer-assign:
    timeout-minutes: 10
    runs-on: ubuntu-18.04
    if: contains(github.head_ref, 'dependabot/npm_and_yarn') || contains(github.head_ref, 'dependabot/pip')
    steps:
      # ランダムでレビュアーをアサイン
      - name: Assign reviewer
        uses: hkusu/review-assign-action@v1.0.0
        with:
          reviewers: taro, jiro, masaki, ichiro
          max-num-of-reviewers: 1
      # ライブラリアップデートロールをアサイン
      - if: contains(github.head_ref, 'npm_and_yarn')
        run: echo ROLL_USER=kawamataryo >> $GITHUB_ENV
      - if: contains(github.head_ref, 'pip')
        run: echo ROLL_USER=shiro >> $GITHUB_ENV
      - name: Assign roll user
        uses: hkusu/review-assign-action@v1.0.0
        with:
          reviewers: ${{ env.ROLL_USER }}
          assignees: ${{ env.ROLL_USER }}
```

プルリクエストの自動レビュアーアサインには、[Review Assign Action](https://github.com/hkusu/review-assign-action) を利用しています。

https://github.com/marketplace/actions/review-assign-action

上記のコードだと、最初の `name: Assign reviewer` のステップで `reviewers` に指定されているユーザーから、ランダムに 1 人がレビュアーにアサインされます。
そして、その後の `name: Assign roll user` の方で、ライブラリの種類（ここでは npm か pip）によって専任の担当者を決めています。これはライブラリアップデートという役割を持つメンバーがチームにいて、その者をランダムなレビュアーとは別に必ずアサインするためです。このように GitHub Actions の `if` 構文を使うことで条件によって動的にアサイン対象を変えることも可能です。


また、Dependabot のプルリクエストのみを対象にするために、`jobs.xxx.if`で dependabot の作成ブランチのみ true を返すように指定しています。これで、通常のプルリクエストは対象にならず、Dependabot のプルリクエストのみこの GitHub Actions が実行されます。


:::message
Dependabot の標準の設定でも、レビュアーやアサイナーの設定はできるのですが、複数人の候補からランダムに 1 人を選ぶというのはできないので GitHub Actions で対応しています。
もし必ず固定メンバーをアサインということなら、[こちら](https://docs.github.com/en/code-security/supply-chain-security/configuration-options-for-dependency-updates#reviewers)で設定可能です。
:::

:::message
Dependabot と同様の機能を持つ[Renovate](https://github.com/renovatebot/renovate) の場合は、指定メンバーの中からのランダムアサインも可能なようです。
:::

# 詰まったところ

実装上で色々詰まった部分があったのでまとめます。

### 対象ブランチで動作しない・・

当初対象ブランチの指定方法を間違え、以下のように`pull_request.branches`でブランチ名を指定していました。

```yaml
on:
  pull_request:
    types: [opened]
    branches:
     - 'dependabot/**'
# ...
```

これだと、マージ先ブランチが`dependabot/**`の場合のみしか動作しません。`pull_request`のトリガーで起動ブランチを絞りたい場合は、`jobs.xxx.if`により制御する必要があります。
詳細はこちらの記事にもまとめました。

https://zenn.dev/ryo_kawamata/articles/github-actions-specific-branch

### Dependabot作成のPRだけ、403でコメント・レビュアーアサインが落ちる・・

「もう完璧に動くやろ！」とメインブランチにマージした後に気がついたのですが、Dependabot の作ったプルリクエストの場合のみ、同じ GitHub Actions でも書き込み系の操作で 403 エラーが発生しました。これは、Dependabot のみ`GITHUB_TOKEN`で取れるトークンが読み取り専ようになるため起こるようです。

これを回避するためには、起動トリガーを`pull_request_target`に変更する必要があります。こちらの Issue を参考にしました。

https://github.com/dependabot/dependabot-core/issues/3253

:::message 
こういう面倒な点を考慮すると、Renovate を使ったほうが良いのかもしれないです。
:::

### Runの中でエラーでもないのになぜか毎回終了する・・

差分を取るために GitHub Actions の run でコマンドを実行しているのですが、なぜか`git diff`で差分がある場合のみ、コマンドがそこで終了するという現象に悩まされました。
原因は、Git 管理対象外のファイルに`git diff`を行った際に差分がある場合、終了コードが`1`になるためでした。GitHub Actions の run は終了コードが`1`となるコマンドが実行されるとそこで処理を停止するようです。

今回はコマンドでは、最後に`|| true`をつけることで回避しています。

```yaml
#...
git diff --compact-summary /tmp/current /tmp/master > $RESULT_FILE || true
#...
```

### プロジェクト内のSubmoduleのcloneに失敗・・
今回この仕組を導入したプロジェクトが、プライベートリポジトリの Git Submodule を含むプロジェクトだったため `actions/checkout@v2` の通常の submodule の設定だけではうまく行かず詰まりました。結局、以下 Issue を参考に、Personal Access Token を設定することで回避しました。

https://github.com/actions/checkout/issues/287

```yaml
      - name: Checkout current branch
        uses: actions/checkout@v2
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          token: ${{ secrets.PAT }}
          submodules: 'recursive'
```

# おわりに
以上、「GitHub Actions で Dependabot のプルリクエスト滞留問題を解決する仕組み作り」でした。まだ、運用を始めたばかりで道半ばというところですが、この仕組を使って良い感じにバージョンアップを進められればなと思っています。

# 参考

- https://simple-minds-think-alike.hatenablog.com/entry/dependabot-pull-request-fail
- https://github.com/dependabot/dependabot-core/issues/3253