---
title: "Raspberry Piへの公開鍵認証でのSSH接続の設定 & VSCode Remote Developmentの設定方法"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["RaspberryPi", "ラズベリーパイ", "ssh"]
published: false
---

ラズベリーパイの公開鍵認証を設定するまでのメモ

# 1. キーファイルの作成

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

次に、パスフレーズを聞かれます。本当は設定したほうが良いのですが、今回は省きます。

```bash
Enter passphrase (empty for no passphrase):
```

そのまま Enter を押せば OK です。
これで`.ssh/raspberrypi`配下に秘密鍵の`id_rsa`と、公開鍵の`id_rsa.pub`が作成されます。

次にパーミッションの設定をします。

```bash
$ chmod 600 ~/.ssh/raspberrypi/id_rsa
```


最後に、今作成した公開鍵をラズベリーパイに送ります。

```bash
$ scp ~/.ssh/raspberrypi/id_rsa.pub pi@raspberrypi:~
```

パスワードを聞かれるので、`raspberry`（初期設定から変えていない場合）を入力してください。



# 2. ラズベリーパイ側のsshの設定
ssh（Password 認証）でラズベリーパイに接続します。

```bash
$ ssh pi@raspberrypi
```
次に ssh の鍵を管理する`.ssh`ディレクトリを作成します。

```bash
$ sudo mkdir ~/.ssh
```

そして、先ほど送った`id_rsa.pub`を`authorized_keys`と名前を変更しつつ`.ssh`に移動します。

```bash
$ mv ~/id_rsa.pub .ssh/authorized_keys
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
これでラズパイ側の作業は完了です。

```bash
$ sudo /etc/init.d/ssh restart
```

# ssh configの修正

最後にローカル側での作業です。
この段階では、ラズパイの秘密鍵を、`.ssh`直下に置いていないため毎回`-i`オプションで keyfile を指定する必要があります。
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

# VSCode Remote Desktopでの接続