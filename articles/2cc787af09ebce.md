---
title: "Astro + Vercel Serverless FunctionsでのreCAPTCHA v3の導入例"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["astro", "vercel", "recaptcha"]
published: false
---

AstroのHPにreCAPTCHA v3を導入する機会があったので、備忘録として残しておきます。


# reCAPTCHA v3とは？
お問合せフォームでよく見かけるアレです。フォームが設置されているページに導入することで自動化されたスパムの送信を防ぐことができます。

![](/images/2cc787af09ebce/2023-09-07-15-32-08.png)

reCAPTCHA v3ではバックグラウンドでユーザーの動きを検証しスコアを判定するので、「私はロボットではありません」のチェックボックスやイライラする画像の選択などの操作は不要になります。

https://developers.google.com/recaptcha/docs/v3




# 実装例

Astro + Vercel Serverless Functionsでの実装例を紹介します。

今から説明するコードは、すべて以下リポジトリにあります。

https://github.com/kawamataryo/astro-recaptcha-sampler

デモサイトもこちらで公開しているので、実際の動きも確認できます。

https://astro-recaptcha-sample.vercel.app/

![](/images/2cc787af09ebce/2023-09-07-15-11-00.png)

## reCAPTCHA v3の設定

reCAPTCHA v3のコンソールにログインして、新しいサイトを登録し、サイトキーとシークレットキーを取得します。
ドメインには、reCAPTCHAを導入するサイトのドメインを入れてください。

https://www.google.com/recaptcha/admin/create

![](/images/2cc787af09ebce/2023-09-07-14-57-40.png)

サイトキーとシークレットキーをメモしておきます。

![](/images/2cc787af09ebce/2023-09-07-14-59-31.png)


## Server 側での実装

Server側は、AstroのServer Endpointsを使って実装します。

`src/pages/api/`に`recaptcha.ts`を作成します。


https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/pages/api/recaptcha.ts#L1-L18

シークレットキーとフロントエンドから受け取ったreCAPTCHAのトークンを使って、スコアを取得しています。

地味にポイントなのが、`"Content-Type": "application/x-www-form-urlencoded"`というheaderとrequestBodyの形式です。普通にJSONで送ると、`missing-input-response', 'missing-input-secret'`というエラーが出てしまうの注意です。

参考: https://stackoverflow.com/questions/52416002/recaptcha-error-codes-missing-input-response-missing-input-secret-when-v


## Client 側での実装

Client側では、reCAPTCHAのライブラリを読み込んで、Form送信時に先ほど追加したAPIへreCAPTCHAのトークンを送信し、スコアを取得します。

コード的には送信ボタン押下時の関数内で `grecaptcha.ready` にてreCAPTCHAの読み込みを待ってから、`grecaptcha.execute`でトークンを取得し、それをAPIに送信、スコアを取得しています。

今回のコードでは、結果のスコアをバナーの表示に使っていますが、実際の運用ではここでフォーム送信の可否を判定するといった使い方をすると思います。

**Astroコンポーネントでの例**

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/components/ContactForm.astro#L1-L71

**Reactコンポーネントでの実装例**

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/components/ContactForm.tsx#L1-L101


## Vercelへのデプロイ

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

https://github.com/kawamataryo/astro-recaptcha-sampler/blob/main/src/pages/api/recaptcha.ts#L1C1-L1C32

これで、あとは通常通りVercelにデプロイすればOKです。
静的ファイルと、Vercel Serverless Functionsがデプロイされます。

![](/images/2cc787af09ebce/2023-09-07-15-24-12.png)

おわり。
