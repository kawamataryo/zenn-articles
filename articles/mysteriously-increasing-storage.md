---
title: "Cloud Functions for Firebaseの利用で異様にGCPのStorageが消費されると思ったら..."
emoji: "📛"
type: "tech"
topics: ["firebase", "gcp", "cloudFunctions", "CloudStorage"]
published: true
---

知っている人には当たり前の知識かもですが、地味に不安だったことなのでメモ。

# 起こったこと

Firebase でアプリを作っていたら、なぜか Storage のリソースが異様に消費されてることに気がつきました。

![](https://storage.googleapis.com/zenn-user-upload/zbsz812u2ebw83a10yf82baba0qo)

このプロジェクト自体は Cloud Functions for Firebase をメインに使っていて、Storage 自体にはほぼファイルはありませんでした。

しかし、上記画像の通り異様にアクセスされています。
さらに GCP で Storage をみると見覚えのないファイルが大量に！

![](https://storage.googleapis.com/zenn-user-upload/cm10fr7pw6wdzuphs03y2thb579s)

なぜ...🤔

# 原因

原因は Cloud Functions for Firebase のデプロイ時に自動保存されるソースコードでした。

Cloud Functions for Firebase のデプロイは以下手順で行われるようです。

1. 関数のソースコードを含むアーカイブが Cloud Storage のバケットにアップロード
2. アップロードされた ソースコードを使って Cloud Build でコンテナイメージを構築
3. 構築されたイメージを Container Registry にプッシュ
4. Functions のコール時に Container Registry からコンテナを作成して実行

なので、ローカルから `firebase deploy --only functions` をするたびに Cloud Storage にバケットが作成されていたのですね。


※ 上記内容は所属しているコミュニティ（[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)）の Slack で教えてもらったことです。疑問を分報に書いたら即 [@moga](https://twitter.com/_mogaming) 氏 からコメントもらえました。本当に感謝🙏

# 対処

このままだと Storage のリソースが消費され、無駄な費用がかかることもあるので適宜削除しましょう。
Cloud Storage のライフサイクルで削除を実行するのがオススメです。

以下手順で設定を行いました。

（1）GCP で Storage コンソールを開く

![](https://storage.googleapis.com/zenn-user-upload/mzrg1prfy9dipshf9u2iym2x9qb9)

（2）対象のバケットを選択しライフサイクルのタブを開く

![](https://storage.googleapis.com/zenn-user-upload/ymk2vb7hjh3tflx0tm1g38c62mca)


（3）ルールを追加を選択し、「オブジェクトを削除する」を選ぶ

![](https://storage.googleapis.com/zenn-user-upload/5ljzznfubqrb6rqeq4x9x2f0jedo)

（4）条件を「年齢」値を「1 日」にする

![](https://storage.googleapis.com/zenn-user-upload/hbz90zsl09xpstkcylfbs3aq94bn)

これでバケットに追加されてから 24 時間経過後に自動的に削除されます。
解決！

※ バケットを削除しても Cloud Functions for Firebase の実行自体は問題なく動作します（起動用の実態は Container Registry にあるので）。

# 参考
- [Cloud Functions のデプロイ  |  Google Cloud Functions に関するドキュメント](https://cloud.google.com/functions/docs/deploying)
