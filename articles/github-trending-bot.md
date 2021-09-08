---
title: "GitHub Trendingを定期的につぶやくTwitter BotをFirebaseで作ってみた"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Firebase", "TypeScript", "GitHub", "Twiter"]
published: true
---

# 📦 作ったもの

[GitHub Trending](https://github.com/trending)に掲載されたリポジトリを定期的につぶやく Twitter Bot を作りました。
全体のトレンドをつぶやく Bot と、JavaScript・TypeScript のトレンドをつぶやく Bot の 2 種類があります。

|[@gh_trending_](https://twitter.com/gh_trending_)|[@gh_trending_js](https://twitter.com/gh_trending_js)|
|---|---|
|![](https://user-images.githubusercontent.com/11070996/132124873-b698f5ee-5f7f-4d71-93bb-fd52763c7603.png)|![](https://user-images.githubusercontent.com/11070996/132124876-5f8ba485-231c-4008-8fe3-e628e4b547b9.png)|

仕様はこちらです。

- 30 分から 1 時間おきに GitHub Trending に掲載されているリポジトリをツイート
- 一度ツイートたリポジトリは再度掲載されていても 1 週間はつぶやかない
- 投稿内容はリポジトリ名、URL、スター数、作者の Twitter アカウント、言語、概要

https://twitter.com/gh_trending_js/status/1435200188352765955

技術スタックは Firebase Cloud Functions と Firebase Firestore です。
実装コードは全て以下リポジトリで公開しています。

https://github.com/kawamataryo/github-trending-bot


# 💪 モチベーション

複数のつよい人から GitHub Trending を定期的に見て最新の情報をキャッチアップしているという話を聞いたのがはじまりです。自分も見る習慣をつけようと思ったのですがなかなか続かず、それなら毎日見る Twitter のタイムラインで見れるようにすればよいのでは？　と怠惰なモチベーションではじめました。

:::message
最初に既存の GitHub Trending をつぶやく Twitter Bot がないか探しました。ただ、[@TrendingGitHub](https://twitter.com/trendinggithub)や、[@github_trending](https://twitter.com/github_trending)など複数見つかるものの、大抵 2019 年あたりで更新が止まっていたので自分で作ってみることにしました。
:::


# 🎯 技術的なポイント

## GitHub Trendingのリポジトリ情報の取得
GitHub Trending のリポジトリ情報は、GitHub 公式の API がなかったので、定期的にページをスクレイピングして取得しています。

Node.js でスクレイピングといえば Puppeteer や Playwright ですが、今回は GitHub Trending のページが静的なページだったので、単純に GET リクエストを投げてそのレスポンスの HTML をパースするという方法で行っています。

HTML のパースには[node-html-parser](https://www.npmjs.com/package/node-html-parser)を使っています。

https://www.npmjs.com/package/node-html-parser

[functions/src/lib/ghTrendScraper.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/ghTrendScraper.ts)
```ts:functions/src/lib/ghTrendingScrapper.ts
export class GHTrendScraper {
  static async scraping(params = ""): Promise<GHTrend[]> {
    const res = await got.get(`https://github.com/trending${params}`);
    const dom = parse(res.body);
    const rows = dom.querySelectorAll(".Box-row");

    return await Promise.all(
      rows.map(async (row) => {
        const { owner, repository } = GHTrendScraper.getOwnerAndRepoName(row);
        const { description } = GHTrendScraper.getDescription(row);
        const { starCount } = GHTrendScraper.getStarCount(row);
        const { forkCount } = GHTrendScraper.getForkCount(row);
        const { todayStarCount } = GHTrendScraper.getTodayStarCount(row);
        const { language } = GHTrendScraper.getLanguage(row);
        const { ownersTwitterAccount } =
          await GHTrendScraper.getOwnersTwitterAccount(owner);

        return {
          owner,
          repository,
          language: language ?? "",
          description: description ?? "",
          starCount: starCount ?? "",
          forkCount: forkCount ?? "",
          todayStarCount: todayStarCount ?? "",
          ownersTwitterAccount: ownersTwitterAccount ?? "",
          url: `https://github.com/${owner}/${repository}`,
        };
      })
    );
  }

  private static getOwnerAndRepoName(dom: HTMLElement) {
    const path = dom.querySelector("> h1 a").attributes.href;
    const result = path.split("/");
    return {
      owner: result[1],
      repository: result[2],
    };
  }
  // ...
}
```

そして、スクレイピングで取得した情報は Firestore に保存しています。
その時の保存条件として、1 週間以内にすでに追加済みのリポジトリは除外しています。

[functions/src/lib/firestore.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/firestore.ts)

```ts
//...
const getTrendDataWithinOneWeek = async (
  collectionRef: FirebaseFirestore.CollectionReference
): Promise<GHTrend[]> => {
  const oneWeekAgo = dayjs().add(-7, "day").unix();
  const querySnapshotFromOneWeekAgo = await collectionRef
    .where("createdAt", ">=", oneWeekAgo)
    .get();
  return querySnapshotFromOneWeekAgo.docs.map((doc) => doc.data() as GHTrend);
};

export const bulkInsertTrends = async (
  collectionRef: FirebaseFirestore.CollectionReference,
  trends: GHTrend[]
): Promise<void> => {
  const trendsFromOneWeekAgo = await getTrendDataWithinOneWeek(collectionRef);

  await Promise.all(
    trends.map(async (trend) => {
      // exclude repositories submitted within a week.
      if (trendsFromOneWeekAgo.some((d) => d.url === trend.url)) {
        return Promise.resolve();
      }
      return await collectionRef.add({
        ...trend,
        createdAt: dayjs().unix(),
        tweeted: false,
      });
    })
  );
};

//...
```

:::message
今回は後述するリポジトリオーナーの Twitter アカウント取得のためにもスクレイピングで良かったのですが、もっと簡単に行うのなら非公式の GitHub Trending の API 使うことも全然ありだと思います。
https://github.com/huchenme/github-trending-api
:::

### ツイートの定期実行

ツイートは Functions のスケジュール実行で 30 分ごとに行っています。
この時のツイート内容のデータはスクレイピングで取得した Firestore に保存されているものを使っています。

https://firebase.google.com/docs/functions/schedule-functions?hl=ja

[functions/src/pubsub/index.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/pubsub/index.ts)

```ts
export const tweetTrend = functions.pubsub
  .schedule("every 30 minutes")
  .onRun(async (_context) => {
    try {
      await Promise.all([tweetAllLanguagesTrends(), tweetFrontendTrends()]);
    } catch (e) {
      console.error(e);
    }
  });
```

Titter API を叩く際には Twitter-api-v2 というライブラリを利用しています。

https://www.npmjs.com/package/twitter-api-v2

[functions/src/lib/twitter/index.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/twitter.ts)

```ts
// ...
const createTweetText = (trend: GHTrend): string => {
  return `
📦 ${trend.repository}${
    trend.ownersTwitterAccount ? `\n👤 ${trend.ownersTwitterAccount}` : ""
  }
⭐ ${trend.starCount} (+${trend.todayStarCount})${
    trend.language ? `\n🗒 ${trend.language}` : ""
  }
${trend.description ? `\n${truncateText(trend.description, 85)}\n` : ""}
${trend.url}
`.trim();
};

const tweetFromTrend = async (
  client: TwitterApi,
  trend: GHTrend
): Promise<void> => {
  await client.v1.tweet(createTweetText(trend));
};

export const tweetRepository = async (
  collectionRef: FirebaseFirestore.CollectionReference,
  twitterClient: TwitterApi
): Promise<void> => {
  const snapshot = await getUntweetedTrend(collectionRef);
  if (snapshot.empty) {
    console.log("No matching documents.");
    return;
  }
  for (const doc of snapshot.docs) {
    await tweetFromTrend(twitterClient, doc.data() as GHTrend);
    await updateTweetedFlag(doc, true);
  }
};
```

# 🛠 工夫したところ

## 1つのFunctionsで実行タイミングを分ける

データを集めるスクレイピング、集めたデータからのツイートの実行は異なる処理なので、別の Functions で定義したほうがシンプルなのですが、Firebase には Functions の定期実行機能が 1 つのアカウントで 3 つまでという制限があるので、今回は 1 つの関数の中で実行時間を見て処理を間引くという形でそれぞれの実行タイミングを制御しています。
あまり良いやり方でもないのかもですが、今の所良い感じに機能しています。

[functions/src/lib/core/frontend.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/core/frontend.ts)

```ts:functions/src/lib/core/frontend.ts
// ...
export const tweetFrontendTrends = async (): Promise<void> => {
  // update trends data at several times a day.
  if (isUpdateTime()) {
    await updateFrontendTrends();
    console.info("Update frontend repositories collections");
  }

  // tweet trends repository with a bot
  await tweetRepository(collectionRef, twitterClient);
};
// ...
```

[functions/src/lib/lib/utils.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/lib/utils.ts)

```ts:functions/src/lib/lib/utils.ts
// ...
export const isUpdateTime = (): boolean => {
  const datetime = dayjs();
  return datetime.hour() % 2 === 0 && datetime.minute() <= 30;
};
// ...
```

## リポジトリオーナーのTwitterアカウントにメンションする
ツイート対象のリポジトリのオーナーページに、Twitter アカウントが設定されている場合は、ツイート時にそのアカウントにメンションするようにしました。
Twitter アカウントの情報はトレンドのスクレイピング時に一緒に取得しています。

[functions/src/lib/ghTrendScraper.ts](https://github.com/kawamataryo/github-trending-bot/blob/main/functions/src/lib/ghTrendScraper.ts)
```ts
// ...
  private static async getOwnersTwitterAccount(owner?: string) {
    if (!owner) {
      return {
        ownersTwitterAccount: null,
      };
    }
    const res = await got.get(`https://github.com/${owner}`);
    const dom = parse(res.body);
    const ownersTwitterAccount = dom
      .querySelector(".vcard-details a[href^=\"https://twitter.com\"]")
      ?.innerText.trim();
    return {
      ownersTwitterAccount,
    };
  }
// ...
```

狙いとしては、GitHub Trending に掲載されたことを本人もしらないと思うので知らせて上げたいという気持ち 1 割、そのメンションからこの Twitter アカウントが認知されてリポジトリオーナーのリツイートから海外のユーザーにも広がらないかなという打算的な狙い 9 割はです😅

実際にやってみるといくつかの投稿はリツイートしてもらえてそこから海外のフォロワーが増えているので一定の効果はありそうです。

https://twitter.com/gh_trending_/status/1435320986912575489

# 終わりに
初めての Twitter Bot の作成だったのですが、思いの他簡単に出来ました。Firebase 最高！
また、リリース後 2 日ですでに 200 人以上の方にフォローしてもらえて嬉しいです。フォロワー数として目に見えてユーザー数がわかるのは良いですね。

当初のモチベーション通り、GitHub Trending のリポジトリから技術キャッチアップして、つよいエンジニアを目指して頑張りたいです。