---
title: "Firebase Trigger Email で作る SPA サイトのお問い合わせフォーム"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "vue", "typescript"]
published: false
---

この記事は [Firebas Advent Calendar 2020](https://qiita.com/advent-calendar/2020/firebase) 18 日目の記事です。

SSG や SPA サイトを構築する際に迷うのがお問い合わせフォームの実装です。
今回は Firebase Trigger Email を使って手軽に実装する方法を解説します。


# 何を作る？

こんな感じのお問い合わせフォームを作ります。

@[codesandbox](https://codesandbox.io/embed/cotactform-sample-rdqs3?fontsize=14&hidenavigation=1&theme=dark)

フォームを入力し送信ボタンを押下すると、管理者とフォーム記載のメールアドレス宛に以下メールが送られます。
（CodeSandbox はサンプルです。送信ボタンを押してもメールは送信されません）

- 管理者側
![](https://i.gyazo.com/2b298a112f11b05e5b33574bcaf1ac66.png)


- お問い合わせ実行ユーザー側
![](https://i.gyazo.com/477b55b4610bce467a85446b3370dd0a.png)

技術スタック及び構成は以下の通りです。

![](https://i.gyazo.com/82094b5dd251a0beee46b2e25a019fce.png)

これから解説するコードは全てこちらのリポジトリにあります。

https://github.com/kawamataryo/contact-form-with-firebase-trigger-email

# Firebase Trigger Email とは？

Firebase Trigger Email は Firebase Extensions の 1 つで、Firebase でメール送信を簡単に実装できるようにするものです。
後から説明しますがコンソールで項目を入力するだけでメール送信の Cloud Functions が自動的に作られます。

Firebase Trigger Email の動作の流れはこちらです。

1. Firebase Trigger Email と連携させた Firestore コレクションに特定の構造でドキュメントを追加する
2. ドキュメントの追加をフックに Firebase Trigger Email が起動
3. 事前に設定していた SMTP サーバー（SendGrid, mailgun...etc）と通信してメールを送信
4. メールの送信結果で ドキュメント をアップデート（送信履歴の記録）


# Firebase Trigger Email の設定

Firebase で新規プロジェクトを作成して、メニュー左下の Extensions を開きます。
Extensions の一覧が出るので、Trigger Email のインストールボタンを押しましょう。

![](https://storage.googleapis.com/zenn-user-upload/4ng4yva3bopk7k32x9khofr1mglm)

構成内容の入力フォームが表示されるので適宜入力します。

![](https://storage.googleapis.com/zenn-user-upload/rhd3kkz8sz64iuj624u7o53accs2)

|項目|内容|
|---|---|
|Cloud Functions location|Firebase Trigger Email の Cloud Functions を配置する region|
|SMTP connection URI| 実際にメール送信を担うメールサービスの SMTP URI|
|Email documents collection| メール送信の対象となるデータを挿入するコレクション（このコレクションへの挿入がキックでメールが送信される|
|Default FROM address|配信メールに送信者として記載されるメールアドレス（optional）|
|Default REPLY-TO address|配信メールに返信先として記載されるメールアドレス（optional）|
|Users collection|email のドキュメントの付加情報として、ユーザ情報を使う場合のコレクション（optional)|
|Templates collection|メールテンプレートを使う場合にそのテンプレートを保存するコレクション（optional)|

重要なのは、`SMTP connection URI` です。

ここに指定した先で実際のメール送信が行われます。[SendGrid](https://sendgrid.com/) や [Mailgun](https://www.mailgun.com/) などのメールサービスが指定できます。

今回は SendGrid を使います。
SendGrid の `SMTP connection URI` は ユーザーページの `Email API > Integration Guide > SMTP Relay` から API キーを作成することで取得出来ます。

以下画面の各値を組み合わせたものが SMTP URI となります。

![](https://storage.googleapis.com/zenn-user-upload/o4463xeis302mbu2dyjuh62vnjh1)

```
// smtps://[user]:[password]@[server]
smtps://apikey:hogehoge@smtp.sendgrid.net
```


拡張機能をインストールを押すしてしばらく待つと Functions に新しい関数が追加されます。
これで Trigger Email の設定は完了です。

:::message
Firebase Trigger Email の設定の `Default FROM address` に入力するアドレスを SengGrid 側で独自ドメインとして登録しないと、メールソフトによっては迷惑メールに振り分けられる場合があります。
プロダクトで使うのならば、必ず独自ドメインの設定を行いましょう。設定方法は以下リンクで解説されています。
https://sendgrid.kke.co.jp/docs/Tutorials/D_Improve_Deliverability/using_whitelabel.html
:::


# Functions の作成

次に、先ほど指定した `Email documents collection`（Firebase Trigger Email の起動へのフックとなるコレクション）へのドキュメント追加のための Functions を作成します。
直接クライアントから Firestore にドキュメントを追加する形でも良いのですが、今回は同時に thanks メールも送りたいので、ドキュメント追加は Functions で管理しクライアントからは httpsCallable 関数を呼ぶ形にします。

Functions のコードはこちらです（Functions のプロジェクト全体は[こちら](https://github.com/kawamataryo/contact-form-with-firebase-trigger-email/tree/master/functions)）。

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

特に重要なのは `mail` コレクションにデータを追加している以下の部分です。

```ts
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
```

`mail`コレクションのドキュメントとして`to`フィールドに送信したいメールアドレス、 `message`フィールドの Map の`subject`に件名、`text`にメール本文を設定しています。
これが Firebase Trigger Email の規約です。
この構造を持つドキュメントを`mail`コレクションに追加するとメール送信がキックされます。

関数を Functions にデプロイすれば、Firebase 側の準備は完了です。

# お問い合わせフォームとの繋ぎ込み

お問い合わせフォームの送信ボタン押下で、先ほど作った Functions を実行させます。
お問い合わせフォームのコンポーネントはこちらです。

```vue
<template>
  <div class="form-wrapper box has-text-left has-background-light">
    <div class="field">
      <label class="label">お名前<span class="has-text-danger">※</span></label>
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
      <label class="label"
        >メールアドレス<span class="has-text-danger">※</span></label
      >
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
      <label class="label">内容<span class="has-text-danger">※</span></label>
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
        <button class="button is-link is-fullwidth" :disabled="disabled">
          送信
        </button>
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
import { defineComponent, reactive, ref } from "@vue/runtime-core";
import { functions } from "@/lib/firebase";
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
    const disabled = ref(false);

    const resetForm = () => {
      form.name = "";
      form.email = "";
      form.content = "";
    };

    const showMessage = (color: "danger" | "primary", message: string) => {
      notification.visible = true;
      notification.color = color;
      notification.message = message;
    };

    const onSubmit = async () => {
      disabled.value = true;
      if (!(form.name && form.email && form.content)) {
        showMessage("danger", "必須内容が入力されていません");
        disabled.value = false;
        return;
      }

      try {
        // お問い合わせフォームの送信
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

    return {
      form,
      notification,
      disabled,
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

重要なのは送信ボタン押下時の処理です。
reactive で保持していたフォームの入力値を引数に、httpsCallable 関数を実行しています。

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

# 参考
- [SMTPリレーサービス【入門】 | SendGridブログ](https://sendgrid.kke.co.jp/blog/?p=636)
- [How to send E-mails using Firebase Extensions | by Gastón Saillén | Firebase Tips & Tricks | Medium](https://medium.com/firebase-tips-tricks/how-to-send-e-mails-using-firebase-extensions-a10d7cd685c2)
