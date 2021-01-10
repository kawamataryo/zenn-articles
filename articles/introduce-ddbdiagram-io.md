---
title: "素敵な ER 図作成ツール dbdiagram.io で手軽にER図を書こう"
emoji: "🧰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ER図", "database", "tool", "効率化"]
published: false
---

最近 ER 図を書く機会があって、その時使ってみた [dbdiagram.io](https://dbdiagram.io/home) がとても良いツールだったので紹介します。

# dbdiagram.ioとは？

https://dbdiagram.io

dbdiagram.io はブラウザ上で、手軽に ER 図を作れるツールです。
とても分かりやすい独自の記法でテーブルの構成や、リレーションを定義できます。

作った ER 図は PDF や PNG、MySQL, PostgreSQL など各種形式でエクスポートが可能です。
そのほか、MySQL や PostgreSQL、Rails の schema.rb から ER 図を作成する import 機能もあります。

![](https://i.gyazo.com/1f0b3be06d36e3ddfcc1d08b50309322.gif)

# 価格

通常利用は無料です。有料にすると様々な拡張機能が使えるようです。
※ この記事は無料版での利用を前提としています。

https://dbdiagram.io/pricing

![](https://i.gyazo.com/02e1d922730e8bf917acb71ecf66af5c.png)

# 使い方

簡単に使い方を解説します。

## テーブルの作成

テーブルは`Table`オブジェクトを作成することで定義できます。

以下`id`を主キーとして持ち、`name`, `address`の情報をもつ`users`テーブルの例です。

```
Table users {
  id int [pk]
  name varchar
  address varchar
}
```

これで以下のようなエンティティが表示されます。

![](https://i.gyazo.com/72efaeebd2945ed67748c8b37833ea9b.png)


記法はこちらです。

```
Table テーブル名 {
  カラム名 型 属性（プライマリーキー, リレーション etc）
}
```

## 関連の作成

ER 図でとても重要な要素の関連の作成は、`ref`属性を付与することで設定できます。

以下先ほどの users テーブルと多対一の関連を持つ`user_items`テーブルの例です。

```
Table user_items {
  id int [pk]
  user_id int [ref: > users.id]
  name varchar
  price int
}
```

これで以下のようなエンティティと関連が作られます。

![](https://i.gyazo.com/abc5385d3ea545d6e064bc481126843b.png)


記法はこちらです。
関連の種類は 3 種類あります。
- `<` 一対多
- `>` 多対一
- `-` 一対一

```
Table foo {
  foo_id int [ref: 関連の種類(<, -, >) テーブル名.カラム名]
}
```

この書き方以外にも以下のようにテーブル定義とは別に書く方法もあります。

```
ref: user_items.user_id > users.id
```

どちらもとても直感的ですね。

# diagram.io の利点・欠点

簡単に利点と欠点をまとめます。

**👍利点**
- 記法が単純明快、直感的で分かりやすい（**一番強み**）
- エディタ上で補完が聞くので入力しやすい
- エディタ上のエラーメッセージが分かりやすい
- 作られたエンティティの配置を自由に変更できる
- 様々な形式で import, export 出来る

**👎欠点**
- `0対多`か`一対多`かなどの詳細なリレーションが定義できない
- テーブルが多いと auto-arrange だけでは見辛く、配置を手動で調整する必要がある

# 終わりに

以上簡単ですが、[dbdiagram.io](https://dbdiagram.io/home) の紹介でした。
UML 記法をどうしても覚えられない自分としては、神ツールだと思っています（覚えよう😅）。
今後もちょこちょこ利用していきたいです。