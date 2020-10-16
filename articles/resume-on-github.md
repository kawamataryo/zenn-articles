---
title: "GitHubの機能をフルに使って職務経歴書の継続的インテグレーションを実現する"
emoji: "📃"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "職務経歴書", "転職", "キャリア"]
published: false
---

# 何を作った

GitHub 職務経歴書を継続的に改善していく環境を作りました。リポジトリはこちらです。

https://github.com/kawamataryo/resume

# なぜ職務履歴書を？

特に転職の予定はないです。ただ最近「安定しているときこそ職務経歴書をまとめておくべき。いざと言うときはメンタルが消耗して書く余裕はない」という話を聞き、それは確かにと思ったからです。
そして、どうせ書くなら仕組みで何か面白いことをと思い GitHub に公開 & CI 環境の構築をしてみました。

# 機能紹介

## 📱GitHub Pages での Web ページ公開

https://kawamataryo.github.io/resume/

ただの GitHub のファイルビューでも良いのですが、より Web に最適化してみやすいほうがよいですよね。
GitHub の無料ホスティグ GitHub Pages を使って Web ページとして公開が可能です。

GitHub ページのデザイン は`docs/_config.yml`で設定しています。今は theme の設定だけですが、より細かい調整が可能です。

![GitHub Pages](https://storage.googleapis.com/zenn-user-upload/0sarbt933462xyrt0h6iokjk0ecd)

## 💅 textlint での文章校正

職務経歴書で`javascript`や、`JAVA`とか書かれていたら「大丈夫か。.?」と思いますよね。
大事な書類の typo を防ぐために、textlint による文書校正を行っています。husky による pre-commit フックでの実行 & GitHub Actions 実行を整備しています。

自動修正にも対応しているので Warning や Error が出た場合は以下を実行してみてください。

```bash
$ yarn lint --fix
```
![lint demo](https://storage.googleapis.com/zenn-user-upload/y3g6sw31tsg0qzrz5555drvd9ijo)


## 📝 md-to-pdf での PDF 生成

Web で職務経歴書をみてもらえればそれで簡潔なんですが、企業側としてはデータ or 紙で欲しいという場合もあると思います。
その対応のために md-to-pdf での Markdown から PDF への変換も可能です。

```bash
$ yarn build:pdf
```

md-to-pdf は内部で Puppeteer を使用しているので、PDF のデザインは、CSS で細かい調整が可能です（例： ゴシック体から明朝体への変換など）。
`pdf-configs/style.css`、`pdf-configs/config.js`を編集してください。

実際に出力される PDF はこのようなものです。
結構綺麗に出力されているのではないでしょうか。

![pdf demo](https://storage.googleapis.com/zenn-user-upload/91bnxughl3crx11s0is0bqleev85)

## 🛠GitHub Actions でのリリースビルド

GitHub ビルドインの CI 機能 GitHub Actions を使って、Release の自動作成と PDF の自動ビルド・登録を行っています。
tag 付きで push したときに Workflow が流れて以下のように Release の作成、自動ビルドされた PDF ファイルの assets への登録されます。

PDF を共有するときはこちらの画面を説明すれば良いのでとても手軽なのではと思っています。
Release 情報で更新日・バージョンも明確です。

https://github.com/kawamataryo/resume/releases

![リリースビルド](https://storage.googleapis.com/zenn-user-upload/soleu14nmiawocs6hzphpd5o28h6)


## 📅 GitHub Actions での更新リマインダー

職務経歴書を書いたは良いものの更新を忘れがちになりますよね。
GitHub Actions のスケジュール実行機能を利用して、3 か月ごとに更新の Issue を自動作成することで職務経歴書の更新を促します。
**Issue はクローズしたくなるのがエンジニアの性**なので、職務経歴書アップデートへの良い圧力になるのではと期待しています！ !

https://github.com/kawamataryo/resume/issues/3

![更新リマインダー](https://storage.googleapis.com/zenn-user-upload/d2rrbsbd17lulicht97e5iwcnfcs)

# おまけ

ここまで紹介した機能をすぐに使える GitHub テンプレートを公開しています！
もし良かったら使ってみてください！（ついでにスターももらえると泣いて喜びます）

# 終わりに

以上、「GitHub の機能をフルに使って職務経歴書の継続的インテグレーションを実現する」
今回 GitHub アクションを初めて自分で書いたのですが、本当に便利ですね。この機能を使って職務経歴書をアップデートしていきたいです。
（ちなみに環境を作るのが面白すぎて肝心の職務履歴書の進捗はダメです。.😇）

あと是非エンジニア採用の方に聞いてみたいのですが、こういう職務経歴書の形式はアリでしょうか？　もしお考え等あればコメント頂けると嬉しいです！
