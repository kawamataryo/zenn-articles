---
title: "Pythonの並行処理を理解したい理解したい [マルチスレッド編]"
emoji: "🧵"
type: "tech"
topics: ["python"]
published: false
---

# Pythonの並行処理を理解したい（マルチスレッド編）

# 並行処理とは？

# マルチスレッドとは？

# マルチスレッドを使ってみる

Python のマルチスレッドは concurrent.futures の ThreadPoolExecutor を使います。
例として、time.sleep を使って処理に時間がかかる関数を作り、その関数を複数実行した場合の処理時間を、逐次処理とマルチスレッドで比べてみます。

## 逐次処理

まず逐次処理です。`call_slow_request`が I/O バウンドな処理のモック関数です。
`run_sequential` で 3 回実行しています。
処理の前後で tim.time()の値を比較することで実行時間を表示しています。

```python:sequential.py
import time

def call_slow_request():
    time.sleep(3)
    return 'ok'

def run_sequential():
    for arg in range(3):
        call_slow_request()

st = time.time() # 実行前の時間を保存
run_sequential() # 実行
print(f"time: {time.time() - st}") #実行後の時間と実行前の時間の差を出力
```

これを実行すると以下のような出力がでます。
3 秒待つ処理を 3 回順番に実行しているので、`run_sequential`の実行時間をは 9 秒弱となります。

```bash
$ python3 sequential.py
time: 9.005066871643066
```

## マルチスレッド
次に同様の処理をマルチスレッドで行う例です。
新しく run_concurrent を作り、そこで`ThreadPoolExecutor`の send で処理を別スレッドに流しています。
これで並行処理が実現出来ます。

```python:concurrent-demo.py
import time
from concurrent.futures import (ThreadPoolExecutor, wait)

def call_slow_request():
    time.sleep(3)
    return 'ok'

def run_concurrent():
    with ThreadPoolExecutor() as executor:
        futures = [executor.submit(call_slow_request) for _ in range(3)]
        wait(futures)

st = time.time() # 実行前の時間を保存
run_concurrent() # 実行
print(f"time: {time.time() - st}") #実行後の時間と実行前の時間の差を出力
```

実行すると逐次処理と比べて大幅に高速化しているのが分かると思います。

```bash
$ python3 concurrent-demo.py
time: 3.0046510696411133
```
