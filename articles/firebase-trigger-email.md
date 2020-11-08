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

SendGridのSMTP URIは `Email API > Integration Guide > SMTP Relay` からAPIキーを作成することで取得出来ます。

以下画面の各値を組み合わせた値がSMTP URIとなります。

```
// smtps://[user]:[password]@[server]
smtps://apikey:hogehoge@smtp.sendgrid.net
```

![](https://storage.googleapis.com/zenn-user-upload/o4463xeis302mbu2dyjuh62vnjh1)



:::message
SMTP の詳細についてはこちらの SendGrid の記事が分かりやすいです。
https://sendgrid.kke.co.jp/blog/?p=636
:::


# SendGridからSMTP URIの取得

# Firebase Cloud Functionsの作成

# vue

問い合わせフォーム部分の実装はこちらです。
送信後の通知部分の実装があるので、少し長いです。

```html
<template>
  <div class="form-wrapper box has-text-left">
    <div class="field">
      <label class="label">お名前</label>
      <div class="control">
        <input
          class="input"
          type="text"
          placeholder="山田 太郎"
          v-model="form.name"
        />
      </div>
    </div>

    <div class="field">
      <label class="label">メールアドレス</label>
      <div class="control">
        <input
          class="input"
          type="text"
          placeholder="example@example.com"
          v-model="form.email"
        />
      </div>
    </div>

    <div class="field">
      <label class="label">内容</label>
      <div class="control">
        <textarea
          class="textarea"
          placeholder="お問い合わせ内容"
          v-model="form.content"
        />
      </div>
    </div>

    <div class="field">
      <div class="control" @click="onSubmit">
        <button class="button is-link is-fullwidth">送信</button>
      </div>
    </div>
    <Notification
      :visible="notification.visible"
      :message="notification.message"
      :color="notification.color"
      @close="notification.visible = false"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive } from "@vue/runtime-core";
import { db, FieldValue } from "@/lib/firebase";
import Notification from "@/components/Notification.vue";

export default defineComponent({
  components: {
    Notification
  },
  setup() {
    const form = reactive({
      name: "",
      email: "",
      content: ""
    });

    const notification = reactive({
      visible: false,
      message: "",
      color: ""
    });

    const resetForm = () => {
      form.name = "";
      form.email = "";
      form.content = "";
    };

    const showErrorMessage = () => {
      notification.visible = true;
      notification.color = "danger";
      notification.message =
        "送信に失敗しました。時間をおいてもう一度お試しください。";
    };

    const showSuccessMessage = () => {
      notification.visible = true;
      notification.color = "primary";
      notification.message = "送信完了しました";
    };

    const onSubmit = async () => {
      const collectionRef = db.collection("contact_form");

      try {
        await collectionRef.add({
          name: form.name,
          email: form.email,
          content: form.content,
          createdAt: FieldValue.serverTimestamp()
        });
        showSuccessMessage();
        resetForm();
      } catch (_e) {
        showErrorMessage();
      }
    };

    return {
      form,
      notification,
      onSubmit
    };
  }
});
</script>

<style scoped lang="scss">
.form-wrapper {
  width: 500px;
  margin: auto;
}
</style>
```

大事なところは以下送信ボタン押下での処理です。
Firestore の contact_form コレクションに入力データを保存しています。

```ts
const onSubmit = async () => {
  const collectionRef = db.collection("contact_form");

  try {
    await collectionRef.add({
      name: form.name,
      email: form.email,
      content: form.content,
      createdAt: FieldValue.serverTimestamp()
    });
    showSuccessMessage();
    resetForm();
  } catch (_e) {
    showErrorMessage();
  }
};
```
# firebase

```bash
$ firebase init
```

```ts:functions/src/index.ts
import * as functions from 'firebase-functions';
import * as admin from 'firebase-admin';

admin.initializeApp()
const db = admin.firestore()

type FormPayload = {
  name: string;
  email: string;
  content: string;
}

const adminMailBody = (params: FormPayload) => {
  return `
以下内容で問い合わせフォームよりお問い合わせを受けつけました。

お名前:
${params.name}

メールアドレス:
${params.email}

内容:
${params.content}
`
}

const thanksMailBody = (params: FormPayload) => {
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
`
}

export const sendMail = functions.region("asia-northeast1")
  .firestore.document('/contact_form/{id}').onCreate(async (snapshot, _context) => {
  const { name, email, content } = snapshot.data()
  if(!email) { return }

  const adminMailData = {
    to: functions.config().mail.admin_address,
    message: {
      subject: 'ホームページお問い合わせ',
      text: adminMailBody({ name, email, content }),
    },
  }

  const thanksMailData = {
    to: email,
    message: {
      subject: 'お問い合わせありがとうございます',
      text: thanksMailBody({ name, email, content }),
    },
  }

  await db.collection('mail').add(adminMailData)
  await db.collection('mail').add(thanksMailData)
})


```

# smtp uri
smtps://apikey:[Password]@smtp.sendgrid.net

# 参考
- https://sendgrid.kke.co.jp/blog/?p=636
- https://medium.com/firebase-tips-tricks/how-to-send-e-mails-using-firebase-extensions-a10d7cd685c2