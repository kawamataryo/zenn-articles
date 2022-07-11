---
title: "ファイル群の変更を検知して、特定のコマンドを実行する bash スクリプトを書いてみた"
emoji: "👀"
type: "tech"
topics: ["bash", "shell", "linux"]
published: false
---

社会人大学院の課題で独自の bash スクリプトを提出するものがあり、その時書いたたものがわりと便利だと思ったので公開します。

# どんな bash スクリプト？

特定のファイル群の変更を検知して、特定のコマンドを実行する bash スクリプトです。
ビルドやテストフレームワークにある`watch`オプションを、任意のコマンド、任意のファイル群で実行できるイメージです。

`-c`で実行するコマンド、`-f`で監視対象のファイル群を指定します。

```bash
$ ./watch-files.sh -c 'echo "update"' -f './src/**/*.py'

watched ./src/**/*.py ...
If want to terminate, press to Ctrl-D

update # -cで指定した条件に合致するファイルを変更した際に-cで指定したコマンドが実行される
update # Ctrl-Dで停止するまで、変更ごとに指定のコマンドが再実行される
```

標準でwatchオプションの存在しないテストフレームワークや、ビルドツールで利用することを想定しています（Pythonのunittestとか）。

## 開発のきっかけ

業務ではDjango を使っていて、テストフレームワークは unittest です。
TDD 的な頻繁にテストを書く開発するタイルなのですが、unittest では jest の watch オプションのように、ファイルの変更を検知してテストを再実行する機能が標準で付いてらず、ファイル変更後に毎回手動でテストを再実行する必要がありました。それが面倒で、変更を検知してコマンドを実行してくれる汎用的なスクリプトがあったらなーと思っていたので開発を思い立ちました。

## 実装

実装はこちらです。

https://github.com/kawamataryo/watch-files/blob/main/watch-files.sh#L1-L145

MacとLinuxであれば、任意のディレクトリにスクリプトをダウンロードして`chmod +x ./watch-files.sh`で実行権限をつければ実行できます。

# 工夫ポイント

もろもろ工夫ポイントがあったので紹介します。

## 変更有無の判定処理

監視対象の変更有無は、1秒ごとに`stat`コマンドを実行し、監視対象のファイル群の更新日時の差分を取ることで判定しています。
ファイルの群の更新日時を取得する際に、毎回`find`で条件に合致するファイルを収集することで、既存ファイルの変更だけでなく、ファイルの新規追加・削除にも対応しています。

```bash
function get_updated_times_from_files() {
    local files=$1
    local changed_time=''

    # 監視対象のファイル群を収集
    find_result=$(find . -wholename "$files")
    # ファイル群すべての更新日時を結合した文字列を取得
    for file in $find_result; do
        changed_time+="$(safe_stat "$file"),"
    done

    echo "$changed_time"
}

# ...

function main() {
    # ...

    # 基本の更新日時
    base_changed_time=$(get_updated_times_from_files "$files")

    # ...

    while :
    do
        # 一秒待機
        sleep $WAIT_TIME

        # 今の更新日時
        changed_time=$(get_updated_times_from_files "$files")

        # もし基本の更新日時と差分があったら
        if [[ "$base_changed_time" != "$changed_time" ]]; then
            # ...

            # コマンドを実行
            eval "$command &"

            # ...
        fi
    done
}
```

## コマンドの重複起動の防止

単純に変更を検知してコマンドを実行するだけだど、複数のファイルを短時間で一気に変更した場合や、実行するコマンドの実行時間が長い場合、前回のコマンドの終了を待てず、同時にコマンドが実行される可能性があります。

それを防ぐために、コマンド実行のプロセスIDを記録して、前のプロセスが残っている場合はそれをkillしてからコマンドを実行するようにしました。

```bash
function kill_process_if_exists() {
    local pid=$1

    if [[ -n "$pid" ]] && ps -p "$pid" > /dev/null
    then
        kill "$pid"
    fi
}

# ...

function main() {
    # ...
    while :
    do
        # ...
        if [[ base_changed_time -ne changed_time ]]; then
            kill_process_if_exists "$pid"

            # execute command
            eval "$command &"

            pid=$!
            base_changed_time="$changed_time"
        fi
    done
}
```

## GNU/Linux, BSDの両方に対応
ファイル更新日の取得に使っている`stat`コマンドはGNU/LinuxとBSDでオプション形式が異なります。素直に使うとLinuxかMacどちからで動かないコマンドとなってしまいます。それは避けたかったので、以下のように、実行コマンドを切り替える関数を作りオプションの差分を吸収するようにしました。

`--help`オプションが、GNU/Linuxだけにあることに着目して、exitコードでの分岐で判定しています。

```bash
function safe_stat() {
    local file=$1
    if stat --help >/dev/null 2>&1; then
        echo "$(stat -c %Y "$file")"
    else
        echo "$(stat -f %m "$file")"
    fi
}
```

# 参考

bash スクリプトの雛形として、とても参考にさせて頂きました。感謝。
https://www.m3tech.blog/entry/2018/08/21/bash-scripting

# おわりに

bash スクリプト苦手意識が強かったのですが、なんとか自分がほしかったスクリプトを作れたので良
かったです。
あと、なにげにはじめて1から書いた bash スクリプトなので、おかしいところ等あれば気軽にコメントもらえると嬉しいです 🙏
