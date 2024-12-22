---
title: "Sky Follower Bridge の今までとこれから"
emoji: "🦋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bluesky", "twitter","個人開発"]
published: false
---

この記事は [Bluesky Advent Calendar 2024](https://adventar.org/calendars/10752) の8日目の記事です。

個人で開発しているBlueskyのフォロー移行ツール[Sky Follower Bridge](https://skyfollowerbridge.com/)の開発の振り返りと、これからの展望を書きたいと思います。

# 🦋 Sky Follower Bridgeとは？
Sky Follower Bridgeは、XやThreadsなどのSNSのフォロー情報をもとにBlueskyのユーザーを検索して、フォローをすることができるブラウザ拡張機能です。リストのインポートやブロックリストのインポートにも対応しています。

https://www.sky-follower-bridge.dev/

https://www.youtube.com/watch?v=CnjjfSxm0G0

Blueskyを始めた当初、知り合いを探すためにXでフォローしている人を、Blueskyの検索画面で探してフォローしていたのですが、その際「これ自動化できたら便利では？」と思い、開発を始めました。

開発した当初は、自分が使えれば良いかなと思っていたのですが、Blueskyの躍進とともにユーザー数が増え、現在では全世界で300,000人以上のユーザーが使ってくれています。GitHubのスター数も700を超え、自分が開発しているOSSでは最大のスター数を達成しました。素直に嬉しいです。

https://github.com/kawamataryo/sky-follower-bridge

[![Star History Chart](https://api.star-history.com/svg?repos=kawamataryo/sky-follower-bridge&type=Date)](https://star-history.com/#kawamataryo/sky-follower-bridge&Date)


# 📅 開発の振り返り

### 2023/05 初回リリース

開発を思い立って1ヶ月ほどで初期バージョンをリリースしました。当時はBlueskyのユーザー数も少なく、ほぼ注目されなかった気がします。

また当時は今とは全く異なるUIで、Xのフォロー一覧に直接Blueskyのユーザー情報を埋め込む形式でした。実装的にも、素のJSで直接XのDOMを操作してUIを埋め込むなど、力業のコードベースでした。

https://bsky.app/profile/kawamataryo.bsky.social/post/3jwd4i6jnvm27

https://www.youtube.com/watch?v=XcRcWjStIMc


## 2023/08 Van.jsでリファクタリング

Followボタンの挙動などカオスなコードでメンテが大変だったので、当時Hacker Newsで話題になっていたVan.jsで素のJSでのDOM操作を置き換えました。この時もまだUIはそのままです。

https://zenn.dev/ryo_kawamata/articles/1ad6e51eed13ae


## 2024/01 UIリニューアル & Plasmoでリアーキテクチャ

2023年の年末ごろから徐々にユーザーが増え、GitHubでのIssueも多くなりました。その中で一番多かったのが、UIの改善でした。今まではユーザーを検索するために、Xの仮想スクロールへの対応のため、自分でスクロールする必要があったのですが、それを自動化したいという要望です。

https://github.com/kawamataryo/sky-follower-bridge/issues/28

これに応えるためUIを刷新しました。今までとは異なり、XのUIへの埋め込み型ではなく別にモーダルを表示して内部で結果を表示するようにしました。

また、このタイミングでVan.jsでの実装には限界を感じていたため、Chrome拡張のフレームワークであるPlasmo + React を使って全面的にコードも書き換えました。これがその後の保守や新規開発にもつながったのでこの判断は良かったなと思っています。

https://docs.plasmo.com/

https://www.youtube.com/watch?v=pVqoDv-1uac

## 2024/08 大手ITメディアに紹介され海外ユーザーに拡散
2024/2に、Blueskyが招待制ではなくなり、一般に開放された頃からさらにユーザーが増えました。特にTechCrunchやWiredなどの海外大手メディアに紹介されたことも大きかったです。

https://www.wired.com/story/how-to-get-started-on-bluesky/

https://techcrunch.com/2024/11/14/how-to-use-bluesky-the-twitter-like-app-thats-taking-on-elon-musks-x/

https://lifehacker.com/tech/use-this-extension-to-find-all-your-x-followers-on-bluesky

## 2024/10 Bluesky Teamのsamuelから修正PRをもらう

Elonの発言や、ブラジルでのX停止などでBlueskyへの流入が爆発的に増え、比例して自分のツールもさらに使われるようになりました。

しかし、自分の書いたコードにバグがあり、Blueskyサーバーに負荷をかけてしまうという問題がありました。

これはBlueskyチームの [@samuel](https://bsky.app/profile/samuel.bsky.team) が見つけてくれて、修正してくれました。ツールの利用する停止させるのではなく、修正PRを送ってくれたところに、OSSの利点を感じると共にBlueskyコミュニティの強さを感じました。

@samuelが送ってくれた修正PR

https://github.com/kawamataryo/sky-follower-bridge/pull/51

## 📅 UIリニューアル

以前のUIはモーダルでユーザー情報表示している都合上、表示領域やCORSなど諸々制約がありました。そこで、optionページにユーザー情報を表示するようにUIを修正しました。

これが今のUIの原型です。このUIのおかげで、一括フォローや、リストのインポート、再検索などの機能を実装することができました。


## 📅 2024/11 偽サイト発見 -> 公式サイト作成

Sky Follower Bridgeの偽サイトが出回っていることにユーザーの通報から気がつきました。まさか自分のツールでそんなことがあるとは思わず、本当に驚きました。また自分のこれまで築いてきたツールの信頼が勝手に利用されている気がして、とても悔しかったです。

この件は偽サイトが同名の類似ツールを提供してきたり、同名の公式アカウントを取得したりと、その後も問題があったのですが、Blueskyチームや海外の有志の方の助けもあり、一度は偽サイトと偽公式アカウントの閉鎖までこぎ着けました。

ただし、その後も、偽サイトは別サーバーで再開して、今も継続的にユーザーを騙しているようです。この件は別に記事にしたいと思っています


## 📅 2024/12 Product Huntへ応募


## 📅 2024/12 Threads対応



# 🚀 今後の展望

## 🌐 対応SNSの拡充

今はXとThreadsのみ対応していますが、アーキテクチャ的にスクレイピング部分は分離されていて、そこにコードを追加すれば対応SNSを増やすことができます。直近は要望のあったTikTok対応を検討しています。

その後、余裕あれば以下のSNS対応も行いたいなと思っています。

- Mastodon
- Instagram
- Facebook
- LinkedIn

## 📱 モバイル対応（iOS/Androidアプリ or モバイル版ブラウザ拡張）

今はPCを持っているユーザーしか対応しておらず、要望としてもSP対応の声が多いので、React Nativeでアプリ化を検討しています。
今開発中React Nativeの画面


## 🎉 最後に

開発当初はこんなにも多くの人に使ってもらえるとは思っていませんでした。地道にIssueに応えつつ開発を続けてよかったなと思います。今後もより多くの人に使ってもらえるように頑張りたいです。
