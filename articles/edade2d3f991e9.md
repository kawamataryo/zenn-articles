---
title: "deno-puppeteerとGitHub ActionsでWEBサイトの外形監視をお手軽に"
emoji: "🚓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["deno", "githubactions", "puppeteer", "slack"]
published: false
---

はるか昔に受託で作って運用管理しているWPサイトが最近たびたび落ちるので、deno-puppeteerとGitHub Actionsとでお手軽に外形監視ツールを作ってみました。

# 🚨 作ったもの

以下のように毎日指定時刻に指定したURLにアクセスして、HTTPレスポンスの確認とスクショの撮影を実行、結果をSlackに通知してくれるツールです。

![](/images/edade2d3f991e9/2023-05-08-08-39-13.png)
_サイト画像はdummyです_

もし、200以外のステータスコードが返ってきた場合は、異常とみなしメンション付きで通知してくれます。

![](/images/edade2d3f991e9/2023-05-08-08-39-37.png)

異常があった場合にどんな状態だったか何日前から異常があったのかもSlackチャネルをみればすぐにわかるので便利です。

# 🛠️ 仕組み・実装

仕組みはとても単純です。
GitHub Actionsで定期実行するdenoのスクリプトで、指定したURLにアクセスして、HTTPレスポンスの確認とスクリーンショットの撮影を行い結果をSlackに送信しているだけです。

![](/images/edade2d3f991e9/2023-05-07-17-34-09.png)

:::message alert
PublicリポジトリのGitHub Actionsの定期実行の場合は注意が必要です。リポジトリに変化がないと、60日で停止します。Privateリポジトリであれば、この制限はないようです。
https://docs.github.com/en/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
:::

main関数はこちらです。`findInvalidSites` 関数で、指定したURLにアクセスして、200以外のステータスコードが返ってくるものを抽出しています。そして、`takeScreenshots` 関数で、スクリーンショットを撮影し、最後にSlackに結果を通知しています。

```ts:main.ts
const main = async () => {
  const slackClient = new SlackClient(
    Deno.env.get("SLACK_TOKEN")!,
    Deno.env.get("NOTIFICATION_SLACK_CHANNEL")!,
  );

  // ステータスコードの確認
  const invalidSites = await findInvalidSites(TARGET_SITES);

  // スクリーンショットの撮影
  const screenshots = await takeScreenshots(TARGET_SITES);
  const uploadImageLinks = await slackClient.uploadImage(screenshots);

  // Slackへの通知
  if (invalidSites.length === 0) {
    await slackClient.notify({
      text: "🎉 全てのサイトが正常に稼働しています。",
    });
  } else {
    await slackClient.notify(createInvalidSiteMessage(invalidSites));
  }
  await slackClient.notify({
    text: uploadImageLinks.map((link) => `<${link}| >`).join(" "),
  });

  console.log("Done.🎉");
};

if (import.meta.main) {
  main();
}
```

`findInvalidSites` ではHEADメソッドでステータスコードのみを取得して、200以外のレスポンスを返すサイトを抽出しています。

```ts:findInvalidSites.ts
export const findInvalidSites = async (
  sites: Site[],
): Promise<SiteWithStatus[]> => {
  const results = await Promise.all(sites.map(async (site) => {
    try {
      const res = await fetch(site.url, {
        method: "HEAD",
      });
      return {
        ...site,
        status: res.status,
      };
    } catch (e) {
      console.error("Failed to fetch: ", e);
      return {
        ...site,
        status: 500,
      };
    }
  }));
  return results.filter((result) => result.status !== 200);
};
```

