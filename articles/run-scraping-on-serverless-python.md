---
title: "Lambdaで商品入荷情報を定期的にスクレイピングしてSlackに通知する（serverless framework）"
emoji: "🛒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "serverlessframework", "lambda", "aws"]
published: true
---

欲しいディスプレイが品切れだったので「なんとか再入荷時に購入したい！!」と思い立ち、Python の勉強がてら商品ページをスクレイピングして Slack に流すスクリプトを書きました。そして [serverless framework](https://www.serverless.com/) を使って、AWS Lambda の定期実行環境を組んだので、その過程をまとめます。

# 環境構築
AWS のコンソールでポチポチしたくないので [serverless framework](https://www.serverless.com/)を使って、Lambda を構築するようにします。

まず serverless framework をインストール。

```bash
npm install -g serverless
```

任意の作業ディレクトリで以下を実行。

```bash
sls create \
  --template aws-python3 \
  --name my-scraping-app \
  --path my-scraping-app
```

my-scraping-app ディレクトリ内に serverless framework 関連のファイルが生成されます。
その後 venv の設定や、serverless framework で AWS にデプロイするための credentials の設定をします（本記事では省略）。
以下 credentials 設定の参考ページです。

https://www.serverless.com/framework/docs/providers/aws/guide/credentials/

# スクレイピング & slack通知スクリプトの実装

スクレピングは様々な方法があると思うのですが、今回は該当商品の商品ページに出ている「現在品切れ中」というボタンの有無を確認することで、入荷状況を判断することとします。

![](https://i.gyazo.com/77d27266845e0d53720e3e4b367ff925.png)

依存モジュールを追加して、handler.py にスクレピングコードと Slack 通知コードを書いていきます。

```bash
pip install requests beautifulsoup4 slack_sdk
```

```python:handler.py
import requests
import re
import os
from bs4 import BeautifulSoup
from slack_sdk.webhook import WebhookClient


def scraping(event, context):
    TARGET_URL = "https://www.dell.com/ja-jp/shop/dell-%E3%83%87%E3%82%B8%E3%82%BF%E3%83%AB%E3%83%8F%E3%82%A4%E3%82" \
                 "%A8%E3%83%B3%E3%83%89%E3%82%B7%E3%83%AA%E3%83%BC%E3%82%BA-u4021qw-40%E3%82%A4%E3%83%B3%E3%83%81%E3" \
                 "%83%AF%E3%82%A4%E3%83%89%E6%9B%B2%E9%9D%A2usb-c-hub-%E3%83%A2%E3%83%8B%E3%82%BF/apd/210-aypy/%E3%83" \
                 "%A2%E3%83%8B%E3%82%BF%E3%83%BC-%E3%83%A2%E3%83%8B%E3%82%BF%E3%83%BC%E3%82%A2%E3%82%AF%E3%82%BB%E3" \
                 "%82%B5%E3%83%AA%E3%83%BC"
    html = requests.get(TARGET_URL)
    soup = BeautifulSoup(html.content, "html.parser")

    search = re.compile('.*現在品切れ中.*')
    find_result = soup.find("div", string=search)

    if find_result is None:
        _send_slack(TARGET_URL)
    else:
        print('まだ品切れ中😭')

    return {"statusCode": 200}


def _send_slack(target_url):
    url = os.environ['SLACK_WEBHOOK_URL']
    webhook = WebhookClient(url)
    res = webhook.send(
        text="探しものが見つかったよ🔎",
        blocks=[{
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "探しものが見つかったよ🔎"
            },
            "accessory": {
                "type": "button",
                "text": {
                    "type": "plain_text",
                    "text": "サイトを開く",
                    "emoji": True
                },
                "url": target_url
            }
        }],
    )
    print(res)
```

ポイントだけ簡単に。
`現在品切れ中`ボタンの判定は以下コードです。
BeautifulSoup にて正規表現で対象文字列を検索しています。そして、文字列がない場合のみ Slack 通知のメソッドを実行しています。

:::message
2021/01/31 20:40 追記
これは、BeautifulSoup を使わず、request 単体でレスポンスに対して文字列マッチしたほうが低コストだとアドバイス頂きました。
確かにそうのとおりなので後で書き直します。
:::

```python
    search = re.compile('.*現在品切れ中.*')
    find_result = soup.find("div", string=search)

    if find_result is None:
        _send_slack(TARGET_URL)
    else:
        print('まだ品切れ中😭')
```

Slack 通知部分は以下のコードです。
環境変数から Incoming Webhook の URL を取得して、その URL に対して SDK 経由でメッセージを送信しています。

```python
    url = os.environ['SLACK_WEBHOOK_URL']
    webhook = WebhookClient(url)
    res = webhook.send(
        text="探しものが見つかったよ🔎",
        blocks=[{
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": "探しものが見つかったよ🔎"
            },
            "accessory": {
                "type": "button",
                "text": {
                    "type": "plain_text",
                    "text": "サイトを開く",
                    "emoji": True
                },
                "url": target_url
            }
        }],
    )
```

# serverless frameworkでデプロイ

serverless framework の設定をして、スクリプトを Lambda にデプロイします。

## 外部モジュールのデプロイ設定

まず、lambda で Python の外部モジュールを使うために serverless framework のプラグイン`serverless-python-requirements`を追加します。
以下コマンド実行すれば OK です。これで`package.json`の追加と、`serverless.yml`のプラグイン設定の追記が完了します。

```bash
sls plugin install -n serverless-python-requirements
```

その後、依存モジュールの情報を `requirements.txt` にまとめます。

```bash
pip freeze > requirements.txt
```
※ こちらの作業の詳細は以下記事にまとめました。

https://zenn.dev/ryo_kawamata/articles/python-exclude-package-on-serverless-framework

## 環境変数の設定

続いて Incoming Webhook の URL で参照している環境変数の設定をします。
Git 管理しない`.env`ファイルで値を保持したいので、`serverless-dotenv-plugin` を使います。

以下コマンドでインストールします。

```
sls plugin install -n serverless-dotenv-plugin
```

`.env`ファイルを作成し、Slack の Incoming Webhook URL を設定します。

```.env
SLACK_WEBHOOK_URL=xxxxxxxxxxxxxxxxxxxxxxxx
```

URL の取得方法・設定方法はこちらをご確認ください。

https://slack.com/intl/ja-jp/help/articles/115005265063-Slack-%E3%81%A7%E3%81%AE-Incoming-Webhook-%E3%81%AE%E5%88%A9%E7%94%A8

## handlerの指定、Lambdaの起動イベントの設定

商品入荷情報をいちはやく知るためには定期的な実行が必要です。
今回は CloudWatch Events を使って、Lambda の定期実行を実現します。

serverless framework では events に schedule を指定するだけでその環境が構築されます。
今回は以下のように指定しました。

7 時から 21 時の間、1 時間に 1 回スクリプトが定期実行されます。
※ reasion を`ap-northeast-1`にするのも忘れずに。

```yml:serverless.yml
provider:
  name: aws
  runtime: python3.8
  lambdaHashingVersion: 20201221
  region: ap-northeast-1

functions:
  scraping:
    handler: handler.scraping
    events:
      - schedule: cron(0 7-21 * * ? *)
```

## デプロイ

最終的な serverless.yml はこちらです。

```yml:serverless.yml
service: my-scraping-app
frameworkVersion: '2'

provider:
  name: aws
  runtime: python3.8
  lambdaHashingVersion: 20201221
  region: ap-northeast-1

functions:
  scraping:
    handler: handler.scraping
    events:
      - schedule: cron(0 7-21 * * ? *)

plugins:
  - serverless-python-requirements
  - serverless-dotenv-plugin
```

最後にデプロイコマンドを実行して完了です。

```bash
sls deploy
```

AWS にリソースが作成されます。

![](https://i.gyazo.com/ef2696080e71e9291b1afe58ab4a2355.png)


以下テスト実行の結果です。もし商品が入荷されれば Slack に通知がきます。

![](https://i.gyazo.com/eb7b0ed3246a04acc5ae1190ee6a88de.png)


# 終わりに

これで、いちはやくディスプレイをゲットできる（はず。..）💪

# 参考

- [Serverless Framework - Google Cloud Functions Guide - Credentials](https://www.serverless.com/framework/docs/providers/google/guide/credentials/)
- [AWS LambdaでNumpy、Scipyを使ってみる！Serverless Frameworkとプラグインを使えばパッケージ管理が簡単！ | Developers.IO](https://dev.classmethod.jp/articles/serverless-framework-lambda-numpy-scipy/)
- [[小ネタ]Serverless FrameworkでLambdaを定期実行させる | Developers.IO](https://dev.classmethod.jp/articles/serverless-framework-lambda-cron-execute/)