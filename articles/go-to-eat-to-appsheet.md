---
title: "県のGo To Eat サイトが使いづらかったのでPlaywright + AppSheetで自分用アプリを作ってみた話"
emoji: "🦞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["playwright", "スクレイピング", "AppSheet", "typescript", "NoCode"]
published: true
---

Go To Eat とてもお得な施策で良いのですが「どのお店が使えるんだろう？」ってなりますよね。そんな時に見る茨城県の Go To Eat キャンペーンサイトが思いの外使いづらかったので、Playwright + AppSheet で自分用アプリを作ってみた話です。

# なぜ作った？


茨城県の Go To Eat キャンペーンサイトはこちらです。
トップページは良い感じです。

https://www.gotoeat-ibaraki.com/

![](https://storage.googleapis.com/zenn-user-upload/3zvhcn2e2ziaqzlsxahw5fuvoauy)


が…

店舗一覧ページに遷移するといきなり古き良き業務システムのような感じになります😇

![](https://storage.googleapis.com/zenn-user-upload/p1z4pveq4znagc1c86jzepmfcwqz)

このページだと、

- スマホ対応してないのでスマホで見辛い
- マップ上での検索が出来ないので、現在地周辺の店舗を探すのが面倒

ということでノーコードツールの [AppSheet](https://www.appsheet.com/) で作り直してみました。

:::message
こういう形式の店舗一覧アプリを作る場合は、[Glide](https://www.glideapps.com/)も便利なのですが、Glide だと無料枠に 500 行の制限があるので、今回は AppSheet としました。
Glide については以前こちらの記事でまとめています。

[ノーコード！ Glideで飲食店検索アプリを爆速で作ってみた - Qiita](https://qiita.com/ryo2132/items/af8f9324832bcfcfdfcb)
:::




# 作ってみたもの

AppSheet で作ってみたものがこちらです。

![](https://storage.googleapis.com/zenn-user-upload/22d8qznkt9vayx9a60ekytvrt1fl)

スマホに対応し、店舗名や要素での検索や Maps からの現在地周辺での店舗検索も出来るので、自分用には必要十分です。

この作り方を次項から解説していきます。

:::message
スクレピングでデータを取得する場合は、利用規約に注意してください。
今回のアプリは一般公開せずあくまで個人利用のみで使います。
:::


# 作成方法

## 1. Playwrightでのスクレイピング

アプリを作るためにはデータが必要なので、それをスクレイピングで取得します。
スクレイピングには Node.js のスクレイピングライブラリの[Playwright](https://github.com/microsoft/playwright)を利用します。

:::message
Node.js のスクレイピングといえば[Puppeteer](https://github.com/puppeteer/puppeteer)が有名ですが、今回はお試しをかねて Playwright を使ってみました。 Playwright は Puppettier から派生しているので、ほぼ同じ API で使えます。
:::

https://github.com/microsoft/playwright

## 1-1. 環境構築

まず、任意のディレクトリでプロジェクトを作成します。

```bash
$ yarn init -y
```

そして依存モジュールを追加します。

[json2csv](https://github.com/zemirco/json2csv) はオブジェクトの CSV にパースで使用します。

[ts-node](https://github.com/TypeStrong/ts-node)は Node.js での TypeScript の実行環境です。TypeScript を明示的なコンパイル不要で実行できます。

```bash
$ yarn add ts-node playwright typescript json2csv
```

ts-node の実行スクリプトを`package.json`に追加します。

```json:package.json
  "scripts": {
    "start": "ts-node index.ts"
  },
```

これで環境構築は完了です。

## 1-2. スクレイピング

`index.ts`を追加して以下のコードを記載します。

コードはスクレイピングの関係上ちょっと分かりにくいのですが、以下のような動きをします。

1. GoTo Eat 茨城の店舗一覧ページを開く
2. 店舗明細行ごとに店舗詳細ページに遷移 & 情報を取得
3. 全ページ終わったら結果を CSV 形式にパース
4. ファイル書き込み

```ts:index.ts
import { chromium } from "playwright";
import { Parser } from "json2csv";
import * as fs from "fs";

const options = {
  headless: false,
};

const BASE_URL =
  "https://area34.smp.ne.jp/area/table/27130/3jFZ4A/M?S=%70%69%6D%67%6E%32%6C%62%74%69%6E%64&_page_27130";
const PER_PAGE = 10;

(async () => {
  const stores = [];
  const browser = await chromium.launch(options);
  const page = await browser.newPage();
  const navigationPromise = page.waitForNavigation();

   await page.goto(BASE_URL);

  // 最大ページ数を取得
  const maxPage = await page.evaluate(() => {
    return Math.ceil(
      Number(document.querySelector(".smp-count").textContent) / 10
    );
  });

  // 最大ページ数の分だけループ
  for (let pageNumber = 1; pageNumber <= maxPage; pageNumber++) {
    const currentPageUrl = `${BASE_URL}=${pageNumber}`;
    await page.goto(currentPageUrl);

    // ５行目からが店舗データなので5始まりとする
    for (let rowNumber = 5; rowNumber < PER_PAGE + 5; rowNumber++) {
      // 詳細リンクを取得
      const detailLinkSelector = (rowNumber: number) => { return `#smp-table-27130 > tbody > tr.smp-row-${rowNumber}.smp-row-data .smp-cell-col-2 > a` }
      const detailLink = await page.$(detailLinkSelector(rowNumber));

      // 詳細リンクがなければスキップ
      if ( !detailLink ) { continue; }

      // 詳細ページに遷移してデータを保存
      await page.click(detailLinkSelector(rowNumber));
      await navigationPromise;

      const store = await page.evaluate(() => {
        const detailContentData = (rowNumber: number) => {
          return document.querySelector(`body > table > tbody > tr > td > div > div.whole > table > tbody > tr:nth-child(${rowNumber}) > td`)?.textContent?.trim() || ''
        }

        return {
          type: detailContentData(10),
          name: detailContentData(3),
          "phone number": detailContentData(8),
          address: detailContentData(6).replace(/\s+/g, "") + detailContentData(7),
          homepage: detailContentData(9),
          holiday: detailContentData(11),
          "business hours": detailContentData(12)
        };
      });

      await page.goto(currentPageUrl);
      await navigationPromise;

      stores.push(store)
    }
  }
  await browser.close();

  // CSVへのパース
  const parser = new Parser;
  const csv = parser.parse(stores);

  // ファイル書き込み
  fs.writeFileSync('goto-eat.csv', csv)
})();
```

あとは実行するだけです。

```bash
$ yarn start
```


Playwright の options で headless を false にしているので、こんな感じに chromium が立ち上がって動くと思います。

![](https://storage.googleapis.com/zenn-user-upload/p1dqbemhnmwiy4e8eh95dcqytoyv)


完了すると`goto-eat.csv`が出来上がります。
（260 ページあるのでかなり時間かかります）

![](https://storage.googleapis.com/zenn-user-upload/vxo84j46ctjssemim7v6n29kxd3s)


## 2. AppSheetでのアプリ作成

店舗データが出来上がったので、これを AppSheet でアプリ化していきます。

## 2-1. Google スプレッドシートの作成

まず、CSV を Google スプレッドシートで import します。

スプレッドシートを開いて `ファイル > インポート > アップロード`で先ほどの`goto-eat.csv` をアップロードすれば OK です。

https://www.google.com/intl/ja_jp/sheets/about/

![](https://storage.googleapis.com/zenn-user-upload/jwy2jszxhnbpsy6nv8uq0kp88rh6)

## 2-2. AppSheetでのインポート

[AppSheet](https://www.appsheet.com/) にアクセスします。

https://www.appsheet.com/

`Start for free`をクリックして、

![](https://storage.googleapis.com/zenn-user-upload/d3iv77xr7af87ypnj9fv789altgb)

`Make a new app` > `Start with your own data` を選択

![](https://storage.googleapis.com/zenn-user-upload/r0mfcg53lzqq71xkd7rjzl6i78ry)

`Choose your data`から先ほど作ったスプレッドシートを選択すれば OK です。


![](https://storage.googleapis.com/zenn-user-upload/vxwckllt144c875y0cy4msg77qt6)

インポートが完了すると、アプリが作成されます。
シートのヘッダーから自動的に App のカテゴリを認識して良い感じに作ってくれるはずです（すごい）。

![](https://storage.googleapis.com/zenn-user-upload/xhamdbm7ugzjx1m68wv7hdqflt8e)

あとは、ちょいちょい項目名とかを直してデプロイすれば完了です。

![](https://storage.googleapis.com/zenn-user-upload/22d8qznkt9vayx9a60ekytvrt1fl)

AppSheet のデプロイについてはこちらを参考にさせていただきました。

https://tonari-it.com/appsheet-free-deploy/

# おわりに

以上「茨城県の Go To Eat サイトが見づらかったので、Playwright + AppSheet で自分用に作り直した話」でした。

Playwright も AppSheet 初めて使ったのですが、どちらも使いやすいですね。

かかった時間は AppSheet のキャッチアップ含めて 3 時間くらいでした。今回みたいな限定した用途でアプリを作るには便利そうです。

三密気をつけながらこのアプリを使って Go To Eat していきたいです。