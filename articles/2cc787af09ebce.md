---
title: "Astro + Vercel Serverless FunctionsでのreCAPTCHA v3の導入例"
emoji: "🏙️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["astro", "vercel", "recaptcha"]
published: true
---

Astroで作ったWebサイトにreCAPTCHA v3を導入する機会があったので、備忘録として残しておきます。


# reCAPTCHA v3とは？
お問合せフォームでよく見かけるアレです。

![](/images/2cc787af09ebce/2023-09-07-15-32-08.png)

フォームが設置されているページにreCAPTCHAを導入することで、スパムの送信を防ぐことができます。

また、reCAPTCHA v3ではバックグラウンドでユーザーの動きを検証しスコアを判定するので、「私はロボットではありません」のチェックボックスやイライラする画像の選択などの操作は不要になります。

https://developers.google.com/recaptcha/docs/v3




# 実装例

Astro + Vercel Serverless Functionsでの実装例を紹介します。
今から説明するコードは、すべて以下リポジトリにあります。

https://github.com/kawamataryo/astro-recaptcha-sampler

デモサイトも[こちら](https://astro-recaptcha-sample.vercel.app/)で公開しているので、実際の動きも確認できます。

https://astro-recaptcha-sample.vercel.app/

![](/images/2cc787af09ebce/2023-09-07-15-35-53.png)

## 1. reCAPTCHA v3の設定

reCAPTCHA v3のコンソールにて新しいサイトを登録し、サイトキーとシークレットキーを取得します。
ドメインには、reCAPTCHAを導入するサイトのドメインを入れてください。

https://www.google.com/recaptcha/admin/create

![](/images/2cc787af09ebce/2023-09-07-14-57-40.png)

サイトキーとシークレットキーをメモしておきます。

![](/images/2cc787af09ebce/2023-09-07-14-59-31.png)


## 2. Server 側での実装

Server側は、AstroのServer Endpointsを使って実装します。

`src/pages/api/`に`recaptcha.ts`を作成します。


https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/pages/api/recaptcha.ts#L1-L18

シークレットキーとフロントエンドから受け取ったreCAPTCHAのトークンを使って、スコアを取得しています。

:::message
地味にポイントなのが、`"Content-Type": "application/x-www-form-urlencoded"`というheaderとrequestBodyの形式です。普通にJSONで送ると、エラーが出てしまうの注意です。

参考: https://stackoverflow.com/questions/52416002/recaptcha-error-codes-missing-input-response-missing-input-secret-when-v
:::


## 3. Client 側での実装

Client側では、サイトキーをクエリパラメーターに設定した上で、reCAPTCHAのライブラリを読み込みます。

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/pages/index.astro#L9-L11

そして、送信ボタン押下時の関数内で `grecaptcha.ready` にてreCAPTCHAのスクリプトの読み込みを待ってから、`grecaptcha.execute`でトークンを取得し、それをAPIに送信、スコアを取得しています。

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/components/ContactForm.astro#L33-L67

今回のコードでは、結果のスコアをバナーの表示に使っていますが、実際の運用ではここでフォーム送信の可否を判定するといった使い方をすると思います。

以下Formの全コードです。AstroコンポーネントとReactコンポーネントの両方で実装してみました。

**Astroコンポーネントでの例**

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/components/ContactForm.astro#L1-L71

**Reactコンポーネントでの実装例**

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/components/ContactForm.tsx#L1-L101


## 4. Vercelへのデプロイ

Astroのリポジトリを、Vercelと連携させれば自動的にデプロイされるのですが、Server Endpointsを使う場合は事前準備が必要です。

以下をコマンドラインで実行し、AstroのVercelアダプターを追加します。

```
npx astro add @astrojs/vercel
```

`astro.config.mjs`を以下のように修正します。

```
import { defineConfig } from 'astro/config';
import vercel from "@astrojs/vercel/serverless";

import react from "@astrojs/react";
export default defineConfig({
  output: "hybrid",
  adapter: vercel(),
});
```

この`output: "hybrid"`という指定は、デフォルトでプリレンダリングするが、`export const prerender = false;`が設定されているファイルはサーバーサイドでリクエストごとにレンダリングするというものです。

https://docs.astro.build/en/reference/configuration-reference/#output

今回の例だとreCAPTCHAのAPI（`pages/api/recaptcha.ts`）についてはプリレンダリングしてほしくないので、ファイル先頭に`export const prerender = false;`を追加しています。

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/pages/api/recaptcha.ts#L1

あとは通常通りVercelにデプロイすればOKです。
静的ファイルと、Vercel Serverless Functionsがデプロイされます。

![](/images/2cc787af09ebce/2023-09-07-15-24-12.png)

おわり 🚀
