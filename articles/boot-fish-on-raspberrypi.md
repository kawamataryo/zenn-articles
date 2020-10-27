---
title: "Raspberry Piでfishシェルを設定する方法"
emoji: "🐠"
type: "tech"
topics: ["電子工作", "RaspberryPi", "fish", "shell"]
published: true
---

Raspberry Pi で fish シェルを設定する方法を備忘録としてまとめます。

# fish とは？

https://fishshell.com/

強力な入力補完、親切なシンタックスハイライトを持つユーザーフレンドリーなシェルです。
自分は zsh よりも、fish 派です。

# fish のインストール

まず、自分の Raspbian のバージョンを確認します。

```bash
$ lsb_release -a
# No LSB modules are available.
# Distributor ID:	Raspbian
# Description:	Raspbian GNU/Linux 10 (buster)
# Release:	10
# Codename:	buster
```

これはバージョン 10 ですね。

次に以下コマンドで fish をインストールします。

```bash
# aptの利用可能パッケージにfishを追加
$ echo 'deb http://download.opensuse.org/repositories/shells:/fish:/release:/3/Debian_10/ /' | sudo tee /etc/apt/sources.list.d/shells:fish:release:3.list

# gpgでソフトウェアの検証を実行（あまりわかってない）
$ curl -fsSL https://download.opensuse.org/repositories/shells:fish:release:3/Debian_10/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/shells:fish:release:3.gpg > /dev/null

# パッケージのインデックスファイルの更新（ここでfishが認識される）
$ sudo apt update

# fishのインストール
$ sudo apt install fish
```

:::message
もし、自分の Rasbian のバージョンが 9 の場合は、上記コマンドの`Debian_10`の部分を全て`Debian_9`に変更してください。
:::

これでインストールの完了です。
以下で fish シェルが起動するはずです。

```
$ fish
```

やったぜ！　Hello 🐠!


# fish をデフォルトのシェルに設定

標準で起動するシェルを`bash`から`fish`に変更します。
起動シェルの変更は`chsh`コマンドで行います。

まず、fish シェルがシェルとして認識されているか確認します。

```bash
$ cat /etc/shells
# /etc/shells: valid login shells
# /bin/sh
# /bin/bash
# /bin/rbash
# /bin/dash
# /usr/bin/fish
```

`/usr/bin/fish`がありますね。
あとはこれを`chsh`コマンドで設定すれば完了です。

```bash
$ chsh
# パスワード:
# pi のログインシェルを変更中
# 新しい値を入力してください。標準設定値を使うならリターンを押してください
# ログインシェル [/bin/bash]: /usr/bin/fish
```

パスワードを入力するとログインシェルの設定を聞かれるのでここで、先ほど確認した`/usr/bin/fish`を設定します。
その後、ssh を一度ログアウトして、再度ログインすると fish シェルが起動するはずです。

# 終わりに

以上「Raspberry Pi で fish シェルを設定する方法」でした。
fish 追加したり vim 追加したり、足回りを固めてばかりで一向に電子工作が進みません w

# 参考

- https://software.opensuse.org/download.html?project=shells%3Afish%3Arelease%3A3&package=fish