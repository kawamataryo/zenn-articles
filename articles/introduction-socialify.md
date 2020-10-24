---
title: "GitHub SocialifyでOGP画像を生成してSNS映えするOSSに"
emoji: "📷"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Twitter", "GitHub"]
published: true
---

GitHub の OGP 画像を生成してくれる [GitHub Socialify](https://socialify.git.ci/) が凄く良かったので紹介です。

# SNS映えするOSS??
主たる OSS はほぼ設定されているのですが、GitHub の各リポジトリは OGP 画像を設定することが出来ます。
OGP 画像を設定しないと、リポジトリ作者のプロフィール画像がトリミングされて表示されるので、場合によってはとても残念な感じになります。

以下 OGP 画像を設定していないリポジトリのツイート。
（大胸筋しか目立ちません）

@[tweet](https://twitter.com/KawamataRyo/status/1317790025967235072)

これが OGP 画像を設定したリポジトリのツイートだと。。

@[tweet](https://twitter.com/KawamataRyo/status/1319803175067480064)

素敵😆!!
でも、いい感じの OGP 画像を作るのは、サイズ調整やら何やらなかなか大変ですよね。。
そんな手間を解消して、サクッと良い感じの OGP を生成してくれるサービスがあります。

# GitHub Socialifyでの画像生成

それが GitHub 専用の OGP 生成サービス [GitHub Socialify](https://socialify.git.ci/) です！

https://socialify.git.ci/

にアクセスして、自分のリポジトリ URL を入力すると、使用言語、スター数などから自動的にいい感じの OGP 画像を生成してくれます。

![](https://storage.googleapis.com/zenn-user-upload/xlchr0pj6wa8ywsnh54hwkt7bjad)

表示するバッチや、独自のロゴ、背景など色々なカスタマイズも可能です。

![](https://storage.googleapis.com/zenn-user-upload/s49sixy374qck9hdsfcci5djxooh)

画像が決まったらダウンロードして、対象リポジトリの Settings > Social Preview にアップロードすれば完了です。

![](https://storage.googleapis.com/zenn-user-upload/6qvuzl1cx068vsfrzmaj1kd22sva)

最高便利🤩！

# 終わりに

簡単ですが、「GitHub Socialify で OGP 画像を生成して SNS 映えする OSS に」でした。
とても簡単に OGP 画像を生成できて便利ですね。まだ自分の OSS に OGP を設定していないほうは是非設定してみてください。OGP 画像があるだけで、なんか良い OSS ぽいと思ってもらえるかもです。

Socialify は Product Hunt にも出しているようなので感謝を込めて UPVOTED しました！

https://www.producthunt.com/posts/socialify?utm_source=badge-featured&utm_medium=badge&utm_souce=badge-socialify