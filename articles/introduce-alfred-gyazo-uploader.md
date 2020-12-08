---
title: "最短で画像をGyazoにアップロードし記事に挿入するAlfred Workflowを作ってみた"
emoji: "🎑"
type: "tech"
topics: ["alfred", "gyazo", "zenn"]
published: true
---

先日、 Zenn の記事には Gyazo の画像リンクを挿入できると知りました。

@[tweet](https://twitter.com/KawamataRyo/status/1335502198088683521)

そこで「これは Zenn-CLI で記事を書く際に少し不便だった画像アップロードのプロセスを改善できるのでは？」と思い立ち、 Alfred Workflow を作ってみたので紹介です。

# 作ったもの
[alfred-gyazo-uploader](https://github.com/kawamataryo/alfred-gyazo-uploader)という Alfred Workflow を作りました。

https://github.com/kawamataryo/alfred-gyazo-uploader

機能はこちらです。

1. Finder で選択中のファイルを API 経由で Gyazo にアップロードする
2. アップロードされた画像の公開 URL を Markdown の画像挿入形式でクリップボードに保存する

今までの画像アップロードのプロセス（ブラウザを開く、画像を D&D、コピー）と比べてノーストレスで記事に画像を挿入できます。

![](https://i.gyazo.com/e4cf93b2f2b5ca0b430f62446d0efce0.gif)


# 使い方

[Alferd Powerpack](https://www.alfredapp.com/powerpack/) を購入済み、[Gyazo](https://gyazo.com)の無料アカウントを作成済みの前提で説明します。

## 1. Gyazoの API トークンの取得
画像のアップロードの際に、Gyazo の API トークンを使うので最初に取得します。

事前に Gyazo のアカウントを作成したうえで、[Gyazo API](https://gyazo.com/api?lang=en)にアクセスして、`Register applications` でアプリを登録します。

![](https://storage.googleapis.com/zenn-user-upload/cd8fr53oey8b2yl5u4r2kny378xr)

アプリ名と、callback URL は適当なもの入力します。

![](https://storage.googleapis.com/zenn-user-upload/xrnbvhbxw3f06owemvwuindna7tg)

作成されたアプリの詳細画面にて`Generate`ボタンを押してアクセストークンを生成します。生成されたアクセストークンを workflow の環境変数として設定するのでメモしておいてください。

![](https://storage.googleapis.com/zenn-user-upload/8tp167tsyy3pfvdors6cy0na09t0)


## 2. Workflowのインストール
alfred-gyazo-uploader の[releases](https://github.com/kawamataryo/alfred-gyazo-uploader/releases)ページから最新の alfredworkflow ファイルをダウンロードします。

![](https://storage.googleapis.com/zenn-user-upload/1o2rovhagt9a35b0xrrxnu9dctd3)

ダウンロードされたファイルをダブルクリックすると Alfred Workflow のインポート画面が立ち上がるので環境変数の`GYAZO_API_TOKEN`の値に先ほどメモしたアクセストークンを入力します。
その後`import`を押せばインストールの完了です。

![](https://storage.googleapis.com/zenn-user-upload/3wwvrj72pm0ce86oe8fdmbnc15dd)

## 3. 画像のアップロード

記事に貼りつけたい画像を選択したうえで、Alfred のランチャーにて`gyazou`と入力してエンターキーを押せばアップロード完了です。
アップロード画像の Markdown の画像書式で公開 URL がクリップボードに保存されるので、あとは記事に貼り付けるだけです。便利！


![](https://i.gyazo.com/e4cf93b2f2b5ca0b430f62446d0efce0.gif)


# Appendix

もし、アニ GIF など画像のサイズが大きすぎて記事中での読み込みが遅そうだなと思ったら [alfred-imaemin](https://github.com/kawamataryo/alfred-imagemin) というアプリを試してみてください。
alfred-gyazou-uploader と同じような使用感で、画像の圧縮を手軽に行えます。

https://github.com/kawamataryo/alfred-imagemin

使い方などはこちらの記事で説明しています。

https://qiita.com/ryo2132/items/358ce9d4baa8a8a27092

# おわりに

以上「最短で画像を Gyazo にアップロードし記事に挿入する Alfred Workflow を作ってみた」でした。
Zenn-CLI を使うと記事執筆環境を自分で最適化できるので良いですね。
今後も alfred-gyazo-uploader を使って記事をたくさん書いていきたです。

また使用中に不具合などあれば、気軽にこの記事のコメントや[Twitterのメンション](https://twitter.com/KawamataRyo)にてお知らせください。

# 参考

- [sindresorhus/alfy: Create Alfred workflows with ease](https://github.com/sindresorhus/alfy)
- [shokai/node-gyazo-api: Gyazo API wrapper for Node.js](https://github.com/shokai/node-gyazo-api)
