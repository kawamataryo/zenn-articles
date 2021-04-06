---
title: "Puppeteer +Lighthouse +GitHubActionsで認証付きWebアプリのWebperfを定期計測"
emoji: "🏎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GithubActions", "puppeteer", "lighthouse", "typescript", "datadog"]
published: true
---

Puppeteer + Lighthouse + GitHub Actions を使って Web アプリのフロントエンドパフォーマンスを定期計測するプロジェクトを作ってみたら良い感じだったので紹介です。

# 何を作った？

このように GitHub Actions 上で 認証付きの Web アプリに対して Puppeteer 介し Lighthouse を定期実行し、結果を Datadog に送信するプロジェクトを作りました。

![](https://i.gyazo.com/a2bf47367a9af79a8fec6b06246b1b2a.png)

実際にそのプロジェクトの計測値を使った Datadog のダッシュボードはこちらです。
Webperf の主要指標をページ別に時系列で表示しています。

![](https://i.gyazo.com/849787f0f1d3f4969bfcd25b64fd2a8f.png)

サンプルプロジェクトはこちらにあります。

https://github.com/kawamataryo/webperf-watcher-sample

以降で実装について簡単に解説します。

# Puppeteer + Lighthouse によるパフォーマンス計測

Puppeteer + Lighthouse によるパフォーマンス計測はこちらのコードで行っています。

[webperf-watcher-sample/blob/master/src/main.ts](https://github.com/kawamataryo/webperf-watcher-sample/blob/master/src/main.ts)

```ts:src/main.ts
//...
const login = async (browser: puppeteer.Browser) => {
  const page = await browser.newPage();
  await page.goto(`${BASE_URL}/login`);
  const navigationPromise = page.waitForNavigation();

  await page.type("input#id", LOGIN_ID);
  await page.type("input#password", LOGIN_PASS);
  await page.click(".login-button");

  await navigationPromise;
};

// ...

const main = async () => {
  // chrome-launcher でChromeを起動しPuppeteerに接続
  const chrome = await chromeLauncher.launch(CHROME_OPTIONS);
  const res = await fetch(`http://localhost:${chrome.port}/json/version`, {
    method: "GET",
  });
  const { webSocketDebuggerUrl } = await res.json();
  const browser = await puppeteer.connect({
    browserWSEndpoint: webSocketDebuggerUrl,
  });

  // ログインの実行
  await login(browser);

  const ddClient = new DDClient(DD_API_KEY);

  for (const url of TARGET_URLS) {
    // lighthouseの実効（直列実効でないと落ちるので注意）
    const { lhr } = await lighthouse(
      url,
      { ...CHROME_OPTIONS, port: chrome.port },
      LIGHTHOUSE_OPTIONS
    );
    const json = reportGenerator.generateReport(lhr, "json");

    const audits = JSON.parse(json as string).audits;

    // ...
  }

  await browser.disconnect();
  await chrome.kill();
};

```

Node.js から Chrome を起動する [Chrome Launcher](https://github.com/GoogleChrome/chrome-launcher) で Headless Chrome を開き、それを Puppeteer と接続し、その Chrome を利用して、Lighthouse を実行しています。
Puppeteer と Lighthouse の連携は以下リポジトリを参考にしました。

https://github.com/addyosmani/puppeteer-webperf

実装のポイントは、`main`関数にて **Lighthouse の実行前にログイン処理を行い、Chrome に認証情報を記録している点**です。
ここで事前に認証しておけば認証ありのページでもログイン画面にリダイレクトされることなく、Lighthouse での計測が可能になります。

# Datadog へのメトリクス送信

計測値の Datadog への送信は、Datadog の HTTP REST API を使っています。

:::message
この例では Datadog を使っていますが、送信先は Google スプレッドシートでも、BigQuery でもデータを集積・可視化できるものなら何でも良いと思います。
:::

こちらのコードが計測値の送信処理です。Lighthouse で計測したデータの中から`TARGET_METRICS`で定義したプロパティを抜き出し、それを後述する Datadog クライアントで送信しています。

[webperf-watcher-sample/blob/master/src/main.ts](https://github.com/kawamataryo/webperf-watcher-sample/blob/master/src/main.ts)

```ts:src/main.ts
// ..bundleRenderer.renderToStream
const main = () => {
  // ...
    const { lhr } = await lighthouse(
      url,
      { ...CHROME_OPTIONS, port: chrome.port },
      LIGHTHOUSE_OPTIONS
    );
    const json = reportGenerator.generateReport(lhr, "json");

    const audits = JSON.parse(json as string).audits;

    // Datadogへの送信
    await Promise.all(
      TARGET_METRICS.map(async (metrics) => {
        await ddClient.sendMetrics(
          metricsName(url, audits[metrics]),
          audits[metrics]
        );
      })
    );
  //...
}
// ...
```

サンプルプロジェクトのでは以下メトリクスを送信しています。

|metrics|description|
|---|---|
|FCP|First Contentful Paint. 何らかの DOM コンテンツがレンダリングされるまでの時間（[詳細](https://web.dev/first-contentful-paint/)）|
|LCP|Largest Contentful Paint. Web Vitals の指標のひとつ。 ブラウザの表示範囲内で、もっとも大きなコンテンツが表示されるまでの時間（[詳細](https://web.dev/largest-contentful-paint/)）|
|TTI|Time to Interactive. ページが完全にインタラクティブになるまでの時間（[詳細](https://web.dev/interactive/)） |
|Speed Index|ページの描画速度を示す指標（[詳細](https://web.dev/speed-index/)）|
|MPFID|Max Potential First Input Delay. Web Vitals の指標のひとつ。ユーザーが経験する可能性のある最悪の場合の最初の入力遅延（[詳細](https://web.dev/lighthouse-max-potential-fid/)|
|TBT|Total Blocking Time. FCP から TTI までの間のユーザーの入力応答を阻害する時間の合計（[詳細](https://web.dev/lighthouse-total-blocking-time/)）|


Datadog のクライアントはこちらです。メトリクス名と測定値を受け取り、内部で Datadog の REST API で提供されている Submit Metrics の API を叩いています。

https://docs.datadoghq.com/api/latest/metrics/#submit-metrics

[webperf-watcher-sample/blob/master/src/lib/ddClient.ts](https://github.com/kawamataryo/webperf-watcher-sample/blob/master/src/lib/ddClient.ts)

```ts:src/lib/ddClient.ts
export class DDClient {
  private apiUrl: string;

