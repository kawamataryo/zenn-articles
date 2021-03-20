---
title: "Puppeteer + Lighthouse で認証ありWebアプリのフロントエンドパフォーマンスを定期測定する"
emoji: "⏱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GithubActions", "puppeteer", "lighthouse", "typescript", "datadog"]
published: false
---

Puppeteer + Lighthouse + GitHub Actions を使って認証あり Web アプリのフロントエンドパフォーマンスを定期測定する基盤を作ってみたので紹介します。

# 何を作る

下記の図のように、Puppeteer で headless Chrome を動かしてその上で Lighthouse を実効して Web アプリのフロントエンドパフォーマンスを計測。その処理を GitHub Actions 上で動かして定期実効。さらにデータを Datadog に送り可視化するまでを書きます。

ここで紹介するコードは全て以下リポジトリに上がっています。

# パフォーマンスの計測

Lighthouse + puppeteer を使う

# 計測データの可視化

可視化部分は外部の DB に接続して計測データを永続化し、Redash などの BI ツールで可視化するなど色々やり方はあると思うのですが、今回は DataDog にメトリクスとして送信することでデータの可視化を行う例を書きます。

まず DataDog のクライアント部分の実装です。

```ts
import fetch from "node-fetch";
import { AuditsResult } from "./@types/lighthouse";

export class DDClient {
  private apiUrl: string;

  constructor(apiKey: string) {
    this.apiUrl = `https://api.datadoghq.com/api/v1/series?api_key=${apiKey}`;
  }

  async sendMetrics(url: string, data: AuditsResult) {
    const pageName = url.replace("https://example.com/", "");
    const dataName = data.id.replace("-", "_");

    const requestBody = JSON.stringify({
      series: [
        {
          metric: `application.example.${pageName}.${dataName}`,
          points: [
            [
              `${Math.floor(Date.now() / 1000)}`,
              `${Math.round(data.numericValue / 10) * 10}`,
            ],
          ],
          type: "gauge",
        },
      ],
    });
    return await this.post(requestBody);
  }

  private async post(requestBody: string) {
    return await fetch(this.apiUrl, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: requestBody,
    });
  }
}
```

`sendMetrics`は Datadog のエンドポイントに向けて取得したデータを送信する処理です。url から page 名を取り出し、その page 名と Lighthouse で計測したデータ名を組み合わせてメトリクス名としています。

あとは Lighthouse での計測後、結果の送信処理を追記すれば OK です

```ts:src/index.ts
// ...
  for (const url of URLS) {
    const { lhr } = await lighthouse(
      url,
      { ...CHROME_OPTIONS, port: chrome.port },
      LIGHTHOUSE_OPTIONS
    );
    const json = reportGenerator.generateReport(lhr, "json");

    const audits = JSON.parse(json as string).audits as Audits;

    // Datadogへの送信
    await ddClient.sendMetrics(url, audits["first-contentful-paint"]);
    await ddClient.sendMetrics(url, audits["interactive"]);
    await ddClient.sendMetrics(url, audits["largest-contentful-paint"]);
    await ddClient.sendMetrics(url, audits["speed-index"]);
    await ddClient.sendMetrics(url, audits["total-blocking-time"]);
    await ddClient.sendMetrics(url, audits["max-potential-fid"]);
  }
// ...
```

これでスクリプトを実行すると Datadog にメトリクスが集計されるので、Datadog の Timeline ウィジェットなどで可視化が可能です。

# 定期実効

GitHub Actions を使う

# おわりに
