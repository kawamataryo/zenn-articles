---
title: "Firebase Trigger EmailでSPAサイトのお問い合わせフォームを作る"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "vue", "typescript"]
published: false
---

この記事は[Firebas Advent Calendar 2020](https://qiita.com/advent-calendar/2020/firebase) 18 日目の記事です。

SSG や SPA サイトを構築する際に地味に迷うお問い合わせフォームの実装で、 Firebase Trigger Email を使う方法を解説します。


# 何を作る？

こんな感じの問い合わせフォームを作ります。

@[codesandbox](https://codesandbox.io/embed/cotactform-sample-rdqs3?fontsize=14&hidenavigation=1&theme=dark)
※ こちらの例は、Firebase の env の設定をしていないため、送信ボタンを押しても失敗します。

お問い合わせを実行すると、管理者のメールアドレス宛に問い合わせ内容と、フォーム記載のメールアドレス宛に thanks メールが送られます。

技術スタックは以下の通りです。

- フロントエンド
  - TypeScript
  - Vue.js
- バックエンド
  - Firebase Trigger Email
  - Cloud Functions for Firebase
  - Firestore
  - SendGrid


![](https://i.gyazo.com/82094b5dd251a0beee46b2e25a019fce.png)

以下で解説するコードは全て以下リポジトリにあります。
もしタイポで動かないなどあればこちらをご確認ください。

https://github.com/kawamataryo/contact-form-with-firebase-trigger-email

# Firebase Trigger Emailとは？

最初に簡単に今回の主役である Firebase Trigger Email について。
Firebase Trigger Email は Firebase Extensions の 1 つで、Firebase 経由のメール送信を簡単に実装できるようにするものです。
後から説明しますがコンソールで項目を入力するだけでメール送信の Cloud Functions が自動的に作られます。

Firebase Trigger Email の動作の流れはこちらです。

1. Firebase Trigger Email と連携させた Firestore コレクションに特定の構造でデータを追加する
2. そのデータ追加を検知して Firebase Trigger Email が起動
3. 事前に設定していた SMTIP サーバー（SendGrid, mailgun...etc）と通信してメールを送信
4. メールの送信結果を Firestore に記録


# Firebase Trigger Emailの設定

Firebase で新規プロジェクトを作成してメニュー左下の Extensions を開きます。

Extensions の一覧が出るので、Trigger Email のインストールボタンを押しましょう。

![](https://storage.googleapis.com/zenn-user-upload/4ng4yva3bopk7k32x9khofr1mglm)

構成内容の入力フォームが表示されるので適宜入力します。

![](https://storage.googleapis.com/zenn-user-upload/rhd3kkz8sz64iuj624u7o53accs2)

|項目|内容|
|---|---|
|Cloud Functions location|Trigger Email の Cloud Functions を配置する region|
|SMTP connection URI| 実際にメール送信を担うメールサービスの SMTP URI|
|Email documents collection| メール送信の対象となるデータを挿入するコレクション（このコレクションへの挿入がキックでメールが送信される|
|Default FROM address|配信メールに送信者として記載されるメールアドレス（optional）|
|Default REPLY-TO address|配信メールに返信先として記載されるメールアドレス（optional）|
|Users collection|email のドキュメントの付加情報として、ユーザ情報を使う場合の Collection（optional)|
|Templates collection|メールテンプレートを使う場合にそのテンプレートを保存する Collection（optional)|

重要なのは、SMTP connection URI です。

ここに指定した先で実際のメール送信が行われます。SendGrid や Mailgun などのメールサービスが指定できます。

今回は SendGrid を使います。
SendGrid の SMTP connection URI は `Email API > Integration Guide > SMTP Relay` から API キーを作成することで取得出来ます。

以下画面の各値を組み合わせた値が SMTP URI となります。

![](https://storage.googleapis.com/zenn-user-upload/o4463xeis302mbu2dyjuh62vnjh1)

```
// smtps://[user]:[password]@[server]
smtps://apikey:hogehoge@smtp.sendgrid.net
```


:::message
`Default FROM address` のアドレスは SengGrid 側で独自ドメインとして登録しないとメールソフトによっては迷惑メールに振り分けられる場合があります。
プロダクトで使うのならば必ず独自ドメインの設定を行いましょう。設定方法は以下リンクで解説されています。
https://sendgrid.kke.co.jp/docs/Tutorials/D_Improve_Deliverability/using_whitelabel.html
:::

拡張機能をインストールを押すしてしばらく待つと Functions に新しい関数が追加されます。
これで Trigger Email の設定は完了です。

![](https://i.gyazo.com/82094b5dd251a0beee46b2e25a019fce.png)

# Firestoreにデータを挿入するFunctionsの作成

次に、先ほど指定した Email documents collection（Firebase Trigger Email の起動へのフックとなる Collection) へのデータ挿入のための Functions を作成します。
直接クライアントから Firestore にデータを挿入する形でも良いのですが、今回は Template を一律に管理したい、自動返信メールも送信したいという考えがあるので Functions で管理し、クライアントからは httpsCallable 関数を呼ぶ形にします。

Functions のコードは以下の通りです。

```ts:functions/src/index.ts
import * as functions from "firebase-functions";
import * as admin from "firebase-admin";
import { adminMailBody, thanksMailBody } from "./lib/mailBody";

admin.initializeApp();
const db = admin.firestore();

export const sendMail = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    const { name, email, content } = data;
    if (!email) {
      throw new functions.https.HttpsError(
        "invalid-argument",
        "email is required"
      );
    }

    const adminMailData = {
      to: functions.config().mail.admin_address,
      message: {
        subject: "ホームページお問い合わせ",
        text: adminMailBody({ name, email, content }),
      },
    };

    const thanksMailData = {
      to: email,
      message: {
        subject: "お問い合わせありがとうございます",
        text: thanksMailBody({ name, email, content }),
      },
    };

    await db.collection("mail").add(adminMailData);
    await db.collection("mail").add(thanksMailData);
  });
```

メールの本文を作る関数はこちらです。

```ts:functions/src/lib/mailBody.ts
type FormPayload = {
  name: string;
  email: string;
  content: string;
};

export const adminMailBody = (params: FormPayload) => {
  return `
以下内容で問い合わせフォームよりお問い合わせを受けつけました。

お名前:
${params.name}

メールアドレス:
${params.email}

内容:
${params.content}
`;
};

export const thanksMailBody = (params: FormPayload) => {
  return `
${params.name} 様

お問い合わせありがとうございます。
以下内容でお問い合わせを受け付けました。

お名前:
${params.name}

メールアドレス:
${params.email}

内容:
${params.content}

後ほど担当者よりご連絡を差し上げます。
よろしくお願いいたします。
`;
};
```

この関数をデプロイすれば、Firebase 側の準備は完了です。

# お問い合わせフォームとの繋ぎ込み

最後に、お問い合わせフォームの送信ボタン押下で先ほど作った Functions を実行すれば OK です。

詳細は [CodeSandbox](https://codesandbox.io/embed/cotactform-sample-rdqs3?fontsize=14&hidenavigation=1&theme=dark) をみてもらうとして、該当コードだけ抜粋します。

送信ボタン押下時の処理です。reactive で保持していたフォームの入力値を引数に、httpsCallable 関数を実行しています。

```ts
const onSubmit = async () => {
  disabled.value = true;
  if (!(form.name && form.email && form.content)) {
    showMessage("danger", "必須内容が入力されていません");
    disabled.value = false;
    return;
  }

  try {
    const sendMail = functions.httpsCallable("sendMail");
    await sendMail({
      name: form.name,
      email: form.email,
      content: form.content
    });

    showMessage("primary", "送信が完了しました");
    resetForm();
  } catch (_e) {
    showMessage(
      "danger",
      "送信に失敗しました。時間をおいてもう一度お試しください。"
    );
  } finally {
    disabled.value = false;
  }
};
```

これで送信ボタンが押されると Functions により Firebase Trigger Email と連携する Firestore にデータが挿入され、管理者とお問い合わせ実行ユーザーにメールが送信されます。
完成🎉

# おわりに

以上、「Firebase Trigger Email で SPA サイトのお問い合わせフォームを作る」でした。
Firebase Trigger Email メール送信部分を抽象化してくれるので とても使い勝手が良いなーと思いました。

# 参考
- [SMTPリレーサービス【入門】 | SendGridブログ](https://sendgrid.kke.co.jp/blog/?p=636)
- [How to send E-mails using Firebase Extensions | by Gastón Saillén | Firebase Tips & Tricks | Medium](https://medium.com/firebase-tips-tricks/how-to-send-e-mails-using-firebase-extensions-a10d7cd685c2)
