---
title: "ブラウザ履歴・ブックマーク・タブを高速にインクリメンタルサーチできるChrome拡張を作ってみた"
emoji: "🔎"
type: "tech"
topics: ["vue","typescript","extension","chrome"]
published: false
---

個人的に超便利な Chrome 拡張を作ってみたので紹介です。ぜひお試しください🙏

# 作ったもの

Chikamichi（近道）という Chrome 拡張機能を作りました。

https://chrome.google.com/webstore/detail/chikamichi/gkhobepjbiepngbeikhbpnfgjcjgmgha?hl=ja&authuser=1

https://www.youtube.com/watch?v=Oi8MlZeaa4Y

特徴はこちらです。

* 履歴・ブックマーク・現在開いているタブを横断的にインクリメンタルサーチ
* クリック・エンターでの対象への移動
* キーボードショートカットでのスムーズな操作

コードも公開しています。

https://github.com/kawamataryo/chikamichi

:::message
デザイン・機能は[Sidekick](https://www.meetsidekick.com/)のサーチダイアログを参考にしています。
:::

:::message
Chikamichi という名前は[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)というコミュニティで[@にっしー](https://twitter.com/paranishian)さんに案を頂きました。感謝！
:::

# 使い方

使い方はこちらです。マウスを使わずキーボードだけで操作できます。
Sidekick のサーチダイアログの不満が `Ctrl+n` や `Ctrl+p` で上下操作できなかったことなので、個人的にそこを解消しただけでも使いやすいなと思ってます。

| コマンド                       | 動作                                  |
|-------------------------------|------------------------------------------|
| Alt + k                       | サーチダイアログの起動                       |
| ↓ or ↑ (Ctrl + n or Ctrl + p) | 検索結果の対象の選択                    |
| Enter                         | 対象のサイトを現在のタブで開く（タブの場合はタブへの移動）            |
| Ctrl + Enter                  | 対象のサイトを新しいタブで開く（タブの場合はタブへの移動） |

:::message alert
chrome の new tab ページでは、Chrome Extension の content_script の読み込みが出来ず、`Alt + k` でも起動しないので注意です。
対策は検討中です。
:::

# 工夫したところ

## Fuese.jsでのあいまい検索
ただの完全一致や部分一致だと良い検索結果が得られなかったので、Fuse.js という Fuzzy Search を可能にするライブラリを利用して、サイトタイトルと URL を対象にあいまい検索を実装しています。そのため、ちょっとした単語でもより良い検索結果が表示されます。

https://github.com/krisk/fuse

## viteese-webextでの爆速開発
Chrome Extensions の開発は、環境構築が面倒なのですが、今回は viteese-webext というボイラープレートを使いました。Vite、TypeScript、WindiCSS などの環境が整えられた状態で開発をスタートできます。そのおかげで、即主要な機能の開発に着手できました。開発効率にかなり影響したと思います。

https://github.com/antfu/vitesse-webext

# 終わりに

正月の勢いで実装した割には便利な拡張機能が作れたかなと思っています。
多くの人に使ってもらえると嬉しいです。また、何か不具合あればお気軽に Twitter や GitHub Issue でご連絡ください。
コントリビューションも大歓迎です！

https://twitter.com/KawamataRyo
https://github.com/kawamataryo/chikamichi/issues