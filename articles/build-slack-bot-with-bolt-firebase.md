---
title: "Bolt.js⚡ + Firebase🔥で技術記事の指標を良い感じに集計してくれるSlack Botを作った"
emoji: "🤖"
type: "tech"
topics: ["Firebase", "Bolt", "slack", "Firestore", "gcp"]
published: false
---

Slack アプリのフレームワークである Bolt.js と Firebase を使って、自分のアウトプットの指標を良い感じに集計して、良い感じにフォーマットして返してくれる Bot を作ってみたので紹介です。

# 作ったもの
![](https://i.gyazo.com/c52630a9e69f198debad26a483ba677e.gif)

下記機能を持つ Slack Bot です。

- `/report` コマンドを打つと起動して入力モーダルをひらく
- モーダルで Twitter, Qiita, Zenn, note のアカウント名 + コメントの入力が出来る
- 投稿ボタンを押すと、入力内容にそってアウトプットの指標を集計、結果をチャネルに投稿

現在下記サービスに対応しています

- Twitter: ツイート数、フォロワー数、フォロー数
- Qiita: 記事数、LGTM 数、フォロワー数
- Zenn: 記事数、Like 数、フォロワー数
- note: 記事数、Like 数、フォロワー数

工夫ポイントは、いかに楽に入力できるか、いかに楽に自分の進捗を把握できるかです。
入力については、Slack アカウトごとに前回入力済みのサービスアカウント名を保存し、初回以降はモーダル表示時に初期値として入るようにしています。
自分の進捗把握については、初回以降前回の実行時からの差分を表示するようにしています。なので週 1 回とかコマンドを打てば前週との比較が手軽に出来ます。

# 構成

どのような構成かざっくり紹介します。

Bolt.js を Cloud Functions for Firebase で動かして Slack とのやりとりをしています。
そして、フォームの入力内容や取得データを Firestore に保存しています。
指標の集計部分は自分で書いた各サービスの API クライントを使っています。
※ Zenn、note は API が公開されてないのでネットワーク情報からパス、レスポンスをみて構築しています。

コードについては全てリポジトリで公開しています（README.md はまだです。.）。
API クライアントや、Bolt.js での実装など詳細は以下をご覧ください。

# ハマりポイント

今回の Slack Bot 作成においてかなりハマりポイントがあったので紹介します。

実は Cloud Functions for Firebase や AWS Lambda 等の FaaS 上で、Bolt.js による Bot の構築はあまり適していません。
というのも、以下制約があるからです。

- （1）Slack から HTTP リクエストを受けたら 3 秒以内に HTTP レスポンスを返さなければならない
- （2）HTTP リクエストをトリガーに起動する FaaS では処理途中でレスポンスを返すと、後続の処理の実行は確約されない

（1）の 3 秒以内応答ルールの制約があるため Bolt.js では、`ack()`という関数が提供されています。
`ack()`は 200OK の HTTP レスポンスを返す関数で、それを行うと後続の処理を非同期で行うことができます。

```js
app.action('approve_button', async ({ ack, say }) => {
  // アクションリクエストの確認。この時点でレスポンスは返される
  await ack();
  // 以降は非同期で処理される。3秒以内応答の制約はない。
  await superHeavyTask()
});
```

ただ、これは、（2）の通りFaaSでは利用できません。
`ack()`を実行した瞬間に、その環境の処理は完了したと見なされ後続の処理の実行は確約されないからです。
この問題を

なので、

物理的に関数を分けて、FirestoreをQueue的に使うことで回避しています。
以下のような構成です。

# 終わりに
以上「Bolt.js⚡ + Firebase🔥で技術記事の指標を良い感じに集計してくれる Slack Bot を作る」でしたー。
参加しているコミュニティ（[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)）の Slack でも良い感じに皆に使ってもらっているので嬉しいです。

もし、需要あれば公開アプリとしてみようかなとも思っています。反応もらえると嬉しいです。

# 参考

- [Slack | Bolt for JavaScript](https://slack.dev/bolt-js/ja-jp/concepts)
- [Bolt for Python が FaaS での実行のために解決した課題 - Qiita](https://qiita.com/seratch/items/6d142a9128c6831a6718)