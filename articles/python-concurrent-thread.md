---
title: "Pythonの並行処理を理解したい [マルチスレッド編]"
emoji: "🧵"
type: "tech"
topics: ["python"]
published: false
---

Python の並行処理についてまとめてみました。

# 並行処理とは？

`並行処理（Concurrent）`とは、一定の時間内に複数の処理を同時に進行することを指す用語です。
例えば、コーディング中にアプリのビルドを実行しながら Twitter を見るのもコーディングと Twitter の並行処理です。

# 並行処理と並列処理の違い

並行処理と比較して語られる処理に、`並列処理（Parallel）`があります。
2 つの違いを簡単にまとめます。

## 並行処理（Concurrent）
並行処理は瞬間を切り取ったときは 1 つの処理を行っているのですが、ある一定の時間で区切ると処理を切り替えながら複数の処理をこなしているものを指します。
Python でいうと今から説明するマルチスレッド（`ThreadPoolExecutor`）やイベントループ（`asyncio`）がこれに当たります。
並行処理での高速化は複数 API のアクセスや、ファイル書き込みなど I/O の待ち時間が発生するものに向いています。

![](https://i.gyazo.com/7753f2f79e30028f79ddeb7820bbd291.png)

## 並列処理（Parallel）
並列処理は瞬間を切り取ったときにも複数の処理を同時並行に実行しているものを指します。
Python でいうと複数コアでのマルチプロセス（`ProcessPoolExecutor`）がこれに当たります。
並列処理での高速化は暗号の復号化など CPU パワーを使うものに向いています。

![](https://i.gyazo.com/246dafbd479595051e057f6699387e58.png)

:::message
並列処理は並行処理に抱合される概念です。並列処理は平行処理だと言えますが、並行処理だからといって並列処理とは限りません。
参考： [Python実践入門 ──言語の力を引き出し、開発効率を高める](https://gihyo.jp/book/2020/978-4-297-11111-3)
:::


# Pythonのマルチスレッド処理（ThreadPoolExecutor）
マルチスレッドはその名の通り、プロセスから複数のスレッドを作り処理を並行して行うものです。
Python のマルチスレッドは concurrent.futures モジュールの `ThreadPoolExecutor` クラスを使います。

コンテキストマネージャで取得出来る ThreadPoolExecutor のインスタンスに対して `submit()`を呼ぶことでマルチスレッドで行いたい関数を登録します。

```python
with ThreadPoolExecutor() as executor
    feature = executor.submit(<マルチスレッドで実行したい関数>)
```

submit の戻値に対して`result()`を呼ぶことで関数の処理結果を取得出来ます。

```python
result = feature.result() # submitした関数の結果を取得
```

他にも実行中の状態を確認する`running()`や`done()`、実行をキャンセルする`cancelled()`などがあります。

```python
import time
from concurrent.futures import ThreadPoolExecutor

def foo():
    time.sleep(3)
    return 'fooo'

with ThreadPoolExecutor() as executor:
    feature = executor.submit(foo)
    print(feature.running()) # True
    print(feature.done()) # False
    feature.cancelled() # 処理のキャンセル
    print(feature.running()) # False
    print(feature.done()) # True
```

# 逐次処理とマルチスレッド処理の実行速度比較
マルチスレッドの効果検証のために、逐次処理とマルチスレッドで実行速度を比較してみます。

## 下準備
時間のかかる I/O 処理のモックとなる関数を作ります。
`call_slow_request`は time.sleep(3) を使っているので 3 秒の実行時間がかかる処理です。

```python:util.py
def call_slow_request():
    time.sleep(3)
    return 'ok'
```

また、処理時間の計測をしたいので専用のデコレータを作ります。
実行関数に`processing_time`のデコレータをつけることで関数の実行時間が出力されます。

```python:util.py
def processing_time(func):
    @wraps(func)
    def wrapper(*args, **keywords):
        st = time.time()  # 開始前の時間を記録
        result = func(*args, **keywords)  # 関数を実行
        print(f'time: {time.time() - st} s')  # 開始後の時間と開始前の時間の差を出力
        return result

    return wrapper
```

## 逐次処理での実行

まず逐次処理です。`run_sequential`で`call_slow_request`を 3 回実行しています。

```python:run_sequential.py
from util import (call_slow_request, processing_time)


@processing_time
def run_sequential():
    for arg in range(3):
        print(call_slow_request())


if __name__ == '__main__':
    run_sequential()
```

3 秒待つ処理を 3 回順番に実行しているので、`run_sequential`の実行時間をは 9 秒弱となります。

```bash
$ python3 run_sequential.py
ok
ok
ok
time: 9.011210680007935 s
```

## マルチスレッドで実行
次にマルチスレッドです。
`ThreadPoolExecutor`を使って`call_slow_request`を 3 回実行しています。

```python:run_concurrent.py
from concurrent.futures.thread import ThreadPoolExecutor
from util import (call_slow_request, processing_time)


@processing_time
def run_concurrent():
    with ThreadPoolExecutor() as executor:
        features = [executor.submit(call_slow_request) for _ in range(3)]
        for feature in features:
            print(feature.result())


if __name__ == "__main__":
    run_concurrent()

```

逐次処理と比べて大幅に高速化しているのが分かると思います。

```bash
$ python3 concurrent-demo.py
ok
ok
ok
time: 3.0046510696411133 s
```

これは処理がスレッドに分かれて並行実行されるからです。
`call_slow_request`の`time.sleep(3)`に処理に入った段階で、次のスレッドに処理が移るのでほぼ sleep に指定した秒数とタイムラグなく処理を完了できます。

![](https://i.gyazo.com/fa8285f4f39fd9a2bc6fda183d555339.png)

# スレッドセーフとロック

並行処理についてまわる問題としてスレッドセーフがあります。
マルチスレッドで処理を並行実行する際に、共通した値をそれぞれのスレッドが読み書きするなどスレッドセーフではない処理を挟んでしまうとバグの原因になります。

スレッドセーフではない処理の例と、その対処（ロック）についてまとめます。


## スレッドセーフではない処理
以下は、`Counter`というクラスのインスタンス変数をカウントアップする処理です。スレッドセーフではないので、マルチスレッドで処理すると想定通りの値にならず実行ごとに計算結果が変わってしまいます。

```python:thread_safe.py
from concurrent.futures import (ThreadPoolExecutor, wait)


class Counter:
    def __init__(self):
        self.count = 0

    def count_up_to_1000000(self):
        for _ in range(1_000_000):
            self.count += 1


def worker(counter):
    counter.count_up_to_1000000()


def main():
    counter = Counter()
    with ThreadPoolExecutor() as executor:
        features = [executor.submit(worker, counter) for _ in range(3)]
    print(counter.count)


if __name__ == '__main__':
    main()
```

```bash
# 300000    0にならない!!!
$ python3 thread_safe.py
1848519

$ python3 thread_safe.py
1802127

$ python3 thread_safe.py
1702320
```

これは `self.count`をインクリメントする際の値読み取り中にスレッドが切り替わり、複数スレッドで同じ値に同時にアクセスすることで不整合な状態となってしまうためです。

## 改善（ロックの導入）
これを改善するためには、インクリメント中にスレッドの排他制御（ロック）をする必要があります。
改善コードはこちらです。出力も想定の 3000000 になっています。


```python:thread_safe.py
import threading
from concurrent.futures import (ThreadPoolExecutor, wait)


class Counter:
    lock = threading.Lock()

    def __init__(self):
        self.count = 0

    def count_up_to_1000000(self):
        for _ in range(1_000_000):
            with self.lock:
                self.count += 1


def worker(counter):
    counter.count_up_to_1000000()


def main():
    counter = Counter()
    with ThreadPoolExecutor() as executor:
        features = [executor.submit(worker, counter) for _ in range(3)]
    print(counter.count)


if __name__ == '__main__':
    main()
```

```bash
$ python3 thread_sage.py
3000000
```

Counter クラスの中で`Lock`インスタンスを作り、カウントアップを`Lock`インスタンスのコンテキストマネージャ内で実行しています。
`with self.lock:`のブロック内で行われる処理はロックがかかるので、カウントアップ時の複数スレッドでの値共有という不具合の原因が解消されます。

:::message
`Lock`インスタンスをコンテキストマネージャで実行しているのは、ロックの解放漏れを防ぐためです。コンテキストマネージャ使わない場合は、`lock.acquire()`でロックをかけ、`lock.release()`でロックを解放するという形でも出来るのですが、release を実行し忘れると、デッドロックになってまうので注意が必要です。
:::

# 終わりに

以上「Python の並行処理を理解したい [マルチスレッド編]」でした。
マルチスレッド以外にも、マルチプロセス、イベントループと他の並行処理もあるので続編書いていきたいです。


# 参考

ほぼこちらの書籍で学んだ内容です。良書ありがとうございます🙏

- [Python実践入門 ──言語の力を引き出し、開発効率を高める：書籍案内｜技術評論社](https://gihyo.jp/book/2020/978-4-297-11111-3)
- [並行処理から並列処理へ – IBM Developer](https://developer.ibm.com/jp/articles/j-java-streams-4-brian-goetz/)