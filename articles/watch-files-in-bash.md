---
title: "特定のファイル群の変更を検知して、特定のコマンドを実行するbashスクリプトを書いてみた"
emoji: "👁️‍🗨️"
type: "tech"
topics: ["bash"]
published: false
---

社会人大学院の課題で bash スクリプトを書く機会があり、その時書いたたものがわりと便利だと思ったので公開します。

# どんな bash スクリプト？

特定のファイル群の変更を検知して、特定のコマンドを実行する bash スクリプトです。
ビルドやテストフレームワークにある`watch`オプションを、任意のコマンド、任意のファイル群で実行できるイメージです。

`-c`で実行するコマンド、`-f`で監視対象のファイル群を指定します。

```bash
$ ./watch-files.sh -c 'echo "update"' -f './src/**/*.py'

watched ./src/**/*.py ...
If want to terminate, press to Ctrl-D

update # ./src/**/*.pyの条件に合致するファイルを変更した際
```

## 開発のきっかけ

業務では、Django を使っていて、テストフレームワークは unittest です。TDD 的な頻繁にテストを書く開発するタイルなのですが、unittest では jest の watch オプションのように、ファイルの変更を検知してテストを再実行する機能が標準で付いてらず、ファイル変更後に毎回手動でテストを再実行する必要がありました。それが面倒で、変更を検知してコマンドを実行してくれる汎用的なスクリプトがあったらなーと思っていたので開発を思い立ちました。

## 実装

実装はこちらです。

https://github.com/kawamataryo/watch-files/blob/main/watch-files.sh#L1-L145

# 工夫ポイント

もろもろ工夫ポイントがあったので紹介します。

## 変更有無の判定

監視対象の変更有無は、1秒ごとに`stat`コマンドでの更新日時の差分を取ることで判定しています。
ファイルの群の更新日時を取得する際に、毎回`find`で条件に合致するファイルを収集することで、既存ファイルの変更だけでなく、ファイルの新規追加・削除にも対応しています。

```bash
function get_updated_times_from_files() {
    local files=$1
    local changed_time=''

    # ファイル群すべての更新日時を結合した文字列を取得
    find_result=$(find . -wholename "$files")
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

## GNU/Linux, BSDの差分を吸収して、LinuxとMacの両方でつかえるものに
ファイル更新日の取得に使っている`stat`コマンドはGNU/LinuxとBSDでオプション形式が異なるのですが、両方で使えるコマンドを作りたかったので以下のように、実行コマンドを切り替える関数を作り差分を吸収するようにしました。

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
あと、なにげにはじめて書いた bash スクリプトなので、おかしいところ等あれば気軽にコメントもらえると嬉しいです 🙏
