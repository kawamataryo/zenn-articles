---
title: "Vue3リリース間近!!「UIT meetup vol.10 Vue 三昧」参加レポート"
emoji: "📝"
type: "tech"
topics: ["vue", "勉強会", "JavaScript", "TypeScript"]
published: true
---

2020/09/18 19:00より開催された「UIT meetup vol.10 Vue三昧」の参加レポートです。

@[youtube](EjFKRr2ArFQ)


# UIT Quiz
![](https://storage.googleapis.com/zenn-user-upload/co50rpfwyum3ulcf1jswqpssqqcq)

**👨‍💻 発表者:** [@spring_raining](https://twitter.com/spring_raining)
**📷 動画開始位置:** https://youtu.be/EjFKRr2ArFQ?t=302

:::message
QRコードからLINEアプリを開いて行う視聴者参加型のクイズでした。
新しい試みで面白かったです! ただ、問題進行の同期制御がなかなか大変そうでした。（自分の環境だとなかなか上手く次の問題に行きませんでした..）
:::

# Vue3直前Q&A「kazuponさんに聞くVue3」前編

**👨‍💻 発表者:** [@kazu_pon](https://twitter.com/kazu_pon)
**📷 動画開始位置:** https://youtu.be/EjFKRr2ArFQ?t=968

:::message
@spring_rainingさんが事前に募集した質問を@kazu_ponさんに質問していくインタビュー形式のコンテンツでした。どの質問も気になっていたものだったので、回答を聞けて良かったです。
特にマイグレーションで使えるツールのvue-codemodは初耳だったので、調べてみようと思っています。
:::

（以下内容メモ）

### 1. Composition APIと今までのAPIとの使い分けは？
- コンポーネント間でのロジックの共有のために使う。2.xのoptionスタイルでそれを行うためにはMixinを使うしかなかったが、ネームスペースの問題などがあった。Composition APIなら従来のJavaScriptの構文でロジックを共通化できる
- optionスタイルをどういった時に使う？
  - JavaScriptのスキルが求められる
  - Composition APIは設計スキルが、optionスタイルの方が良い
- 競合しないのかというと競合する
  - 解決潤としては先にOptionスタイルが処理されて、その後にsetupが処理される
  - setupで上書きされる

### 2. Composition APIにおけるVuexとの親和性は？
- Composition APIとVuexは使える
- Vuex4ではVue3のAPIとの互換性を持つ。Composition APIと一緒に使える
- Vuex5のロードマップもすでに出ている。今はデザインレベルの話まで。実装はこれから。

### 3. もしVue3を新規プロジェクトで始めるとしたらVuexを導入すべきかどうか？
- 小さい規模で使う場合は、オーバーヘッドになるので使うべきかというと難しい。Composition APIの方が良い場合もある
- VuexはSSRの時の状態のハイドレーションが出来る。SSRを使う場合はVuex or Nuxtを使うのが良い

### 4. Vue3の共有ステートの管理は何を使うべき？
- 状態管理の最適解は、アプリケーションの規模と、チームの規模によって変わる
- 3つの状態管理方法がある
  - Vuex
    - VuexはFluxでレールが決まってるので規約が楽。DevToolsでデバッグができる。SSR時のハイドレーションに対応
    - TypeScriptの型が弱い。データフローが冗長になる
  - Vue Observable + Provide/Inject
    - 開発のガイドラインをチームで決める必要があるが自分たちに合わせてカスタマイズできる
    - Provide/Injectを使えばDIで疎結合を保てる
    - DevToolsが対応してないのは不便。SSRのハイドレーションは不便
  - Composition API + Provide/Inject
    - Observableとほぼ同じ
    - Vue3にマイグレーションしやすいのは利点
- Vuex5はTypeScriptの4.1の機能で変わってくる。完全なTS化も出来そう

### 5. `<script setup>`とサードパーティライブラリの対応
- `<script setup>`は実験的機能
- JavaScriptの機能をハックしているので、気持ち悪いと感じる人もいそうではある

### 6. Vue2からのマイグレーションをする際の戦略は？
- Vue2.7にVue3の機能をバックポートしている
- [vue-codemod](https://github.com/vuejs/vue-codemod)でマイグレーションを実行してから、 Vue2.7を入れて修正、Vue3に対応させるのが良い


# （がんばらない）Vue3移行


![](https://storage.googleapis.com/zenn-user-upload/g76kcais0l5p486wu03xe691oxdu)

**📷 動画開始位置:** https://youtu.be/EjFKRr2ArFQ?t=3621
**📁 資料:** -（見つけたら追加します）
**👨‍💻 発表者:** [@nhayashida](https://twitter.com/nhayashida)
**📝 内容メモ:**
- LINE LIVEではVue3を利用中
- esllint-plugin-vue@nextを入れて、Vue3で廃止される機能を使うとwarningを出すようにしてた
- vue-codemodも利用
- 移行完了までComposition APIの利用は我慢する

:::message
まずLINE LIVEでプロダクトとしてVue3を使ってるということに驚きでした。
esllint-plugin-vue@next, vue-codemodはかなり使えそうですね。eslint-plugin-vue@nextは早速プロダクトでも使っていきたいと思いました。あと、Composition APIをあえて入れないというのはなかなかキモだと思いました。
:::

# Vue Router Next ~ 意外と語られないVue3時代のルーティング ~

![](https://storage.googleapis.com/zenn-user-upload/6fy8l618jqhqg5kd0gul8f17dpwn)

**📷 動画開始位置:** https://youtu.be/EjFKRr2ArFQ?t=4540
**📁 資料:** -（見つけたら追加します）
**👨‍💻 発表者:** [@NozomuIkuta](https://twitter.com/NozomuIkuta)
**📝 内容メモ:**
- Vue Routerは実は大型アップデート。魅力的なアップデートである
- useRouteでrouteの情報を取得できる
- モーダル表示状態をネイティブにヒストリーに残せる Router view route prop

:::message
Vue Router Nextの変更は全く追ってなかったので勉強になりました。
特にuseRouteでのsetup関数内でのroute情報の取得と、Router view route propでのモーダル表示状態をヒストリーに残せる機能が良いですね。他にも便利な機能がたくさんありそうなので、RFCを読んでみようと思いました。
:::

# Vue3直前Q&A「kazuponさんに聞くVue3」後編

![](https://storage.googleapis.com/zenn-user-upload/vl71c5lo33j5afp3f4fvrwmsm3yt)

**📷 動画開始位置:** https://youtu.be/EjFKRr2ArFQ?t=5662
**👨‍💻 発表者:** [@kazu_pon](https://twitter.com/kazu_pon)

:::message
最後にまたUITのLINEアプリを使って、今度は@kazu_ponさんから視聴者へのアンケートを実施しました。
Vue3をすぐ使うかどうかのアンケートから、IE11への対応アンケートまで幅広い内容でした。自分の回答結構マイノリティ側だったので新鮮でした。
:::

# 終わりに
以上、「UIT meetup vol.10 Vue 三昧」参加レポートでした。
いやー面白かったです。運営の皆様ありがとうございました。
今夜[VueJSGlobal](https://vuejs.amsterdam/)にて（おそらく）Vue3が発表されるので、そちらも必見です。

** 2020/09/19 6:04追記**

Vue3リリースされました〜!!まさかのOne PIece!!

@[tweet](https://twitter.com/vuejs/status/1306992969728380930)