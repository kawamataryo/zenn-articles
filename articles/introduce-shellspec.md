---
title: "ShellSpecを使ってテスト駆動でシェルスクリプトを書く"
emoji: "🐌"
type: "tech"
topics: ["shell", "bash", "shellspec", "TDD"]
published: false
---

シェルスクリプトにテストを書くというイメージはなかったのですが、最近使った[ShellSpec](https://github.com/shellspec/shellspec)がとても使いやすくて、シェルスクリプトのテストのイメージが覆されたので紹介です。

# ShellSpecとは？

ShellSpecはシェルスクリプトでBDD（behavior driven development) のユニットテストを行うためのためのフレームワークです。

https://github.com/shellspec/shellspec

RSpecのように流れるようなテストを記述できます。豊富な[Mather](https://github.com/shellspec/shellspec/blob/master/docs/references.md#matchers) も提供されています。

```bash
Describe 'hello.sh'
  Include hello.sh
  It 'says hello'
    When call hello ShellSpec
    The output should equal 'Hello ShellSpec!'
  End
End
```

# TDDライクにシェルスクリプトを書いてみる

ShellSpecの便利さを知ってもらうため、TDDライクに簡単なシェルスクリプトを書いてみます。

以下今回書くシェルスクリプトの機能要件です。

- 2つの自然数を位置引数で受け取る
- 受け取った数値を足し合わせた結果を返却する
- 数値以外を受け取った場合や負の数、小数を受け取った場合はエラーが発生する

要するに自然数だけ受け取るsumスクリプトを書きます。

## ShellSpecの環境構築

最初にShellSpecの環境構築です。
shellspecコマンドを使えるように開発環境にShellSpecをインストールします。

```bash
$ wget -O- https://git.io/shellspec | sh
```

次に、プロジェクトを作成します。

```
$ mkdir sandbox-shellspec
```

ShellSpecの初期化を行います。
ShellSpecの設定ファイルである`.shellspec`とヘルパー関数の`spec/spec_helper.sh`が作成されます。

```bash
$ shellspec --init
```

## 最初のテスト

シェルスクリプトを作成して、最初のテストを書きます。

```
$ touch sum.sh
$ chmod +x ./sum.sh
```

```bash:sum.sh
#!/bin/bash
#
# =============================================================
# sum
#
# usage: sum.sh [num1] [num2]
# =============================================================
#

set -euC


function main() {
  echo ''
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main
fi
```

そしてテストはこちらです。
1と2を渡すと3を出力することと、正常終了していること（exit codeが0であること）を検証しています。

```bash
$ touch spec/sum_spec.sh
```
```bash:sum_spec.sh
Describe 'sum.sh'
  It 'sum up'
    When call main 1 2

    The output ./sum.sh equal 3
    The status success
  End
End
```

テストを実行します。

```bash
$ shellspec
# Running: /bin/sh [bash 3.2.57(1)-release]
# F
#
# Examples:
#   1) sum.sh sum up
#      When call ./sum.sh 1 2
#
#     1.1) The output should equal 3
#
#            expected: 3
#                 got: ""
#
#          # spec/sum_spec.sh:5
#
# Finished in 0.17 seconds (user 0.15 seconds, sys 0.02 seconds)
# 1 example, 1 failure
#
#
# Failure examples / Errors: (Listed here affect your suite's status)
#
# shellspec spec/sum_spec.sh:3 # 1) sum.sh sum up FAILED
```

もちろん失敗します（RED🔴）。

## 最初のテストを通す

TDDの慣例にしたがって、まずテストを通す事を優先に。
sum.shにてspecの求める`3`を固定値で返すようにします。

```bash:sum.sh
function main() {
  echo 3
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main
fi
```

これでテストが通りました（GREEN🟢）。

```bash
$ shellspec

# Running: /bin/sh [bash 3.2.57(1)-release]
# .
#
# Finished in 0.17 seconds (user 0.15 seconds, sys 0.02 seconds)
# 1 example, 0 failures
```

## リファクタリング

このままでは他の数値を渡した場合に、要件を満たせないのでリファクタリングします。

まずテストのリファクタリングです。
他の数値パターンでもテストできるように、shellspecの[Parameters](https://github.com/shellspec/shellspec#parameters---parameterized-example)を使って、パラメータテスト化します。

```bash:sum_spec.sh
Describe 'sum.sh'
  Describe
    Parameters
      1 2 3
      120 5 125
      23 55 78
      40 1234 1274
      0 0 0
    End

    Example "sum $1 and $2"
      When call ./sum.sh "$1" "$2"
      The output should eq "$3"
      The status should be success
    End
  End
End
```

この状態で実行します。
5個中4個に失敗していることがわかります（RED🔴）。

```bash
$ shellspec
# Running: /bin/sh [bash 3.2.57(1)-release]
# .FFFF
#
# Examples:
#   1) sum.sh sum 120 and 5
#      When call ./sum.sh 120 5
#
#      1.1) The output should eq 125
#
#             expected: 125
#                  got: 3
#
#           # spec/sum_spec.sh:14
#
# ...省略
#
# Finished in 0.21 seconds (user 0.18 seconds, sys 0.04 seconds)
# 5 examples, 4 failures
#
#
# Failure examples / Errors: (Listed here affect your suite's status)
#
# shellspec spec/sum_spec.sh:12 # 1) sum.sh sum 120 and 5 FAILED
# shellspec spec/sum_spec.sh:12 # 2) sum.sh sum 23 and 55 FAILED
# shellspec spec/sum_spec.sh:12 # 3) sum.sh sum 40 and 1234 FAILED
# shellspec spec/sum_spec.sh:12 # 4) sum.sh sum 0 and 0 FAILED
```

これを通すため、sum.shを以下のように修正します。

```bash
function main() {
    local num1="$1"
    local num2="$2"

    echo $((num1 + num2))
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$1" "$2"
fi
```

再度shellspecを実行すると無事通りました（GREEN🟢）。

```bash
$ shellspec

# Running: /bin/sh [bash 3.2.57(1)-release]
# .....
#
# Finished in 0.21 seconds (user 0.18 seconds, sys 0.03 seconds)
# 5 examples, 0 failures
```

## 異常系のテストを追加

要件には、`数値以外を受け取った場合や負の数、小数を受け取った場合はエラーが発生する` というものがありました。そちらのテストも追加してみます。
標準エラー出力を表す`error`の文字列にErrorが含まれることと、終了ステータスがerrorであることを（0以外であること）を検証しています。



```bash
Describe 'sum.sh'
  #...

  It "raise error 1 and a"
    When call ./sum.sh 1 a
    The error should include "Error:"
    The status should be failure
  End
End
```

実行してみます。
今は引数を何も検証していないので失敗します（RED🔴）。

```
$ shellspec                                                                                                                                                                                         ✘ 101
# Running: /bin/sh [bash 3.2.57(1)-release]
# .....F
#
# Examples:
#   1) sum.sh raise error 1 and a
#      When call ./sum.sh 1 a
#
#      1.1) The error should include Error:
#
#             expected "./sum.sh: line 17: a: unbound variable" to include "Error:"
#
#            # spec/sum_spec.sh:20
```

## とりあえず通す
とりあえず通すため、引数の検証処理をsum.shに実装します。
`validate_args` という関数を新たに作り、そこに検証処理を追加しています。

```bash
  # ...

  function validate_args() {
    arg_1="$1"
    arg_2="$2"

    if [[ "a" == "$arg_2" ]]; then
        echo "Error: invalid argument a" >&2
        exit 1
    fi
}

function main() {
  # ...
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    validate_args "$@"
    main "$1" "$2"
fi
```

これで実行すると、テストがとおります（GREEN🟢）。

```bash
$ shellspec
# Running: /bin/sh [bash 3.2.57(1)-release]
# ......
#
# Finished in 0.21 seconds (user 0.17 seconds, sys 0.04 seconds)
# 6 examples, 0 failures
```

## リファクタリング
今は第二引数が`a`の時にしか対応していないので、さらにテストケースを追加して検証してみます。
正常系と同じくパラメーターテストのExampleを使った形に書き直します。

```bash

```

テストを実行すると失敗します（RED🔴）。

```bash
$ shellspec
~/s/tdd-shell ❯❯❯ shellspec                                                                                                                                                                                         ✘ 101
#  Running: /bin/sh [bash 3.2.57(1)-release]
#  ......FFFFF
# 
# Examples:
#   1) sum.sh raise error b and c
#      When call ./sum.sh b c
# 
#      1.1) The error should include Error:
# 
#             expected "./sum.sh: line 27: b: unbound variable" to include "Error:"
# 
#           # spec/sum_spec.sh:30
# 
# ...
# 
# Finished in 0.27 seconds (user 0.22 seconds, sys 0.05 seconds)
# 11 examples, 5 failures
# 
# 
# Failure examples / Errors: (Listed here affect your suite's status)
# 
# shellspec spec/sum_spec.sh:28 # 1) sum.sh raise error b and c FAILED
# shellspec spec/sum_spec.sh:28 # 2) sum.sh raise error -1 and 1 FAILED
# shellspec spec/sum_spec.sh:28 # 3) sum.sh raise error 1.1 and 1.5 FAILED
# shellspec spec/sum_spec.sh:28 # 4) sum.sh raise error 1 and  FAILED
# shellspec spec/sum_spec.sh:28 # 5) sum.sh raise error 1 and 2 FAILED
```

このテストケースも通るように`validates_args`を以下のように修正しました。
`is_number`関数で自然数かどうかの判定を行い、validate_argsでその結果を元にエラーを出力しています。
また、`$#`の値を見ることで引数の数も検証しています。

```bash
# ...
function is_number() {
    local number="$1"
    if [[ $number =~ ^[0-9]+$ ]]; then
        return 0
    else
        return 1
    fi
}

function validate_args() {
    if [[ $# -ne 2 ]]; then
        echo "Error: Incorrect number of arguments. Please pass two natural numbers as arguments." 1>&2
        usage
        exit 128
    fi

    if ! is_number "$1" || ! is_number "$2"; then
        echo "Error: Incorrect type of arguments. Please pass two natural numbers as arguments." 1>&2
        usage
        exit 128
    fi
}

function main() {
  # ...
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    validate_args "$@"
    main "$1" "$2"
fi
```

最後にもう一度テストを実行します。
今度は通るはずです（GREEN🟢）

```bash
$ shellspec
# Running: /bin/sh [bash 3.2.57(1)-release]
# ..........
#
# Finished in 0.25 seconds (user 0.21 seconds, sys 0.05 seconds)
# 10 examples, 0 failures
```

これで要件をすべて満たせました🎉

# おわりに

ShellSpecには、本記事で紹介した以外にも、コマンドラインスクリプトのMock、テストカバレッジの出力など豊富な機能があります。
シェルスクリプトのテストを書く際には是非試してみてください。

https://shellspec.info/