  constructor(apiKey: string) {
    this.apiUrl = `https://api.datadoghq.com/api/v1/series?api_key=${apiKey}`;
  }

  async sendMetrics(metricsName: string, data: AuditsResult) {
    const requestBody = JSON.stringify({
      series: [
        {
          metric: metricsName,
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

以上で Datadog にメトリクスが送信されるので、後はそのメトリクスを使いダッシュボードを作るだけです。
以下のダッシュボードでは [timeline ウィジェット](https://docs.datadoghq.com/ja/dashboards/widgets/event_timeline/)を使いとき系列データとして可視化しています。

![](https://i.gyazo.com/849787f0f1d3f4969bfcd25b64fd2a8f.png)



# GitHub Actions での定期実行

定期実行については GitHub Actions の cron トリガーにより実現しています。
GitHub Actions の Ubuntu イメージには Chrome が標準で入っているので、特に設定不要で実行出来ます。

以下の例だと 1 時間に 1 回、測定のスクリプトを実行ています。

[.github/workflows/mesure-webperf.yml](https://github.com/kawamataryo/webperf-watcher-sample/blob/master/.github/workflows/measure-webperf.yml)

```yml:.github/workflows/mesure-webperf.yml
name: Measure webperf

on:
  schedule:
    - cron:  '0 */1 * * *'

jobs:
  measure-webperf:
    runs-on: ubuntu-18.04
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v1

      - name: Install packages.
        run: yarn install

      - name: Measure webperf.
        run: yarn start
        env:
          LOGIN_ID: ${{ secrets.LOGIN_URL }}
          LOGIN_PASS: ${{ secrets.LOGIN_PASS }}
          DD_API_KEY: ${{ secrets.DD_API_KEY }}
```

:::message alert
**cron で実行する GitHUb Actions のジョブには必ず timeout-minutes を設定してください。**
GitHub Actions の標準のタイムアウトは 6h です。私はタイムアウトを設定し忘れ、1 日で GitHub Actions の無料枠を葬り去るという偉業を成し遂げました 😇

https://twitter.com/KawamataRyo/status/1373929865460674561
:::

# 詰まりどころ

最後に、実装時に詰まったところの共有です。

## ブラウザでの実行より十数秒遅い

最初、なぜかスクリプトで Lighthouse を実行するとブラウザから直接計測した場合より、どのスコアも十数秒遅いという現象に悩まされました。
Headless Chrome の特有の問題なのかと思ったのですが、結果からいうと Lighthouse のオプションの設定ミスが原因でした。

Node.js の Lighthouse は**デフォルトで低速なネットワーク回線を模したもので計測**を行うようになっていました。

[GoogleChrome/lighthouse/blob/v6.4.1/lighthouse-core/config/constants.js#L20-L29](https://github.com/GoogleChrome/lighthouse/blob/v6.4.1/lighthouse-core/config/constants.js#L20-L29)

そこを通常ブラウザで実行する場合と同様のネットワークを模したものとなるように変更したら問題は解消しました。設定値は Lighthouse のコードベースの以下箇所を参考にしています。
[GoogleChrome/lighthouse/blob/v6.4.1/lighthouse-core/config/constants.js#L43-L51](https://github.com/GoogleChrome/lighthouse/blob/v6.4.1/lighthouse-core/config/constants.js#L43-L51)

サンプルプロジェクトの設定例だとこちらの `throttling` の部分です。

[webperf-watcher-sample/blob/master/src/constants.ts](https://github.com/kawamataryo/webperf-watcher-sample/blob/master/src/constants.ts#L27-L34)

```ts:src/constants.ts
// ...
export const LIGHTHOUSE_OPTIONS = {
  // ...
  settings: {
    // ...
    throttling: {
      rttMs: 40,
      throughputKbps: 10 * 1024,
      cpuSlowdownMultiplier: 1,
      requestLatencyMs: 0,
      downloadThroughputKbps: 0,
      uploadThroughputKbps: 0,
    },
    // ...
  },
};
//...
```

# おわりに

以上簡単ですが、Lighthouse + Puppeteer + GitHub Actions で作るフロントエンドパフォーマンスの測定プロジェクトの紹介でした。
かなり手軽に実現できるのでおすすめです。まだまだ計測だけで、実際のパフォーマンス・チューニングには至ってないのですが、今後頑張っていきたいです。

# 参考

- https://github.com/kawamataryo/puppeteer-webperf
- https://ceblog.mediba.jp/post/186341145447/webperf-measuring-with-lighthouse-and-datadog
