---
title: "XcodeでもVimキーバインドを使いたい! XVim2のススメ"
emoji: "✌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "Xcode", "ios", "Swift"]
published: true
---

最近 Swift UI を初めているのですが、普段のエディタが Vim キーバインドなので、どうしてもそれとの違いで捗らず悩んでいたところ、XVim2 という Xcode で vim キーバインドを設定できるプラグインを見つけました。
使ってみると意外に動きが良いので紹介します。

この記事は以下バージョン時点の情報です。

```
Mac: macos catalina 10.15.5
Xcode: 12.1
XVim2: commit 796850133ddc9e17ea0584faa3727b9e871b6c55
```

# Keychain Accessでの証明書の発行

最初に Xcode の署名の書き換えが必要です。
これは XVim2 を Xcode のプラグインとしてロードするために必要な作業です。

:::message
書き換え理由などについては以下リンクに詳細があります。
セキュリティ的に怖いなどの気持ちもあると思うので、必ず確認してください。
https://github.com/xvimproject//blob/master/signing_xcode.md
:::

アプリケーションから「Keychain Access」を起動し、左のパネルより、`login`を選択します。
その状態で、`Keychain Access -> Certificate Assistant -> Create a Certificate` を選択します。

![](https://storage.googleapis.com/zenn-user-upload/f3ueav4p7mhqmke9ju02vjsh6nkh)

すると新しい証明書を作成する画面が出るので以下を入力・選択しましょう。

- name: `Xcodesigner`
- identity type: `self signed root`
- certificate type: `code signing`

![](https://storage.googleapis.com/zenn-user-upload/4woaf3t3f5baydjh73b4abtqjq6b)

create を押して以下画面が出れば OK です。

![](https://storage.googleapis.com/zenn-user-upload/jfl0fjgv1koprontybgin40eqlzj)

この証明書を使って Xcode に署名します。

```bash
$ sudo codesign -f -s Xcodesigner /applications/Xcode.app
```

この操作にはしばらく時間がかかるはずです。
正常終了すれば完了です。

# XVim2のインストール

任意の場所で XVim2 のリポジトリをクローンします。

```bash
$ git clone https://github.com/XVimProject/XVim2.git
```

次に `xcode-select`のパスが自分の Xcode を指しているのか確認します。

```bash
$ xcode-select -p
```

`/Applications/Xcode.app/Contents/Developer` のようなパスが出れば OK です。もし間違っていたら `xcode-select -s`で再設定しましょう。

この状態で先ほど clone したリポジトリに移動し、`make` を実行します。

```bash
$ cd XVim2
$ make
```

コマンドを実行すると色々標準入力に出力されるのですが、`** BUILD SUCCEEDED **` の文字が最後に出ていたら成功です。



これで XVim2 の設定は完了です。

Xcode を起動して以下、XVim2 をロードするかの確認メッセージが表示されるので`load bundle`を選びましょう。

![](https://storage.googleapis.com/zenn-user-upload/9jms63g5mxl4r02syh2w0h1sja6k =400x)

これで Xcode を起動すれば、もう Xcode で Vim キーバインドが使えるはずで🎉
（もう開発の遅れをキーバインドのせいには出来ません w）

![](https://storage.googleapis.com/zenn-user-upload/k3r240fm8dn27kb70q0uwfd8j15w)

有効なキーバインドの詳細は以下に掲載されています。

https://github.com/xvimproject/XVim2/blob/master/documents/featurelist.md

XVim2 独自のキーバインドなども色々あるようです（以下は一部）

 command   | note
-----------|-----
  :run          | Xcode の run コマンドを実行
  :make         | Xcode の build コマンドを実行

# `.xvimrc` の設定

なんと XVim2 は Vim と同じく`.vimrc`（`.xvimrc`)で設定やキーマップの変更が可能です。
XVim2 が起動時に`~/.xvimrc`をロードするので、ホーム直下に`.xvimrc`を作成しましょう。

```
$ touch ~/.vimrc
```

あとはお好みの設定を追記して、Xcode を再起動すれば設定が反映されるはずです。
他は試してないんですが、わりと色々対応しているようです。ぜひ試してみてください。

```vimc:.xvimrc
" ヤンクでシステムのクリップボードを使う
set clipboard+=unnamed
set clipboard=unnamed
```


# おわりに
以上「Xcode でも Vim キーバインドを使いたい！　XVim2 のススメ」でした。
これで IOS アプリも爆速開発ですね！
（何でも Vim キーバインドにしたくなる Vim の呪縛から逃れたい気持ちもある…）

# 参考

- [XVim2/signing_Xcode.md at master · xvimproject/XVim2](https://github.com/xvimproject/XVim2/blob/master/signing_Xcode.md)
- [xvim の設定は，基本 "いつもの" 感じで .xvimrc に - ばかもりだし](https://baqamore.hatenablog.com/entry/2015/02/08/091658)
