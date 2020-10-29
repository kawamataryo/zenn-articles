---
title: "Raspberry Piへの公開鍵認証でのSSH接続 & VSCode Remote Developmentの設定方法"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RaspberryPi", "linux", "ssh"]
published: false
---

Raspberry Pi に SSH（公開鍵認証）で接続するまで & VSCode の Remote Development の設定を備忘録としてまとめます。

# 公開鍵認証とは？

公開鍵認証とは、パスワードの代わりに 公開鍵と秘密鍵のペアを用いる認証方法です。
詳細は以下記事をご覧ください。

[SSHの公開鍵認証における良くある誤解の話 - Qiita](https://qiita.com/angel_p_57/items/2e3f3f8661de32a0d432)

# キーファイルの作成

まずキーファイルを置くディレクトリを作ります（.ssh 直下に秘密鍵を置いておいても良いのですが、整理しておいたほうが後々楽なので）

```bash
mkdir ~/.ssh/raspberrypi
```

次にキーファイルを`ssh-keygen`コマンドで作成します。

```bash
$ ssh-keygen -t rsa
```

ここで、鍵の作成場所を聞かれるので、先ほど作成したディレクトリを指定します。

```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/kawamataryou/.ssh/id_rsa):
```

今回は `/Users/kawamataryou/.ssh/raspberrypi/id_rsa`と設定しました。

次に、パスフレーズを聞かれます。あったほうが安全なんですが、今回は省略します。
続けて Enter を押せば OK です。

```bash
Enter passphrase (empty for no passphrase):
```

これ`.ssh/raspberrypi`配下に秘密鍵の`id_rsa`と、公開鍵の`id_rsa.pub`が作成されます。

次にパーミッションの設定をします。

```bash
$ chmod 600 ~/.ssh/raspberrypi/id_rsa
```


最後に、今作成した公開鍵を Raspberry Pi に送ります。

```bash
$ scp ~/.ssh/raspberrypi/id_rsa.pub pi@raspberrypi:~
```

パスワードを聞かれるので、`raspberry`（初期設定から変えていない場合）を入力してください。

# Raspberry Pi側のsshの設定
ssh（Password 認証）で Raspberry Pi に接続します。

```bash
$ ssh pi@raspberrypi
```
次に ssh の鍵を管理する`.ssh`ディレクトリを作成します。

```bash
$ sudo mkdir ~/.ssh
```

そして、先ほど送った`id_rsa.pub`を`authorized_keys`と名前を変更しつつ`.ssh`に移動します。

```bash
$ mv ~/id_rsa.pub ~/.ssh/authorized_keys
```

次に`.ssh`, `authorized_keys`のパーミッションを変更します。

```bash
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 700 ~/.ssh
```

600 は`-rw-------`で管理者のみ読み取り書き込みが可能、700 は`xrw-------`で管理者のみ実行読み取り書き込み可能です。
これをしないと ssh 接続の時に怒られるので必ず設定しましょう。

次に`ssh_conf`を修正して、ssh 接続を有効にしましょう。

```bash
$ sudo vi /etc/ssh/sshd_config
```

以下の行のコメントアウト削除して公開鍵認証を有効化してください。

```
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
```

:::message
ここで、Port の変更や、Password 認証の OFF を行うとセキュリティ的に安心です。
Password 認証の OFF は、`PasswordAuthentication no`、Port の変更は`Port （任意の数値）`とすれば OK です。
:::

最後に変更内容を反映させるため sshd を再起動します。
これで Raspberry Pi 側の作業は完了です。

```bash
$ sudo /etc/init.d/ssh restart
```

# ssh configの修正

最後にローカル側での作業です。
この段階では、Raspberry Pi の秘密鍵を、`.ssh`直下に置いていないため毎回`-i`オプションで keyfile を指定する必要があります。
それは面倒なので config に設定を追記します。

`~/.ssh/config`を開き以下を追記します（config がなければ作成してください）。

```~/.ssh/config
Host raspi
  HostName raspberrypi
  User pi
  IdentifyFile ~/.ssh/raspberrypi/id_rsa
```

これで以下コマンドで公開鍵認証でのssh接続ができるようになっているはずです。

```bash
$ ssh raspi
```

:::message
今回は使わなかったのですが、`.ssh/config` を分割する方法もあります。
そちらを使ってさらに整理するのも良さそうです。詳細は以下をどうぞ。
https://qiita.com/masa0x80/items/ecb692ad93f7d06a07b0
:::

# VSCode Remote Developmentでの接続

ssh で接続して、Vim で開発でも良いのですがどうせならより使い慣れたエディタで開発したいですよね。その要望を叶えるのが VSCode の Remote Development です。
まるでローカルに環境があるかのように、Raspberry Pi の内部を VSCode で開けます。

まず、VSCode に Remoete Development のエクステンションをインストールします。

https://github.com/Microsoft/vscode-remote-release

![](https://storage.googleapis.com/zenn-user-upload/9s5nfpd8ld59pk2iis7t5wnrp9p3)

インストールするとサイドバーに Remote Development のアイコンが追加されます。
アイコンをクリックするとタブが開くので、上部の REMOTE EXPLORER のセレクトボックスを`SSH Targets`に設定します。
こうすると`.ssh/config`で設定しているホスト一覧が出るのであとは、先ほど設定した`raspi`をクリックするだけです。

![](https://storage.googleapis.com/zenn-user-upload/z0wzfni5g8kueftok2f1kkg0rt5l)

クリックすると、別ウィンドウで VSCode が開き、Raspberry Pi に SSH 接続して環境を構築しはじめます。
初回は完了までしばらくかかります。

![](https://storage.googleapis.com/zenn-user-upload/7p0ktf01ccrsdsjqs10608ytmzwh)

:::message
もしここで Timeout Error などで失敗する場合は、`remote.SSH.useLocalServer": false`を VSCode の setting.json に追加しましょう。
そうすると接続時に、Remote Host の OS を聞かれるので Linux を選択すれば OK です。
https://github.com/microsoft/vscode-remote-release/issues/1721
:::

完了後、EXPLORER のタブで Open Folder をクリックしてパス を指定して OK を押せば完了です。

![](https://storage.googleapis.com/zenn-user-upload/wr8x134dwst0jdzqi1nyj2ylsgpq)

まるでローカルにファイルがあるかのようにディレクトリ・ファイルが閲覧出来て VSCode 上でそのまま編集できます。最高便利！

![](https://storage.googleapis.com/zenn-user-upload/hspqflcbg3naxcpv4gr30q850z54)

# おわりに

以上「Raspberry Pi への公開鍵認証での SSH 接続 & VSCode Remote Development の設定方法」でした。
Remote Development がとても便利ですね。VSCode すごい。