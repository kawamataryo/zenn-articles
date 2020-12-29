---
title: "自分のZenn記事をインクリメンタル検索できるAlfred Workflowを作った"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["alfred", "zenn", "js"]
published: true
---

Zenn 公開時から投稿を続け、累計投稿数は 20 記事を超えました。
自分の詰まったところベースで記事を書くこともあり、投稿記事を後で見返すことも多いです。その際にもっと手軽に探せたらなと思い、自分の投稿記事を検索する Aflred Workflow を作ってみました。

結構便利なので、Zenn ヘビーユーザーの方々に使ってもらえると嬉しいです。

# alfred-zenn-posts

自分の公開済みの投稿記事 & スクラップをインクリメンタル検索できる Alfred Workflow です。
検索結果をクリックすればブラウザで該当ページが開かれます。

初回は API リクエストが走るので少し時間がかかりますが、その後はキャッシュが効くのである程度高速に動作します。

![](https://i.gyazo.com/aec9563c479ab859572b59e81638d93f.gif)

https://github.com/kawamataryo/alfred-zenn-posts

# インストール・環境変数の設定

[GitHubのリリース](https://github.com/kawamataryo/alfred-zenn-posts/releases) から直接`alfred-zenn-posts.workflow`ファイルをダウンロードし、Alfred に取り込こみます。

![](https://i.gyazo.com/5dabe1ad82faac8da2f515041afe939c.png)


インストールが完了したら、Alfred の Workflow 設定画面から `alfred-zenn-posts` を選択して、環境変数を設定してください。


設定する環境変数は以下 1 つです。

|名前|値|
|---|---|
|USER_NAME | Zenn のユーザー名（@を除く）を設定してください。|

![](https://i.gyazo.com/981c410c2f98c0a0ec4592362ef00ee5.png)


これで準備は完了です。
Alfred を開き`zenn`と打ち、半角スペースを空けて検索したい文字列を入力するだです。

![](https://i.gyazo.com/aec9563c479ab859572b59e81638d93f.gif)

# 実装

以前、[こちらの記事](https://qiita.com/ryo2132/items/358ce9d4baa8a8a27092)で紹介した[Alfy](https://github.com/sindresorhus/alfy) という Node.js ベースで Alfred Workflow を作れるフレームワークを使っています。

主要ファイルは以下 3 つです。

（1）全ての公開記事を取得

```js:lib/fetchAllPosts.js
import alfy from "alfy"
import { emojiPath } from "./emojiPath";

export const fetchAllPosts = async (username) => {
  const options = {
    json: true,
    maxAge: 60000
  };

  const res = await alfy.fetch(
    `https://api.zenn.dev/users/${username}/articles`,
    options
  );
  const posts = res.articles.map((p) => {
    p.url = `https://zenn.dev/${username}/articles/${p.slug}`
    p.icon = emojiPath(p.emoji)
    return p
  })

  return posts
};

```

（2）全てのスクラップの取得

```js:lib/fetchAllScraps.js
import alfy from "alfy"
import { emojiPath } from "./emojiPath";

export const fetchAllScraps = async (username) => {
  const options = {
    json: true,
    maxAge: 60000
  };

  const res = await alfy.fetch(
    `https://api.zenn.dev/users/${username}/scraps`,
    options
  );
  return res.scraps.map((p) => {
    p.url = `https://zenn.dev/${username}/scraps/${p.slug}`
    p.icon = emojiPath("📑")
    return p
  })
};
```

（3）Alfy API でのインクリメンタル検索

```js:lib/index.js
import alfy from "alfy";
import { fetchAllPosts } from "./lib/fetchAllPosts";
import { fetchAllScraps } from "./lib/fetchAllScraps";

const username = process.env.USER_NAME;

const createItem = (title, subtitle, url, icon) => {
  return {
    title,
    subtitle,
    arg: url,
    icon: {
      type: "png",
      path: icon,
    }
  };
}

(async () => {
  if(alfy.input.length > 1) {
    const posts = await fetchAllPosts(username);
    const scraps = await fetchAllScraps(username);

    const items = alfy.inputMatches([...posts, ...scraps], "title").map((p) => createItem(p.title, p.url, p.url, p.icon));

    if (!items.length) {
      alfy.output(
          [createItem("The requested post was not found. ⚠️", "", "")]);
      return;
    }
    alfy.output(items);
  } else {
    alfy.output([createItem("Loading...", "", "")])
  }
})()
```

Alfy の API を使えば、手軽に API レスポンスをインクリメンタル検索できる Workflow が作れます。おすすめです。

# 終わりに

以上「自分の Zenn 記事を手軽に検索できる Alfred Workflow を作った」でした。
Zenn の投稿数が多いほうには何かと便利だと思います。使ってもらえると嬉しいです。
（もし動かない場合は、[Twitter](https://twitter.com/KawamataRyo)か[GitHub Issue](https://github.com/kawamataryo/alfred-zenn-posts/issues), この記事のコメントで教えてください :pray:）

その他、Qiita ユーザーの方には Qiita 版も公開しているのでこちらも是非。

https://qiita.com/ryo2132/items/6909b534a59d3550093e