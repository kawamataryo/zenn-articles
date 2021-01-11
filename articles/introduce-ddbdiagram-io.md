---
title: "dbdiagram.io を使ってコードベースで手軽に ER 図を作成する"
emoji: "🖍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["database", "効率化"]
published: true
---

最近 ER 図を書く機会があって、その際に使ってみた [dbdiagram.io](https://dbdiagram.io/home) がとても良いツールだったので紹介します。

# dbdiagram.ioとは？


[dbdiagram.io](https://dbdiagram.io) は、ブラウザ上で手軽にデータベースの ER 図を作れるツールです。
とても直感的で分かりやすい記法で、テーブルの構成やリレーションを定義できます。

作った ER 図は PDF や PNG、MySQL, PostgreSQL など各種形式でエクスポートが可能です。そのほか、MySQL や PostgreSQL、Rails の schema.rb から ER 図を作成する import 機能もあります。

https://dbdiagram.io

![](https://i.gyazo.com/1f0b3be06d36e3ddfcc1d08b50309322.gif)

# 価格

通常利用は無料です。有料にすると様々な拡張機能が使えます。
※ この記事は無料版での利用を前提として書いています。

https://dbdiagram.io/pricing

![](https://i.gyazo.com/02e1d922730e8bf917acb71ecf66af5c.png)

# 使い方

簡単に使い方を解説します。

## テーブルの作成

テーブルは`Table`オブジェクトを書くことで定義できます。

以下、`id`を主キーとし`name`, `address`のカラムをもつ`users`テーブルの例です。

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

テーブル同士のリレーションは、カラムに`ref`属性を付与することで定義できます。

以下先ほどの`users`テーブルと多対一の関連を持つ`user_items`テーブルの例です。

```
Table user_items {
  id int [pk]
  user_id int [ref: > users.id]
  name varchar
  price int
}
```

これで以下のようなリレーションが作られます。

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

上記の書き方以外にも、以下のようにテーブル定義とは別に書く方法もあります。

```
ref: user_items.user_id > users.id
```

どちらもとても直感的ですね。

# diagram.io の利点・欠点

最後に簡単に diagram.io の利点と欠点をまとめます。

**👍利点**
- 記法が単純明快、直感的で分かりやすい（**一番強み**)
- エディタ上で補完が効くので入力しやすい
- エディタ上のエラーメッセージが分かりやすい
- テーブルの配置をドラッグ&ドロップで自由に変更できる
- 様々な形式で import, export 出来る

**👎欠点**
- UML 記法と異なり他プラットフォームでサポートされていない記法（GitHub, esa etc）
- URL でのシェア機能は有料なので、チームでの共有が少し面倒
- テーブルが多いと auto-arrange だけでは見辛く、配置を手動で調整する必要がある

# 終わりに

以上簡単ですが、[dbdiagram.io](https://dbdiagram.io/home) の紹介でした。
UML 記法をどうしても覚えられない自分としては、神ツールだと思っています（覚えよう😅）。
今後もちょこちょこ利用していきたいです。