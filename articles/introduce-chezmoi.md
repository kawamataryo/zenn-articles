---
title: "chezmoiでdotfilesを手軽に柔軟にセキュアに管理する"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dotfiles", "bash", "fish", "zsh"]
published: false
---

あまり日本語情報がなかった dotfiles マネージャの chezmoi についてまとめました。
個人的にかなり便利だと思います！

# chezmoiとは？

chezmoi は、`.vimrc` や、`.zshrc` などの dotfiles の管理を効率的に実現するためのツールです。

https://github.com/twpayne/chezmoi

1. Symbolic link 不要でコマンド 1 つで環境を再現出来る
1. template 構文を使い、環境ごとの差分を 1 ファイルで管理できる
1. 1password などのパスワードマネージャとの併用でセキュアにファイル管理ができる

という特徴があります。
特に 2 と 3 の通常の Symbolic link での dotfiles 管理だと shell 芸を頑張らないと出来ない部分ですが、chezmoi なら手軽に実現できます。


# 基本操作

### イントール

Homebrew でインストール出来ます。

```bash
$ brew install chezmoi
```
### chezmoiプロジェクトの初期化
chezmoi で dotfile を管理するために chezmoi のプロジェクトディレクトリを作成します。
以下コマンドを実効すると`~/.local/share/chezmoi`にディレクトリが作られます。ここが dotfiles の管理元となります。
自動的に`git init`も行われます。

```bash
$ chezmoi init
```

### dotfilesの追加
`add`コマンドで管理対象としたい dotfiles を追加します。
以下コマンドを実効すると`~/.local/share/chezmoi`配下に`dot_vimrc`ファイルが作成されます。

```bash
$ chezmoi add .vimrc
```

### dotfilesの編集
管理対象としたファイルは`edit`コマンドで編集できます。
edit コマンドを実行すると`~/.local/share/chezmoi`ディレクトリ内のファイルが編集できます。
この状態では、元ファイル（以下例だと`~/vimrc`）には変更が反映されません。
変更を反映するには次で説明する`apply`が必要です。

```bash
$ chezmoi edit .vimrc
```

### dotfiles変更の適応
dotfiles の変更の適応は、`apply`コマンドで行います。

```bash
$ chezmoi apply
```

これで`chezmoi edit ~/.vimrc`で加えた変更（`~/.local/share/chezmoi/dot_vimrc`の変更）が`~/.vimrc`に適応されます。

### 別PCへのdotfilesの移行
別 PC への dotfiles の移行は GitHub 経由で行います
そのためにまず chezmoi プロジェクトでの変更を commit しリモートリポジトリへ push します。

```bash
# chezmoiのプロジェクトへの移動
$ chezmoi cd

# 変更のcommit & push
$ git add .
$ git remote add origin git@github.com:kawamataryo/dotfiles.git
$ git push
```

そして別 PC にて chezmoi のプロジェクト初期化の際に GitHub のリポジトリを指定します

```bash
# 別PCでの操作
$ chezmoi init git@github.com:kawamataryo/dotfiles.git
```

これで`~/.local/share/chezmoi`配下に GitHub リポジトリ上のファイルが作成されます。
この状態で`apply`コマンドを実行すれば適切なパスに dotfiles が作成されます。

```
$ chezmoi apply
```

# template機能の利用
dotfiles を管理していると「この記述は個人環境だと必要だけど、職場の環境だと不要だなー」ということがあると思います。そんな要望も chezmoi の template 機能を使えば手軽に実現出来ます。

