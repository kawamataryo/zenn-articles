---
title: "ラズベリーパイにSSHで公開鍵認証を設定する"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RaspberryPi", "ラズベリーパイ", "ssh"]
published: false
---

ラズベリーパイの公開鍵認証を設定するまでのメモ

# 1. (ローカル）キーファイルの作成

まずキーファイルを置くディレクトリを作ります。（そのまま.ssh直下に置いておいても良いのですが、整理しておいた方が良いと思うので）

```
mkdir ~/.ssh/raspberrypi
```

次にキーファイルを`ssh-keygen`コマンドで作成します。

```
$ ssh-keygen -t rsa
```

ここで、鍵の作成場所を聞かれるので、作成したい場所を指定します。

```
Generating public/private rsa key pair. 
Enter file in which to save the key (/Users/kawamataryou/.ssh/id_rsa):
```

今回は `/Users/kawamataryou/.ssh/raspberrypi/id_rsa`と設定しました。

次に、パスフレーズを聞かれます。本当は設定した方が良いのですが、今回は省きます。

```
Enter passphrase (empty for no passphrase):
```

そのままEnterを押せばOKです。
これで`.ssh/raspberrypi`配下に秘密鍵の`id_rsa`と、公開鍵の`id_rsa.pub`が作成されます。

次にパーミッションの設定をします。

```
$ chmod 600 ~/.ssh/raspberrypi/id_rsa
```


最後に、今作成した公開鍵をラズベリーパイに送ります。

```
scp ~/.ssh/raspberrypi/id_rsa.pub pi@raspberrypi:~
```

パスワードを聞かれるので、`raspberry`（初期設定から変えていない場合）を入力してください。



# 2. sshの設定

```
$ ssh pi@raspberrypi
```

```
$ sudo mkdir ~/.ssh
```

```
$ mv ~/id_rsa.pub .ssh/authorized_keys
```

```
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 700 ~/.ssh

```

600は`-rw-------`で管理者のみ読み取り書き込みが可能、700は`xrw-------`で管理者のみ実行読み取り書き込み可能です。
これをしないとssh接続の時に怒られるので必ず設定しましょう。

次に`ssh_conf`を修正して、ssh接続を有効にしましょう。

```
$ sudo vi /etc/ssh/sshd_config
```

以下の行のコメントアウト削除して公開鍵認証を有効化してください。

```
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
```

ここで、Portの変更や、Password認証の

最後に変更内容を反映させるためsshdを再起動します。

```
$ sudo /etc/init.d/ssh restart
```

続いてローカルでの作業です。

# VSCode Remote Desktopでの接続