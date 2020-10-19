---
title: "GitHubの機能をフルに使って職務経歴書の継続的インテグレーションを実現する"
emoji: "💡"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "キャリア", "GitHubActions", "CI"]
published: true
---

GitHub で職務経歴書を公開 & 継続的に改善していく環境を作ったのでその紹介です。
リポジトリはこちらです。

https://github.com/kawamataryo/resume

[![](https://storage.googleapis.com/zenn-user-upload/xiptxzi39xkarufwodq3r3zp1vv2)](https://github.com/kawamataryo/resume)


# なぜ職務経歴書を？

今のチームがとても好きなので転職の予定はないのですが、「安定しているときこそ職務経歴書をまとめておくべき。**本当に職務経歴書が必要なときはメンタルが消耗していて書く余裕はない**」という話を最近知り合いから聞き、それは確かにと思い書き初めました。どうせ書くなら何か面白いことをしたいなと GitHub に公開 & CI 環境の構築をやってみました。

# 機能紹介

## 🌐 GitHub Pages で Web ページとしての公開

Markdown + GitHub のファイルビューでも良いのですが、より見やすいほうが好ましいですよね。
GitHub の無料ホスティグ GitHub Pages を使って Web ページとして公開しています。

GitHub ページのデザイン は`docs/_config.yml`で設定しています。今は theme の設定だけですが、より細かい調整が可能です。

https://kawamataryo.github.io/resume/

[![GitHub Pages](https://storage.googleapis.com/zenn-user-upload/0sarbt933462xyrt0h6iokjk0ecd)](https://kawamataryo.github.io/resume/)

## ✨ textlint での文章校正

職務経歴書で`javascript`や、`JAVA`とか書かれていたら「大丈夫か。.?」と思いますよね。
大事な書類の typo を防ぐために、[textlint](https://github.com/textlint/textlint) による文書校正を行っています。[husky](https://github.com/typicode/husky) による pre-commit フックでの実行 & GitHub Actions での push 時の実行環境も整備しています。

自動修正にも対応しているので Warning や Error が出た場合は以下を実行してみてください。

```bash
$ yarn lint --fix
```
![lint demo](https://storage.googleapis.com/zenn-user-upload/y3g6sw31tsg0qzrz5555drvd9ijo)


## 📝 md-to-pdf での PDF 生成

Web で職務経歴書をみてもらえればそれで完結なんですが、企業側としてはデータ or 紙で欲しいという場合もあると思います。
その対応のために [md-to-pdf](https://github.com/simonhaenisch/md-to-pdf#readme) での Markdown から PDF への変換機能を追加しています。

```bash
$ yarn build:pdf
```

md-to-pdf は内部で Puppeteer を使用しています。PDF のデザインは、CSS で細かい調整が可能です（例： ゴシック体から明朝体への変換など）。
調整する場合は、`pdf-configs/style.css`、`pdf-configs/config.js`を編集してください。

実際に出力される PDF はこのようなものです。
結構綺麗に出力されているのではないでしょうか。

https://github.com/kawamataryo/resume/releases/download/v0.1.6/resume.pdf

![pdf demo](https://storage.googleapis.com/zenn-user-upload/91bnxughl3crx11s0is0bqleev85)

## 🛠GitHub Actions でのリリースビルド

GitHub 付属の CI 機能 GitHub Actions を使って、Release の自動作成と PDF の自動ビルドを行っています。
tag 付きで push したときに Workflow が流れて以下のように Release の作成、自動ビルドされた PDF ファイルの assets への登録されます。

PDF を共有するときはこちらの画面を説明すれば良いのでとても手軽なのではと思っています。
Release 情報で更新日・バージョンも明確です。

https://github.com/kawamataryo/resume/releases

[![リリースビルド](https://storage.googleapis.com/zenn-user-upload/ny8mxz7jtkl4wofp7k36euin05ku)](https://github.com/kawamataryo/resume/releases)



## 📅 GitHub Actions での更新リマインダー

職務経歴書を書いたは良いものの更新を忘れがちになりますよね。
GitHub Actions のスケジュール実行機能を利用して、3 か月ごとに更新の Issue を自動作成することで職務経歴書の更新を促します。
**Issue はクローズしたくなるのがエンジニアの性**なので、職務経歴書アップデートへの良い圧力になるのではと期待しています。

https://github.com/kawamataryo/resume/issues/3

[![更新リマインダー](https://storage.googleapis.com/zenn-user-upload/d2rrbsbd17lulicht97e5iwcnfcs)](https://github.com/kawamataryo/resume/issues/3)

# おまけ

ここまで紹介した機能をすぐに自分用に展開できる GitHub template も公開しています。もし職務経歴書の GitHub 公開に興味があればぜひ使ってみてください！

https://github.com/kawamataryo/resume-template

`Use this template` ボタンをクリックすると、自分の新しいリポジトリとして作成できます。

GitHub template の詳細はこちらの記事に以前書きました。
[GitHubのTemplate Repository機能のすゝめ - Qiita](https://qiita.com/ryo2132/items/08f0561804c798012146)

# 終わりに

以上、「GitHub の機能をフルに使って職務経歴書の継続的インテグレーションを実現する」でした。
この環境を使って職務経歴書を継続的にアップデートしていきたいです。
（何もしていないと何も書けないので、まず業務で実績を作っていかないと..💪）


# 参考

- [職務経歴書をGitHubで管理しよう - Qiita](https://qiita.com/okohs/items/abcad0b4aefa585bc50b)
- [エンジニアが読みたくなる職務経歴書 - dwango on GitHub](https://dwango.github.io/articles/engineers-resume/)
- [mikeda/resume: 職務経歴書](https://github.com/mikeda/resume)
- [kgsi/curriculum-vitae: kgsiの職務経歴書](https://github.com/kgsi/curriculum-vitae)