`takeScreenshots` では、[deno-puppeteer](https://github.com/lucacasonato/deno-puppeteer)を使ってスクリーンショットを撮影するだけです。

https://github.com/lucacasonato/deno-puppeteer

```ts:takeScreenshots.ts
import puppeteer from "https://deno.land/x/puppeteer@16.2.0/mod.ts";

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export const takeScreenshots = async (sites: Site[]) => {
  await Deno.mkdir("./screenshots", { recursive: true });
  const browser = await puppeteer.launch({
    defaultViewport: { width: 1200, height: 900 },
  });
  const page = await browser.newPage();
  page.setUserAgent(
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.99 Safari/537.36",
  );

  const screenshots: Screenshot[] = [];

  for (const site of sites) {
    try {
      await page.goto(site.url);
      await wait(3000);
      const path = `screenshots/${site.name}.png`;
      await page.screenshot({ path });
      screenshots.push({
        name: site.name,
        path,
      });
    } catch (e) {
      console.error("Failed to screenshots: ", site.name, e);
    }
  }
  await browser.close();

  return screenshots;
};
```

slackへの通知部分では、[deno-slack-api](https://github.com/slackapi/deno-slack-api)を使っています。
事前にSlackアプリの登録と、scopeの設定（`chat:write`, `files:write`）が必要です。

https://github.com/slackapi/deno-slack-api

```ts:slackClient.ts
import { SlackAPI } from "https://deno.land/x/deno_slack_api@2.1.0/mod.ts";
import { SlackAPIClient } from "https://deno.land/x/deno_slack_api@2.1.0/types.ts";

export class SlackClient {
  private client: SlackAPIClient;

  constructor(private token: string, private channel: string) {
    this.client = SlackAPI(this.token);
  }

  async notify(
    params: { text: string; blocks?: unknown[] },
  ) {
    await this.client.chat.postMessage({
      channel: this.channel,
      ...params,
    });
  }

  async uploadImage(filePaths: Screenshot[]) {
    const uploadedFilesLinks: string[] = [];
    for (const file of filePaths) {
      // NOTE: client.files.uploadではなぜか送れないので、fetchを使う
      const image = await Deno.readFile(`./${file.path}`);
      const formData = new FormData();
      formData.append("file", new Blob([image], { type: "image/png" }));
      formData.append("filename", file.name);

      const response = await fetch("https://slack.com/api/files.upload", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${this.token}`,
        },
        body: formData,
      });
      const result = await response.json();
      if (result.ok) {
        uploadedFilesLinks.push(result.file.permalink);
      }
    }
    return uploadedFilesLinks;
  }
}
```

:::message
画像アップロードの箇所だけdeno-soack-apiではうまく送れなかったので以下記事を参考にしてfetchを使っています。
[Slack Web APIを使ってslackに任意の画像を複数枚投稿する](https://zenn.dev/yui/articles/2d965fedca620c)
:::



最後に、これらのスクリプトをGitHub Actionsで定期実行するためのymlです。
`cron` で毎朝9時（JST）に実行するようにしています。もし、頻度を調整したい場合はcronの値を調整すればOKです。

```yml:.github/workflows/monitor.yml
name: monitor

on:
  schedule:
    - cron:  '0 0 * * *' # JST 9:00 every day
  workflow_dispatch:

jobs:
  monitor:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - uses: denoland/setup-deno@main
        with:
          deno-version: "1.28.3"
      - name: Install Japanese fonts
        run: sudo apt-get install -y fonts-noto-cjk
      - name: Install Puppeteer
        run: PUPPETEER_PRODUCT=chrome deno run -A --unstable https://deno.land/x/puppeteer@16.2.0/install.ts
      - name: Cache dependencies
        run: deno task cache-deps
      - name: Run monitor
        run: PUPPETEER_PRODUCT=chrome deno task run
        env:
          SLACK_TOKEN: ${{ secrets.SLACK_TOKEN }}
          NOTIFICATION_SLACK_CHANNEL: ${{ secrets.NOTIFICATION_SLACK_CHANNEL }}
```

# 🎬 おわりに

構想から半日でサクッと実装できたので良かったです。これでもうサイトが落ちることに気づかず依頼者に怒られることはないはず...😅
