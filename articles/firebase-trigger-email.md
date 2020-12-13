---
title: "Firebase Trigger Emailを使ってお問い合わせフォームを作る"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "vue", "typescript"]
published: false
---

以前書いた[Cloud Functions for Firebase で問い合わせフォームを作った記事](https://qiita.com/ryo2132/items/7cdd6c86dd418095f74a)がいまだに反応があるので、より使いやすい Firebase Trigger Email 版で書き直して見ました。


# 何を作る？

こんな感じの問い合わせフォームを作ります。

@[codesandbox](https://codesandbox.io/embed/cotactform-sample-rdqs3?fontsize=14&hidenavigation=1&theme=dark)

お問い合わせを実行すると、管理者のメールアドレスに問い合わせ内容と、フォーム記載のメールアドレスに thanks メールが送られます。

技術スタックは以下の通りです。

- フロントエンド
  - TypeScript
  - Vue.js
- バックエンド
  - Firebase Trigger Email
  - Firestore
  - Cloud Functions for Firebase
  - SendGrid

構成はこんな感じになっています。

1. Vue で作ったお問い合わせフォームからお問い合わせ
2. Firestore にデータを保存
3. 保存をフックに Cloud Functions が起動、Firebase Extensions と連携している FireStore の Collection にデータを保存
4. 保存をフックに SendGrid 経由で管理者と、ユーザーにメールを送信

![](https://storage.googleapis.com/zenn-user-upload/e5azosigewp3hkqhb5ox8ijiodri)

# Firebase Trigger Emailとは？

最初に簡単に今回の主役である Firebase Trigger Email について。
Firebase Trigger Email は Firebase Extensions の 1 つで、Firebase 経由のメール送信を簡単に実装できるようにするものです。
後から説明しますが、ダッシュボードで項目を入力して、数回ボタンを押すだけで Cloud Functions が自動的に作られます。

動作の流れは下記のようなイメージです。

1. Firebase Trigger Email と連携させた Firestore コレクションに特定の構造でデータを追加する
2. そのデータ追加を検知して Firebase Trigger Email が起動
3. 事前に設定していた SMTIP サーバー（SendGrid, mailgun...etc）と通信してメールを送信
4. メールの送信結果を Firestore に記録


# Firebase Trigger Emailの設定

最初に Firebase Project と Trigger Email の設定を行います。
Firebase で新規プロジェクトを作成して左下の Extensions を開きます。

すると Extensions の一覧が出るので、Trigger Email のインストールボタンを押しましょう。

![](https://storage.googleapis.com/zenn-user-upload/4ng4yva3bopk7k32x9khofr1mglm)

すると、構成内容の入力フォームが表示されるので適宜入力します。

![](https://storage.googleapis.com/zenn-user-upload/rhd3kkz8sz64iuj624u7o53accs2)

重要なのは、SMTP Connection URI です。

ここで指定した先が実際のメール送信を行います。ここには SendGrid や Mailgun などのメールサービスが使えます。今回は SendGrid を使います。

SendGrid の SMTP URI は `Email API > Integration Guide > SMTP Relay` から API キーを作成することで取得出来ます。

以下画面の各値を組み合わせた値が SMTP URI となります。

```
// smtps://[user]:[password]@[server]
smtps://apikey:hogehoge@smtp.sendgrid.net
```

![](https://storage.googleapis.com/zenn-user-upload/o4463xeis302mbu2dyjuh62vnjh1)



:::message
SMTP の詳細についてはこちらの SendGrid の記事が分かりやすいです。
https://sendgrid.kke.co.jp/blog/?p=636
:::

# Firebase Cloud Functionsの作成

# vue

# smtp uri
smtps://apikey:[Password]@smtp.sendgrid.net

# 参考
- https://sendgrid.kke.co.jp/blog/?p=636
- https://medium.com/firebase-tips-tricks/how-to-send-e-mails-using-firebase-extensions-a10d7cd685c2
