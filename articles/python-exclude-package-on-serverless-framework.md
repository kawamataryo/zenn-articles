---
title: "serverless frameworkで外部モジュールを利用したPythonのlambdaをデプロイする時は.."
emoji: "🚸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "serverlessframework", "lambda", "aws"]
published: true
---

[serverless framework](https://www.serverless.com/)を使って、 Python で書いたスクリプトを AWS Lambda にデプロイしようとした際に詰まったのでメモ。

# 何が起こった？

serverless framework で deploy は成功するものの、lambda を実行すると外部モジュール の import エラーとなりました。

```
errorMessage": "Unable to import module 'handler': No module named 'BeautifulSoup'
errorType": "Runtime.ImportModuleError
```

もちろんローカルでは外部モジュールを pip インストールしているので、テストしても正常に動作します。


# 原因
Python で lambda に標準で同梱されているモジュール以外を利用する場合は、外部モジュールも明示的にデプロイする Zip ファイルに含める必要があるようです。
Serverless Framework で特に設定せずデプロイした場合、実行ファイルにて import している外部モジュールはデプロイ対象に含まれないのでエラーが発生していました。

https://docs.aws.amazon.com/lambda/latest/dg/python-package.html

# 対応

`serverless-python-requirements` を使って、依存する外部モジュールも一緒に Zip にまとめてデプロイするようにします。

https://github.com/UnitedIncome/serverless-python-requirements


以下コマンドで `serverless-python-requirements` のインストール（npm）と、`package.json`の作成、`serverless.yml`の plugins への追記が完了します。

```bash
sls plugin install -n serverless-python-requirements
```

そして、pip で import した外部パッケージを `requirements.txt` にまとめます。

```bash
pip freeze > requirements.txt
```

これで準備は完了です。あとは通常のデプロイコマンドでデプロイすればエラーが解消されるはずです 🎉

```bash
sls deploy
```

# 補足

## Pure Python以外の外部モジュールを使う場合
外部モジュールとして import しているものの内部で、C 言語を使っている場合など、Amazon Linux 環境でインストール（ビルド）したファイルを使ってパッケージングする必要があります。 Numpy、Scipy などです。
その場合は、通常の`serverless-python-requirements`の設定ではエラーが発生してしまうのですが、Docker と docker-lambda イメージの設定を行うことで解消されます。

`serverless.yml` に以下を追記してください。


```yaml
custom:
  pythonRequirements:
    dockerizePip: true
```

こちらは[安定のクラスメソッドさんの記事](https://dev.classmethod.jp/articles/serverless-framework-lambda-numpy-scipy/)を参考にさせて頂きました。

## 容量の大きい外部モジュールを使う場合
`numpy`、`scipy`などの容量の大きい外部モジュールを使う場合は、lambda のデプロイ容量制限に引っかかることがあります。
その場合の容量削減にも`serverless-python-requirements`は対応しています。

`serverless.yml` に以下を追記してください。

```yaml
custom:
  pythonRequirements:
    zip: true
    slim: true
```

そして、lambda の実行ファイルの先頭に以下を追記します。

```py
try:
  import unzip_requirements
except ImportError:
  pass
```

# 参考

- [How to Handle your Python packaging in Lambda with Serverless plugins](https://www.serverless.com/blog/serverless-python-packaging)
- [AWS LambdaでNumpy、Scipyを使ってみる！Serverless Frameworkとプラグインを使えばパッケージ管理が簡単！ | Developers.IO](https://dev.classmethod.jp/articles/serverless-framework-lambda-numpy-scipy/)
- [Python の AWS Lambda デプロイパッケージ - AWS Lambda](https://docs.aws.amazon.com/ja_jp/lambda/latest/dg/python-package.html)