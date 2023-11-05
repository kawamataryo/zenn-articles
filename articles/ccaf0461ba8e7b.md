---
title: "マイクラサーバーの自動停止の仕組みをlambdaで作る"
emoji: "🗜️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "minecraft"]
published: true
---

最近EC2にマイクラサーバーを立てて家族で遊んでいるのですが、たびたびインスタンスの停止を忘れ、ムダな課金が発生してました。これはまずいと、自動停止の仕組みを作ってみたので紹介です。

# 🛠️ 作ったもの

以下の要件を満たす仕組みをlambdaで作りました。

* 1時間に1回、各サーバーのワールドのオンラインユーザー人数をチェック
* もしオンラインユーザーが0人の場合は対象のEC2インスタンスを停止する。

# 🧑‍💻 実装紹介

手軽にlambdaを開発できる[serverless framework](https://www.serverless.com/)を使って実装しました。

https://www.serverless.com/

これから紹介するコードは以下のリポジトリで公開しています。リポジトリをcloneして、READMEの通りにデプロイすればすぐに試せます。

https://github.com/kawamataryo/minecraft-auto-stop

:::message
今回久しぶりにserverless frameworkを使ったのですが、ServerlessのWEBサイトでダッシュボード機能が提供されることに驚きました。プロバイダーの設定や、ログをGUIで確認できるので便利ですね。
![](/images/ccaf0461ba8e7b/2023-11-05-09-09-59.png)

:::


## マインクラフトの状態確認とEC2の停止

以下の関数で、環境変数で設定している情報（MC_SERVERS）をもとに、各サーバーの状態を確認し、オンラインのユーザーが0人の場合はEC2を停止しています。

https://github.com/kawamataryo/minecraft-auto-stop/blob/main/src/handlers/checkAndStopServer.ts#L6-L53


MC_SERVERSは以下の型の配列です。

https://github.com/kawamataryo/minecraft-auto-stop/blob/main/types.d.ts

また、マイクラサーバーの状態確認には[mine-craft-protocol](https://github.com/PrismarineJS/node-minecraft-protocol)というnpmのping関数を使っています。他にも色々な操作ができて便利そうです。

https://github.com/PrismarineJS/node-minecraft-protocol

https://github.com/kawamataryo/minecraft-auto-stop/blob/main/src/clients/minecraft.ts#L11-L30

## 定期実行

1時間に1回の定期実行にはEventBridgeを使っています。serverless frameworkではserverless.ymlで以下のように設定するだけでOKです。

https://github.com/kawamataryo/minecraft-auto-stop/blob/main/serverless.yml#L18-L23



# おわりに

serverless Frameworkがとても便利で、サクッとできて良かったです。これで課金に怯えずマイクラを心置きなく楽しめそうです。
