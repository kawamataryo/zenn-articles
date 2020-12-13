---
title: "Bolt.js⚡ + Firebase🔥で技術投稿の指標を良い感じに集計してくれるSlack Botを作った"
emoji: "🤖"
type: "tech"
topics: ["Firebase", "Bolt.js", "slack", "Firestore", "gcp"]
published: false
---

この記事は [Slack Advent Calendar 2020](https://qiita.com/advent-calendar/2020/slack) 16 日目の記事です。

Slack アプリのフレームワークである Bolt.js と Firebase を使って、自分のアウトプットの指標を良い感じに集計して、良い感じにフォーマットして出力してくれる Bot を作ってみたので紹介です。

# 作ったもの
自分のアウトプットのカウントを入力するとこんな感じに指標を集計して返してくれる Slack Bot です。

![](https://i.gyazo.com/9d69d32a88460af4d59859cf77855fbe.png)

詳細な機能は以下の通りです。


1.  `/report` コマンドを打つと起動して入力モーダルをひらく
2. モーダルで Twitter, Qiita, Zenn, note のアカウント名 + コメントの入力が出来る
3. 投稿ボタンを押すと、入力内容から各指標を集計、結果をチャネルに投稿

対応しているサービスはこちらです。

|サービス|指標|
|---|---|
|Twitter|ツイート数、フォロワー数、フォロー数|
|Qiita|記事数、LGTM 数、フォロワー数|
|Zenn|記事数、Like 数、フォロワー数|
|note|記事数、Like 数、フォロワー数|

いかに楽に入力できるか、いかに楽に自分の進捗を把握できるかに着目して開発しました。
入力については、Slack アカウトごとに前回入力済みの内容を保存し、初回以降は入力欄の初期値として入るようにしています。
自分の進捗把握については、初回以降前回の実行時からの差分を表示するようにしています。なので週 1 回とかコマンドを打てば前週との比較が手軽に出来ます。

![](https://i.gyazo.com/bdb8e8a5ea3c7146119420b02c01a777.gif)


# 構成

どのような構成かざっくり紹介します。

Bolt.js を Cloud Functions for Firebase で動かして Slack とのやりとりをしています。
そして、フォームの入力内容・指標データを Firestore に保存しています。
指標集計部分は自分で書いた各サービスの API クライントを使っています。

:::message
Zenn、note は API が公開されてないのでネットワーク情報からパス、レスポンスをみて構築しています。
:::

![](https://i.gyazo.com/67a4d02c9df3e48926edbc2a08719b3e.png)

コードについては全てリポジトリで公開しています。
API クライアントや、Bolt.js での実装など詳細は以下をご覧ください。

https://github.com/kawamataryo/blog-index




# 実装で詰まったところ

今回の Slack Bot 作成においてハマりポイントがあったので紹介します。

## Faas で Bolt.js を使う際の問題

実は Cloud Functions for Firebase や AWS Lambda などの FaaS 上で、Bolt.js による Slack Bot の構築は以下制約があるため少し工夫が必要です。

- （1）Slack から HTTP リクエストを受けたら 3 秒以内に HTTP レスポンスを返さなければならない
- （2）HTTP リクエストをトリガーに起動する FaaS では処理途中でレスポンスを返すと、後続の処理の実行は確約されない

（1）の 3 秒以内応答ルールの制約があるため Bolt.js では、`ack()`という関数が提供されています。
`ack()`は 200OK の HTTP レスポンスを返す関数で、それを実行すれば後続の処理を非同期で行うことができます。

```js
app.action('approve_button', async ({ ack, say }) => {
  // アクションリクエストの確認。この時点でレスポンスは返される
  await ack();
  // 以降は非同期で処理される。3秒以内応答の制約はない。
  await superHeavyTask()
});
```

ただ、これは、（2）の通り FaaS では利用できません。
というのも `ack()` を実行した瞬間に、その環境の処理は完了したと見なされ後続の処理の実行は確約されないからです。

これに対応するため Bolt.js 側に`ack()`の実行を処理完了まで遅らせるオプション（`processBeforeResponse`）もあるのですが、これを設定しても（1）の 3 秒以内のルールは守らなくてはなりません。

今回の Bot ではモーダルで受け取った値を利用して、各サービスの API にアクセスして結果を集計するので、どうしても 3 秒以内のレスポンスのルールが守れずタイムアウトエラーとなってしまう問題に突き当たりました。

同期的に処理を行ってしまうとタイムアウトエラー、でも Functions の関数内で非同期に実行すると FaaS の設計上、実行が確約されない・・さてどうするか🤔

## 解決策

今回は Firestore を Queue 的に使い、モーダルのレスポンスと集計処理の実行を関数単位でを分離することで前述の問題を回避しました。

処理の実行手順は以下のようになります。

**スラッシュコマンド実行時からモーダルの表示まで**

1. スラッシュコマンドの実行
2. 実行ユーザーの過去投稿をデータを Firestore から取得
3. 過去投稿があればフォームの初期値として設定
4. モーダルを表示

![](https://i.gyazo.com/7877d7be4a5e201459565ecdaba48d7c.png)

**モーダルでの送信から指標の集計、チャネルへの投稿まで**

1. モーダルで内容入力・送信
2. Firestore にデータを保存
3. 200 レスポンスを返しモーダルを閉じる
4. onCreate のフックで別関数が起動
5. 各指標を集計
6. 結果を送信


![](https://i.gyazo.com/e5cc7714869af3c9ee02452df58d3ddf.png)

特に重要なのは、モーダルでの送信からの処理で、ここで Firestore のデータ保存 -> onCreate で別関数起動という方法をとることで非同期に処理を実行しています。

これなら、指標集計にどれほど時間がかかっても、タイムアウトで落ちることはありません。

:::message
Bolt.js の Python 版である[bolt-python](https://github.com/SlackAPI/bolt-python)ではこの問題は別の方法で解決できるようです。
詳細は、以下をご覧ください。
[Bolt for Python が FaaS での実行のために解決した課題 - Qiita](https://qiita.com/seratch/items/6d142a9128c6831a6718)
:::

# 終わりに
以上「Bolt.js⚡ + Firebase🔥で技術投稿の指標を良い感じに集計してくれる Slack Bot を作る」でした。
参加しているコミュニティ（[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)）の Slack でも皆に使ってもらっているので嬉しいです。

もし、需要あれば公開アプリとしてみようかなとも思っています。反応もらえると嬉しいです。

# 参考

- [Slack | Bolt for JavaScript](https://slack.dev/bolt-js/ja-jp/concepts)
- [Bolt for Python が FaaS での実行のために解決した課題 - Qiita](https://qiita.com/seratch/items/6d142a9128c6831a6718)