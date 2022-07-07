---
title: "Prismaを使ってFirestoreのデータをSQLiteにダンプする"
emoji: "🗻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["prisma", "sqlite", "firestore", "typescript"]
published: false
---

FirestoreのデータをSQLiteにダンプしてみたので、そのメモです。

# なぜFirestoreのデータをSQLiteに？

個人的にGitHub Trendingのリポジトリ情報を流すTwitter Botを運用しているのですが、その元データ（日々のGitHub Trendingのデータ）がFirestoreに溜まっていました。
それを使って何か分析したらおもしろいかなと思ったのですが、Firestoreのままデータ分析をするのが少し面倒で、慣れたSQLでデータを扱いと考えSQLiteへのデータ移行に行き着きました。
SQLiteを選んだ理由は、特にDBのセットアップが不要で、ファイル単位で管理できるというのが手軽で便利だったからです。

**運用中の TwitterBot**

|[@gh_trending_](https://twitter.com/gh_trending_)(3,724 Follower)|[@gh_trending_js](https://twitter.com/gh_trending_js)(2,860 Follower)|
|---|---|
|<a href="https://twitter.com/gh_trending_"><img width="597" alt="github trending" src="https://user-images.githubusercontent.com/11070996/132124873-b698f5ee-5f7f-4d71-93bb-fd52763c7603.png"></a>|<a href="https://twitter.com/gh_trending_js"><img width="600" alt="github trending javascript typescript" src="https://user-images.githubusercontent.com/11070996/132124876-5f8ba485-231c-4008-8fe3-e628e4b547b9.png"></a>|

|[@gh_trending_py](https://twitter.com/gh_trending_py)(1,164 Follower)|[@gh_trending_rs](https://twitter.com/gh_trending_rs)(279 Follower)|
|---|---|
|<a href="https://twitter.com/gh_trending_py"><img width="597" alt="github trending python" src="https://i.gyazo.com/4f76a7358a0822d3219a51b8c14962ad.png"></a>|<a href="https://twitter.com/gh_trending_rs"><img width="600" alt="github trending rust" src="https://i.gyazo.com/42b4a9e35cb6f51fdd8c089c44543b96.png"></a>|

**TwitterBotの作成記事**

https://zenn.dev/ryo_kawamata/articles/github-trending-bot


# Prismaのセットアップ

# RDBの構成

# 実装