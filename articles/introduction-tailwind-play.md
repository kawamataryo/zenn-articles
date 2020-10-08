---
title: "Tailwind Playでインタラクティブにデザインを構築する"
emoji: "🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TailwindCSS", "css", "WEBデザイン", "PostCSS"]
published: true
---

![](https://storage.googleapis.com/zenn-user-upload/9344ot2vnmaj5be23tb6wxgka8k6)

昨日（2020/10/07) [Tailwind CSS](https://tailwindcss.com/) からリリースされた Tailwind Play がとても良かったので紹介します。
Tailwind CSS を使うなら触っておいて損はないと思います。

# Tailwind Playとは？

https://play.tailwindcss.com/

[Tailwind CSS](https://tailwindcss.com/) のインタラクティブなコーディングが行える Web サイトです。
見た目・機能としては、Code Pen や TypeScript Playground が近いですかね。
左ペインのエディタで書いた内容が右ペインの表示にインタラクティブに反映されます。そして、追加の CSS や Tail Wind の Config の設定も自由行えます。

詳細は、サイトにアクセスするか以下紹介動画をみてもらえればわかるのですが、次項以降で特に良いなと思ったポイントを紹介します。

@[youtube](eCWhTZ34Hck)

# tailwind CSS のコード補完が行える
専用エディタだけあって当たり前ですが、Tailwind CSS のコード補完が行えます。class にカーソルを移動して何か入力すると補完候補のポップアップが表示されます。
正直 Tailwind CSS の無数にある Class を覚えてないのでこれがあるだけでとても書きやすいです。
VSCode でも同様のプラグインがあるのですが、Web エディタでインタラクティブに使えるのが良いですね。

![](https://storage.googleapis.com/zenn-user-upload/mj9e0p9y11vzwflo2iw9dfjebduv)

# config, CSSの設定が自由に行える
tailwind.config.js と CSS ファイルの設定を変えて、自分のプロダクトに合わせた環境でのテストが行えます。
config の設定をかえるとブラウザ上でビルドが走るようです。CodePen では出来なかったことなので嬉しいですね。

![](https://storage.googleapis.com/zenn-user-upload/dt2o4tt2duoja4wp03mnhkb6g8p0)

# plugin を自由に追加できる
ここが一番驚いたのですが、config の plugin の欄に追加したい Plugin を書くと、ファイルのロードが始まりエディタ上でその Plugin を使うことが出来るようになります！
本来は、ローカルで npm の設定をして始めなければ出来ないことが、即試せるので最高ですね。

![](https://storage.googleapis.com/zenn-user-upload/k8lodpns8jqm860svbgzrktfbwoc)

# レスポンシブにレイアウトを確認できる
Chrome DevTools を使わずに、ボタンクリックでレスポンシブなレイアウトを確認出来ます。
今のサイトだと、レスポンシブデザインが当たり前なのでこの機能も地味に便利ですね。

![](https://storage.googleapis.com/zenn-user-upload/mt4q6ehpj9qai75g0az8qynqnuvk)

# ワンクリックでデザインを share できる
これはかなり可能性のある機能だと思うのですが、TypeScript Playground ライクにワンクリックで編集内容をシェア出来ます。

![](https://storage.googleapis.com/zenn-user-upload/zyn20wd7tp7i14nq7afwbmlf5y2q)

:::message
おそらく今後、自分の作った Tailwind CSS のコンポーネントデザインをシェアする人が多く出てくると思います。
また、この機能を使った Tailwind CSS のデザインコンポーネントまとめサイトも出てきそうです。そうなったら、Tailwind CSS の導入のハードルが下がりますね。
実際にもう Twitter 上にいくつか上がっていました。今後が楽しみです。

https://twitter.com/victoryoalli/status/1313837379820703744
:::


# 終わりに

以上、「Tailwind Play でインタラクティブにデザインを構築する」でした！
これを使えば Tailwind を試すハードルがかなり下がりそうです。
Tailwind のエコシステムがどんどん進化していて凄い。今後にも期待です。