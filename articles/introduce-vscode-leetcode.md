---
title: "vscode-leetcode で快適 LeetCode 生活"
emoji: "🎓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "leetcode", "競技プログラミング"]
published: true
---

先日[@0317_hiroya](https://twitter.com/0317_hiroya)さんの以下ツイートと [00:00STUDIO](https://0000.studio/) の配信を見てとても刺激を受けたので、[LeetCode]((https://leetcode.com/))をはじめてみようと思いました。

@[tweet](https://twitter.com/0317_hiroya/status/1358392576215191554)

ただ、LeetCode にアカウントを作りいざ問題にチャレンジしようと思ったら、いきなり詰まりました。

1. **関数・変数名の補完が効かない**
2. **シンタックスエラーを表示してくれない**
3. **Vim キーバインドがいつもと微妙に違う**

もう IDE がないとコードを書けない体になってしまっていた・・😇

解決策としてはローカルの好きなエディタでコードを書いて、完成したら都度ブラウザの LeetCode のエディタに貼り付けて実行となるのでしょうが、まどろっこしいことはしたくありません。
絶対誰か解決してるだろうとググっていたら、最高の VSCode プラグインを見つけたので紹介します。


# vscode-leetcodeの使い方

自分の悩みを解決してくれたのは[vscode-leetcode](https://github.com/LeetCode-OpenSource/vscode-leetcode)です。

https://github.com/LeetCode-OpenSource/vscode-leetcode

一切ブラウザを開かず VSCode 上で LeetCode の問題検索、テスト、解答までが行なえます。
VSCode 上で実際にファイルを開いてコーディングするのでコード補完も効きますし、シンタックスエラーも表示されます。もちろんいつもの Vim キーバインドでの操作もできます。最高🤩

![](https://i.gyazo.com/0e102fa67b331cc014b2a94cd4bf8afa.gif)

以下具体的な使い方をみていきます。

## インストール

https://marketplace.visualstudio.com/items?itemName=LeetCode.vscode-leetcode

プラグインのサイトにアクセスし install を押して VSCode にプラグインを追加します。
プラグインの実行には、Node.js の version 10 以上が必要なので Node.js をグローバルにインストールして path の解決をしておいてください。

## サインイン
インストール完了後、サイドバーから LeetCode のアイコンをクリックするとプラグインが起動します。最初に sign in を求められるので LeetCode のアカウントでサインインしましょう。

:::message alert
2021/02/11 現在、通常の Email・Password でのログインが失敗するようです（[参考](https://github.com/LeetCode-OpenSource/vscode-leetcode#%EF%B8%8F-attention-%EF%B8%8F--workaround-to-login-to-leetcode-endpoint)）。
`Third Party login` か `Cookie`を選択しましょう。
:::

![](https://i.gyazo.com/ae0a21047b9f155cfd84cbb3fcd052f1.png)

## 問題の選択

サインインすると、サイドバーに問題の一覧が表示されます。タグや難易度で分けられているので好きな問題を選択しましょう。

そして問題文右下の`Code Now`を押し`just open problem file`を選択するとローカルに実行ファイルが作られます。初回は実行ファイルの保存場所を聞かれるので適宜好きな場所を設定してください。

![](https://i.gyazo.com/6ec2362a91147088d044aab427df143a.png)

## コーディング
`just open problem file`を押すと左ペインに実行ファイル、右ペインに問題文というレイアウトになります。後はここでコーディングするだけです。

![](https://i.gyazo.com/48b94709e9a680fe7bfbb27ff7f0adb3.png)

どの言語で解答するかは、毎回ファイルを開くときに選択することも出来ますし、settings からデフォルトの言語を設定することもできます。

![](https://i.gyazo.com/b9d8f51c58310b86b9ddd9524b53f50a.png)

## テスト

コーディングが完了したらテストの実行です。
エディタ上の`Test`を押すことで実行できます。

![](https://i.gyazo.com/59d62fd6c5c8cb893a5c97f37bd14a1b.png)


どのテストケースでテストを実行するかの選択肢が表示されるのでどれか選びましょう。
ここで、`Write directly` を選ぶと自分でテストケースの入力値を設定することも出来ます。

![](https://i.gyazo.com/15d9a26b166ddf0e003a40f073d5d78a.png)


実行すると右ペインに結果が出ます。

![](https://i.gyazo.com/0866465820d03e78c22093566712311f.png)

## 解答

テストも通れば後は解答です。
解答はエディタ上の`submit`を押すとこで実行できます。

![](https://i.gyazo.com/45c23c8706cc5d0339461e8a3bb5e96c.png)


解答が完了すると右ペインに結果が表示されます。`Accepted`となっていたら無事全てのテストケースも通り解答完了です 🎉

![](https://i.gyazo.com/661e3a908797ebaf76893b0fc2983e16.png)

ブラウザで LeetCode にサインインして確認しても、解答が記録されています。

![](https://i.gyazo.com/09d8273733fb8b24c4d6dcc397dd0118.png)

# 終わりに

以上「vscode-leetcode で快適 LeetCode 生活」でした。
これで準備は整ったので、毎朝 30 分目安に挑戦して LeetCode の草をはやしていきたいです。

（2021/02/11 現在 圧倒的な空白）

![](https://i.gyazo.com/924e4bfb8911fd71a51f7dddfc6a2298.png)
