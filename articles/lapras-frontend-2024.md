---
title: "LAPRASのフロントエンド改善活動ふりかえり [2024年]"
emoji: "📘"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["frontend", "技術的負債", "ふりかえり"]
publication_name: "lapras_inc"
published: true
---

この記事は [LAPRAS Advent Calendar 2024](https://qiita.com/advent-calendar/2024/lapras) 17日目の記事です。

https://qiita.com/advent-calendar/2024/lapras


弁護士ドットコムさんの[こちら](https://creators.bengo4.com/entry/2024/12/23/000000)の記事を読んで、「リアルな改善活動が知れる良い記事！LAPRAS版も書いてみたい！」と思い、まとめてみました。

https://creators.bengo4.com/entry/2024/12/23/000000


# 📘 LAPRASのフロントエンド事情

最初にLAPRASのフロントエンド事情の紹介です。

|項目|内容|
|---|---|
|ファーストコミット|2015年|
|フロントエンド構成| TypeScript + Vue3 のシンプルなSPA|
|フロントエンド専任エンジニア| 0人|

LAPRASのエンジニア組織ではフロントエンド専任のエンジニアはいません。皆プロジェクト単位でバックエンドもフロントエンドも（ときにインフラも）触りながら開発しています。

ただそれだけだと個別領域の技術的負債の解消が進みにくいので、フロントエンドに関心のある有志のエンジニアがフロントエンド改善委員会という会を結成し、週1回1時間集まり改善活動を行っています。今回はその委員会メンバーで行ってきた改善活動を紹介します。

:::message
会の時間以外にも、LAPRASにはクリエイティブな時間（Googleの20%ルールのようなもの）があり、そちらの一部も使っています。

- [LAPRASにおける20%ルール的なものとその心｜rocky_manobi](https://note.com/rocky_manobi/n/n619be588790b)
:::


# 🐉 今年行ったフロントエンドの改善活動

時系列、規模は順不同で紹介します。

## JestからVitestへの移行

LAPRASのテストは開発当初からJestでした。ただ、JestはES Modulesのサポートやパフォーマンスの問題があり、今年Vitestに移行しました。

結果CIのフロントエンドテスト実行時間が約1/3になり、リリースのリードタイムの短縮にも繋がりました。ES Modulesのパッチも必要なくなり、新しいライブラリに関するテストの導入も容易になりました。

https://vitest.dev/

## SWRV、VuexからTanstackQueryへの移行

LAPRASでは非同期状態管理ライブラリとして[SWRV](https://docs-swrv.netlify.app/)を昨年から試験的に導入していました。しかし、SWRVのメンテナンスが1年以上停滞していることや、SWRVの機能不足を感じて、[TanstackQuery](https://tanstack.com/query)へ移行しました。

また、内部で広く使われていたVuexでの状態管理も、stale-while-revalidateのキャッシュ戦略によるパフォーマンス改善や実装の複雑さの回避のため、TanStackQueryでの非同期状態管理へ移行すべく今改善を進めています。

https://tanstack.com/query

## TypeScript 型エラーの継続的な解消

LAPRASは開発初期からTypeScriptを導入していたわけではなく、途中でJavaScriptからTypeScriptへの移行を行いました。そのため、まだすべてのフロントエンドコードが型で守られているわけではありません。

とくにVueのtemplate部分などに型エラーが多く、そちらの改善も継続して行いました。また、漸進的に改善を進めるためCIでPRごとに型エラーの差分を出して、型エラーが新たに増えた場合はコメントする仕組みをGitHub Actionsで整備しました。

https://www.typescriptlang.org/

## webpackからRspackへの移行

ビルド時間やHMRが遅いという問題の解消のため、webpackから[Rspack](https://rspack.dev/)へ移行しました。

Rspackは今年メジャーバージョンがリリースされたばかりの新しいビルドツールで、導入にはまりどころもありましたが、今はほぼ問題なく動いています。ビルド時間も約1/2になり、HMRも一瞬になりました。

まだ主要プロダクトのひとつはwebpackのままので、2025年にはそちらもRspackへの移行したいと思っています。

https://rspack.dev/

移行の経緯や導入時のはまりどころについては以下で詳しく書いています。

https://www.docswell.com/s/kawamataryo/582VNY-webpack-to-rspack


## yarn 1系から npm への移行

パッケージマネージャーとして長らくyarn1系を使っていましたが、依存関係インストール時のパフォーマンスが悪いこと、yarn2,3,4系へのバージョンアップがつらいことから、npmへの移行を検討しています。

ただ、yarnのモジュール解決に依存しているパッケージがあり（非常に良くない・・）、単純にnpm installするだけでは動かず、まだ移行できていません。本腰を入れればすぐできると思うので、2025年には移行したいと思っています。

https://yarnpkg.com/

## script setupの布教

Vue2系の頃からVueを使っているので、プロダクトにはOption APIやComposition API、Composition API（script setup）とさまざまなVue SFCの書き方が混在しています。

その状況を改善し、Vue3 + TypeScriptの利点を最大限生かすべく、script setupの布教を行いました。script setup固有の書き方により問題が起こらないか、どのように改善をすすめていくかなどをCoding Guideにまとめました。

https://ja.vuejs.org/api/sfc-script-setup

## AxiosからFetchへの書き換え

Axiosを長らく使っているのですが、Axiosのバージョンが古いことやバンドルサイズの縮小のため、Fetchへの書き換えを行っています。こちらもまだテストケースをまとめている段階で道半ばですが、 2025年には完全移行したいと思っています。

https://developer.mozilla.org/ja/docs/Web/API/Fetch_API

## E2Eテストの導入

LAPRASのフロントエンドテストはVitestによるユニットテスト + [Percyによる簡易なVisual Regressionテスト](https://zenn.dev/ryo_kawamata/articles/percy-cypress) のみで、E2Eテストはありませんでした。

今年の初頭まではユニットテストの布教を行ってきたのですが、プロダクトの性質やテストを書くコストとテストで防げる不具合のバランス的に、 E2Eテストに主力をおいた方が良いのでは？という議論があり、今はE2Eテストの導入に向けて動いています。

現在はVitest Browser Modeによるコンポーネント単位のブラウザテストを試験的に導入して、効果や手触りを検証しています。

https://vitest.dev/guide/browser/

:::message
当初はバックエンド（Django）も一気通貫した本当のE2Eテストを導入しようとしていました。しかし、バックエンドのDBのデータの初期化をPlaywright側から制御する難しさや、CI実行時間の問題、Flakyなテストの問題などがあり、そちらは断念しました。
:::

## Core Web Vitalsの改善

[mizchiさんの公開パフォーマンスチューニング動画](https://www.youtube.com/watch?v=j0MtGpJX81E&ab_channel=LAPRAS%E5%85%AC%E5%BC%8F) の通り、LAPRASのCore Web Vitalsは悪いです。

ログイン必須の管理画面的なアプリといえど、エンジニアが使うサービスなのでパフォーマンスはできるだけ改善したいと思い改善を進めています。最低限2025年初頭には動画で指摘された問題は解消したいと思っています。

https://www.youtube.com/watch?v=j0MtGpJX81E&ab_channel=LAPRAS%E5%85%AC%E5%BC%8F

https://www.youtube.com/watch?v=rZmbnNn79LE&t=1722s&ab_channel=LAPRAS%E5%85%AC%E5%BC%8F

## 共通コンポーネントの整備（Storybookのアップデート、Viteへの移行）

LAPRASではB側とC側の2つの主要プロダクトがあり、そこで利用する汎用コンポーネントは[lapras-frontend](https://github.com/lapras-inc/lapras-frontend) というリポジトリで管理しています。そこのStorybookが長らく6系で、メンテナンス性が悪かったので8系へアップデートしました。またあわせて、Vue CLIのビルドからViteのビルドへの移行も行いました。

ただ、まだまだ使い勝手が良いとは言えないので、今後も引き続き改善を進めたいと思っています。

https://github.com/lapras-inc/lapras-frontend

https://storybook.js.org/blog/storybook-8/

## その他
その他細かい改善活動も並行して行ってきました。

- エラー表示のtoast化
- フォームデザインの統一
- CSSの整理（z-indexのルール決め・統合）
- CIの最適化（型チェックの改善、ESLintの最適化）
- ESLintのルール追加・修正
- LPサイトのVue導入
- Visual Regressionテストの改善
- Node.jsの継続的アップデートの仕組み作り
- 改善委員会自体の改善（稼働時間、タスクの進め方）
etc..


# 📝 おわりに

ふりかえってみると今年も色々やっていました。ただ改善したいこと山ほどあり、まだまだ道半ばです。

プロダクトの機能開発の合間に改善を進めるのは大変ですが、フロントエンドを良くすることでプロダクトのユーザー体験や、エンジニアの開発体験が向上し、周り回って事業の成長につながると信じて2025年も改善活動を続けていきたいと思います🦾

# 📣 宣伝
最後に宣伝です！

LAPRASでは今年もアウトプットのふりかえりキャンペーンを行っています。
ログインして、シェアボタンを押すだけで参加できるので、ぜひぜひご参加ください！LAPRASから松坂牛を送らせてください🐮

https://note.lapras.com/campaigns/202412_year-end/

![alt text](/images/image-6.png)
