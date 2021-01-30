---
title: "商品入荷情報を定期的にスクレイピングしてSlack通知する（Python + Lambda + CloudWatch)"
emoji: "🎽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "serverless"]
published: false
---

欲しいディスプレイが品切れで、なんとか再入荷時に購入したいので入荷情報をスクレイピングして Slack に流すスクリプトを書いて Lambda + CloudWatch で定期実行してみました。

# 環境構築

```bash
$ serverless create \
  --template aws-python3 \
  --name zenn-sample \
  --path zenn-sample
```

# Serverless Frameworkの設定


# 実例


## 外部モジュールが含まれない

import を解決できずつまりました。

```error
{
    "errorMessage": "Unable to import module 'handler': No module named 'requests'",
    "errorType": "Runtime.ImportModuleError"
}
```
[安定のクラスメソッドさんの記事](https://dev.classmethod.jp/articles/serverless-framework-lambda-numpy-scipy/)によると、plugin を使って Docker image で依存も一緒に zip ファイルに圧縮してデプロイする必要があるそうです。

# deploy


# 参考

- [Serverless Framework - Google Cloud Functions Guide - Credentials](https://www.serverless.com/framework/docs/providers/google/guide/credentials/)
- [AWS LambdaでNumpy、Scipyを使ってみる！Serverless Frameworkとプラグインを使えばパッケージ管理が簡単！ | Developers.IO](https://dev.classmethod.jp/articles/serverless-framework-lambda-numpy-scipy/)
- [[小ネタ]Serverless FrameworkでLambdaを定期実行させる | Developers.IO](https://dev.classmethod.jp/articles/serverless-framework-lambda-cron-execute/)