template 機能は、GO の[template](https://pkg.go.dev/text/template)記法を使って dotfiles 上に任意の条件分岐や変数の展開を組み込むものです。

template を使う場合は、chezmoi に管理に追加する際に`--template`オプションを付与します。

```
$ chezmoi add --template ~/.zshrc
```

すでに chezmoi 管理の dotfiles に template 機能を追加する場合は`chattr +template`を使います。

```
$ chezmoi chattr +template ~/.zshrc
```

これで chezmoi のディレクトリ配下に`dot_zshrc.tmpl`という`tmpl`拡張子がついたファイルが作成されます。このファイルは template 構文に対応しているので、任意の条件分岐や変数展開を記述できます。

### PCのhomenameでの出力制御

例えば会社 PC の場合は、rbenv を利用するけど個人 PC では必要なかったとします。
その場合は以下のように`.zshrc`ファイル内で PC の hostname を使って分岐できます。

```bash
# ...
{{ if (eq .chezmoi.hostname "work-pc") }}
# rbenv
export PATH=~/.rbenv/bin:$PATH
eval "$(rbenv init -)"
{{ end }}
# ...
```

この状態で`chezmoi apply`すると、会社 PC の場合のみ rbenv の記述が`.zshrc`に追加されます。

なお`.chezmoi`変数から参照できる値は以下コマンドで確認できます。
linux の場合のみなど OS での分岐も可能です。

```bash
$ chezmoi data
{
  "chezmoi": {
    "arch": "amd64",
    "fullHostname": "xxxx",
    "group": "staff",
    "homedir": "xxx",
    "hostname": "xxx",
    "os": "darwin",
    "sourceDir": "xxx",
    "username": "xxx"
  }
}
```

## 1passwordを利用したセキュアな管理

ssh の秘密鍵などセキュアなファイルについても chezmoi と[1password-cli](https://support.1password.com/command-line-getting-started/)などの外部ツールを組み合わせることで管理できます。

以下 chezmoi と 1password-cli で ssh の秘密鍵を管理する例です。
1password-cli の環境構築は終了している前提で進めます。

### 1. 管理対象のファイルをtemplateとしてchezmoiに追加
ssh の秘密鍵を template としては chezmoi の管理対象に追加します。
この時点ではまだ chezmoi 上の`.ssh/id_rsa.tmpl`に秘密鍵の生データが存在します。

```bash
$ chezmoi add --template .ssh/id_rsa
```

### 2. 1password-cliでファイルを保存する
秘密鍵を 1password の管理対象に追加します。
その際に戻り値として得られる uuid をメモします。

```bash
# 1password-cliにログイン
$ eval $(op sign my)

### 秘密鍵を1passwordの管理対象に追加
$ op create document ~/.ssh/id_rsa --tags chezmoi --title .ssh/id_rsa
{"uuid":"ti2adie9Aixaidae4dahpoh5io","createdAt":"2021-02-01T17:53:49.596484+02:00","updatedAt":"2021-02-01T17:53:49.596484+02:00","vaultUuid":"eith2iequievuthae9Eedaiboh"} # 出力はダミーです
```

### 3. chezmoi管理のtemplateファイルを書き換える
先程取得した uuid を使って、chezmoi 管理の`.ssh/id_rsa.tmpl`の内容を全て書き換えます。

```bash
$ chezmoi edit .ssh/id_rsa
```

```bash:.ssh/id_rsa
{{- onepasswordDocument "ti2adie9Aixaidae4dahpoh5io" -}}
```

これで完了です。
この状態で Git 管理して、新しい PC で`chezmoi update` & `chezmoi apply`すれば 1password からセキュアに対象ファイルを復元できます。
（その際は新しい PC で`eval $(op sign my)`を行い 1password-cli にログインしておく必要があります）

# 終わりに

以上「chezmoi で dotfiles を手軽に柔軟にセキュアに管理する」でした。
chezmoi にはここで紹介した機能以外にも便利な機能があります。是非ドキュメントを読んでみてください。

https://github.com/twpayne/chezmoi

実際に chezmoi を使っている私の dotfiles はこちらです。今後も chezmoi を使って dotfiles を育てていきたいです。

https://github.com/kawamataryo/dotfiles



# 参考
- [Managing dotfiles and secret with chezmoi • Brice Dutheil](https://blog.arkey.fr/2020/04/01/manage_dotfiles_with_chezmoi/)