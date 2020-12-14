---
title: "Bolt.js⚡ + Firebase🔥で技術投稿の指標を良い感じに集計してくれるSlack Botを作った"
emoji: "🤖"
type: "tech"
topics: ["Firebase", "Bolt", "slack", "Firestore", "TypeScript"]
published: false
---

この記事は [Slack Advent Calendar 2020](https://qiita.com/advent-calendar/2020/slack) 16 日目の記事です。

Slack アプリのフレームワークである Bolt.js と Firebase を使って、自分のアウトプットの指標を良い感じに集計して、良い感じにフォーマットして出力してくれる Bot を作ってみたので紹介です。

# 作ったもの
自分の技術投稿のアカウント名を入力するとこんな感じに指標を集計して返してくれる Slack Bot を作りました。

![](https://i.gyazo.com/9d69d32a88460af4d59859cf77855fbe.png)

詳細な機能は以下の通りです。

1.  `/report` コマンドを打つと起動して入力モーダルをひらく
2. モーダルで Twitter, Qiita, Zenn, note のアカウント名 + コメントの入力が出来る
3. 投稿ボタンを押すと、入力内容から各指標を集計、結果をチャネルに投稿

対応しているサービスはこちらです。

|サービス|指標|
|---|---|
|Twitter|ツイート数、フォロワー数、フォロー数|
|Qiita|記事数、LGTM 数、フォロワー数|
|Zenn|記事数、Like 数、フォロワー数|
|note|記事数、Like 数、フォロワー数|

いかに楽に入力できるか、いかに楽に自分の進捗を把握できるかに着目して開発しました。
入力については、Slack アカウトごとに前回入力済みの内容を保存し、初回以降は入力欄の初期値として入るようにしています。
自分の進捗把握については、初回以降前回の実行時からの差分を表示するようにしています。なので週 1 回とかコマンドを打てば前週との比較が手軽に出来ます。

![](https://i.gyazo.com/bdb8e8a5ea3c7146119420b02c01a777.gif)


# 構成

どのような構成かざっくり紹介します。

Bolt.js を Cloud Functions for Firebase で動かして Slack とのやりとりをしています。
そして、フォームの入力内容・指標データを Firestore に保存しています。

![](https://i.gyazo.com/67a4d02c9df3e48926edbc2a08719b3e.png)

指標集計部分は自分で各サービスの API クライントを書いて使っています。

:::details Zenn の API クライアント
```ts
import {
  ZennArticle,
  Follower,
  ZennMyArticlesResponse,
  ZennMyFollowersResponse,
} from "../types/zennTypes";
import axios from "axios";
import { ApiClient, ZennIndex } from "../types/types";

export class ZennClient implements ApiClient {
  private readonly BASE_API_URL = "https://api.zenn.dev";

  constructor(private userName: string) {
    axios.defaults.baseURL = this.BASE_API_URL;
  }

  async fetchIndex(): Promise<ZennIndex> {
    const articles = await this.fetchMyAllArticles();
    const followers = await this.fetchMyFollowers();

    return {
      postCount: articles.length,
      likeCount: this.tallyUpLikeCount(articles),
      followerCount: followers.length,
    };
  }

  private async fetchMyAllArticles(): Promise<ZennArticle[]> {
    const response = await axios.get<ZennMyArticlesResponse>(
      `/users/${this.userName}/articles`
    );
    return response.data.articles ?? [];
  }

  private async fetchMyFollowers(): Promise<Follower[]> {
    let followers = [] as Follower[];
    let hasNextPage = true;

    try {
      for (let page = 1; hasNextPage; page++) {
        const response = await axios.get<ZennMyFollowersResponse>(
          `/users/${this.userName}/followers?page=${page}`
        );
        hasNextPage = !!response.data.next_page;
        followers = [...followers, ...response.data.users];
      }
    } catch (e) {
      console.error(e);
    }

    return followers;
  }

  private tallyUpLikeCount(articles: ZennArticle[]): number {
    return articles.reduce<number>((count, article) => {
      return count + article.liked_count;
    }, 0);
  }
}
```　
:::

:::details Qiita の API クライアント
```ts
import * as functions from "firebase-functions";
import axios from "axios";
import { QiitaItem, QiitaUser } from "../types/qiitaTypes";
import { ApiClient, QiitaIndex } from "../types/types";

export class QiitaClient implements ApiClient {
  private readonly BASE_URL = "https://qiita.com/api/v2";
  private readonly PER_PAGE = 100;

  constructor(private userName: string) {
    axios.defaults.baseURL = this.BASE_URL;
    axios.defaults.headers["Authorization"] = `Bearer ${
      functions.config().token.qiita
    }`;
  }

  async fetchIndex(): Promise<QiitaIndex> {
    const user = await this.fetchUser();
    const items = await this.fetchAllItems(user);
    const lgtmCount = this.tallyUpLgtmCount(items);

    return {
      postCount: user.items_count ?? 0,
      lgtmCount: lgtmCount,
      followerCount: user.followers_count ?? 0,
    };
  }

  private async fetchUser() {
    const response = await axios.get<QiitaUser>(`/users/${this.userName}`);
    return response.data;
  }

  private async fetchAllItems(user: QiitaUser | null) {
    if (!user) {
      return [];
    }
    // 最大ページ数
    const maxPage = Math.ceil(user.items_count / this.PER_PAGE);
    // 投稿一覧の取得
    let allItems = [] as QiitaItem[];
    await Promise.all(
      [...Array(maxPage).keys()].map(async (i) => {
        const items = await this.fetchItems(i + 1, this.PER_PAGE);
        allItems = [...allItems, ...items];
      })
    );
    return allItems;
  }

  private async fetchItems(page: number, perPage: number) {
    const response = await axios.get<QiitaItem[]>(
      `/items?page=${page}&per_page=${perPage}&query=user:${this.userName}`
    );
    return response.data;
  }

  private tallyUpLgtmCount(items: QiitaItem[]) {
    const lgtmCount = items.reduce(
      (result, item) => result + item.likes_count,
      0
    );
    return lgtmCount;
  }
}
```
:::

:::details note の API クライアント
```ts
import axios from "axios";
import {
  NoteContent,
  NoteContentsResponse,
  NoteUserResponse,
} from "../types/noteTypes";
import { ApiClient, NoteIndex } from "../types/types";

export class NoteClient implements ApiClient {
  private readonly BASE_URL = "https://note.com/api/v2";

  constructor(private userName: string) {
    axios.defaults.baseURL = this.BASE_URL;
  }

  async fetchIndex(): Promise<NoteIndex> {
    const user = await this.fetchUser();
    const contents = await this.fetchAllContent();

    return {
      postCount: user.noteCount ?? 0,
      likeCount: this.tallyUpLikeCount(contents),
      followerCount: user.followerCount ?? 0,
    };
  }

  private async fetchUser() {
    const response = await axios.get<NoteUserResponse>(
      `/creators/${this.userName}`
    );
    return response.data.data;
  }

  private async fetchAllContent() {
    let contents = [] as NoteContent[];
    let isLastPage = false;

    try {
      for (let page = 1; !isLastPage; page++) {
        const responseData = await this.fetchContents(page);
        isLastPage = responseData.isLastPage;
        contents = [...contents, ...responseData.contents];
      }
    } catch (e) {
      console.log(e);
    }

    return contents;
  }

  private async fetchContents(page: number) {
    const response = await axios.get<NoteContentsResponse>(
      `/creators/${this.userName}/contents?kind=note&page=${page}`
    );
    return response.data.data;
  }

  private tallyUpLikeCount(contents: NoteContent[]) {
    const likeCount = contents.reduce(
      (result, content) => result + content.likeCount,
      0
    );
    return likeCount;
  }
}
```
:::

:::details Twitter の API クライアント
```ts
import axios from "axios";
import { PublicMetrics, UsersResponse } from "../types/twitterTypes";
import * as functions from "firebase-functions";
import { ApiClient, TwitterIndex } from "../types/types";

export class TwitterClient implements ApiClient {
  private readonly BASE_URL = "https://api.twitter.com/2";

  constructor(private userName: string) {
    axios.defaults.baseURL = this.BASE_URL;
    axios.defaults.headers["Authorization"] = `Bearer ${
      functions.config().token.twitter
    }`;
  }

  async fetchIndex(): Promise<TwitterIndex> {
    const metrics = await this.fetchUserMetrics();

    return {
      tweetCount: metrics.tweet_count,
      followersCount: metrics.followers_count,
      followingCount: metrics.following_count,
    };
  }

  private async fetchUserMetrics(): Promise<PublicMetrics> {
    const response = await axios.get<UsersResponse>(
      `/users/by/username/${this.userName}?user.fields=public_metrics`
    );
    return response.data.data.public_metrics;
  }
}
```
:::

:::message alert
Zenn、note は 公式な API が公開されてないので、ネットワーク情報からパス、レスポンスをみて構築しています。各サービスの今後の改修で使えなくなる可能性はあります。
:::


コードについては全てリポジトリで公開しています。
API クライアントの詳細や、Bolt.js での実装など詳細は以下をご覧ください。

https://github.com/kawamataryo/blog-index

# 実装で詰まったところ

今回の Slack Bot 作成においてハマりポイントがあったので紹介します。

## Faas で Bolt.js を使う際の問題

実は Cloud Functions for Firebase や AWS Lambda などの FaaS 上で、Bolt.js による Slack Bot の構築は以下制約があるため少し工夫が必要です。

- （1）Slack から HTTP リクエストを受けたら 3 秒以内に HTTP レスポンスを返さなければならない
- （2）HTTP リクエストをトリガーに起動する FaaS では処理途中でレスポンスを返すと、後続の処理の実行は確約されない

（1）の 3 秒以内応答ルールの制約があるため Bolt.js では、`ack()`という関数が提供されています。
`ack()`は 200OK の HTTP レスポンスを返す関数で、それを実行すれば後続の処理を非同期で行うことができます。

```js
app.action('approve_button', async ({ ack, say }) => {
  // アクションリクエストの確認。この時点でレスポンスは返される
  await ack();
  // 以降は非同期で処理される。3秒以内応答の制約はない。
  await superHeavyTask()
});
```

ただ、これは、（2）の通り FaaS では利用できません。
というのも `ack()` を実行した瞬間に、その環境の処理は完了したと見なされ後続の処理の実行は確約されないからです。

これに対応するため Bolt.js 側に`ack()`の実行を処理完了まで遅らせるオプション（`processBeforeResponse`）もあるのですが、これを設定しても（1）の 3 秒以内のルールは守らなくてはなりません。

今回の Bot ではモーダルで受け取った値を利用して、各サービスの API にアクセスして結果を集計するので、どうしても 3 秒以内のレスポンスのルールが守れずタイムアウトエラーとなってしまう問題に突き当たりました。

同期的に処理を行ってしまうとタイムアウトエラー、でも Functions の関数内で非同期に実行すると FaaS の設計上、実行が確約されない・・さてどうするか🤔

:::message
Bolt.js の Python 版である[bolt-python](https://github.com/SlackAPI/bolt-python)ではこの問題を解決できるようです。詳細は、以下をご覧ください。
[Bolt for Python が FaaS での実行のために解決した課題 - Qiita](https://qiita.com/seratch/items/6d142a9128c6831a6718)
:::

## 解決策

今回は Firestore を Queue 的に使い、モーダルのレスポンスと集計処理の実行を Function 単位でを分離することで前述の問題を回避しました。

処理の実行手順は以下のようになります。

**スラッシュコマンド実行時からモーダルの表示まで**

1. スラッシュコマンドの実行
2. 実行ユーザーの過去投稿をデータを Firestore から取得
3. 過去投稿があればフォームの初期値として設定
4. モーダルを表示

![](https://i.gyazo.com/7877d7be4a5e201459565ecdaba48d7c.png)

**モーダルでの送信から指標の集計、チャネルへの投稿まで**

1. モーダルで内容入力・送信
2. Firestore にデータを保存
3. 200 レスポンスを返しモーダルを閉じる
4. onCreate のフックで別関数が起動
5. 各指標を集計
6. 結果を投稿


![](https://i.gyazo.com/e5cc7714869af3c9ee02452df58d3ddf.png)

特に重要なのはモーダルでの送信からの処理で、ここで Firestore のデータ保存 -> onCreate で別関数起動という方法をとることで Function 単位で処理を分離して、前述の制約を回避しています。
これなら指標集計にどれほど時間がかかっても、もうモーダルへのレスポンスは完了しているので、タイムアウトで落ちることはありません。

# 終わりに
以上「Bolt.js⚡ + Firebase🔥で技術投稿の指標を良い感じに集計してくれる Slack Bot を作る」でした。
参加しているコミュニティ（[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)）の Slack でも皆に使ってもらっているので嬉しいです。

もし、需要あれば公開アプリとしてみようかなとも思っています。反応もらえると嬉しいです。

# 参考

- [Slack | Bolt for JavaScript](https://slack.dev/bolt-js/ja-jp/concepts)
- [Bolt for Python が FaaS での実行のために解決した課題 - Qiita](https://qiita.com/seratch/items/6d142a9128c6831a6718)