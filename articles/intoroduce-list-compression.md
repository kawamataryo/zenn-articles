---
title: "いろいろ出来るよ内包表記 [Python]"
emoji: "🥟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python"]
published: true
---

Python 特有の構文である内包表記についてまとめます。

# 内包表記とは？
内包表記はリストやセット、ハッシュなどを生成出来る構文です。
for 文でループを回してリスト等を作る処理をより簡潔にかけます。さらに内包表記で書くほうが実行速度もあがります。

例えば、0〜10 までの連番のリストが欲しいときの処理を for 文で書くと以下の通りです。

```python
my_list = []
for i in range(10):
    my_list.append(i)
# my_list: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

これを、内包表記で書くとわずか 1 行で簡潔にかけます。

```python
my_list = [i for i in range(10)]
# my_list: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```


内包表記は、これ以外にも様々な処理を実行できます。以降で紹介していきます。

# 様々な内包表記

## 条件分岐を加える
内包表記では内部で`if`を使ってリストの filter のような処理をかけます。

**my_list から奇数のみを抽出した odd_list を作る場合**

for を使う例

```python
my_list = [1, 2, 3, 4, 5]
odd_list = []
for i in my_list:
    if i % 2 != 0:
        odd_list.append(i)
# odd_list: [1, 3, 5]
```

内包表記を使う例

```python
my_list = [1, 2, 3, 4, 5]
odd_list = [i for i in my_list if i % 2 != 0]
# odd_list: [1, 3, 5]
```

## 結果に処理を加える
内包表記は結果に対して処理を加えることが出来るので map 的な処理も可能です。

**my_list のそれぞれの値を大文字にする upper_list を作る場合**

for 文を使う例

```python
my_list = ['a', 'b', 'c']
upper_list = []
for s in my_list:
   upper_list.append(s.upper())
# upper_list: ['A', 'B', 'C']
```

内包表記を使う例

```python
my_list = ['a', 'b', 'c']
upper_list = [s.upper() for s in my_list]
# upper_list: ['A', 'B', 'C']
```

## ネストしたループを処理する
内包表記はループのネストも可能です。

**2 次元リストを作る場合**

for 文を使った例

```python
my_list = []
for x in range(2):
    for y in range(2):
        my_list.append([x, y])
# my_list: [[0, 0], [0, 1], [1, 0], [1, 1]]
```

内包表記を使った例

```python
my_list = [[x, y] for x in range(2) for y in range(2)]
# my_list: [[0, 0], [0, 1], [1, 0], [1, 1]]
```

## ハッシュを作る
内包表記でハッシュを作ることも可能です。

**my_list の値をキー、インデックスを値とするハッシュを作る場合**

for を使った例

```python
my_list = ["foo", "bar", "baz"]
my_dict = {}
for val, index in enumerate(my_list):
    my_dict[val] = index
# my_dict: {0: 'foo', 1: 'bar', 2: 'baz'}
```

内包表記を使った例

```python
my_list = ["foo", "bar", "baz"]
my_dict = {index:val for val, index in enumerate(my_list)}
# my_dict: {0: 'foo', 1: 'bar', 2: 'baz'}
```

## セットを作る
内包表記でセットを作ることも可能です。

**my_list の各値に各値に 2 を掛けた値のセットを作る場合**

for を使った例

```python
my_list = [1, 2, 2, 3, 4, 3]
my_set = set([])
for i in my_list:
    my_set.add(i * 2)
# my_set: {2, 4, 6, 8}
```

内包表記を使った例

```python
my_list = [1, 2, 2, 3, 4, 3]
my_set = { i * 2 for i in my_list }
# my_set: {2, 4, 6, 8}
```

## ジェネレータを作る
内包表記でジェネレータを作ることも可能です。

**0〜9 の値の連番を返す my_generator を作る場合**

for の例

```python
def build_generator():
    for i in range(10):
        yield i
my_generator = build_generator()
# <generator object my_generator at 0x1103335f0>
```

内包表記の例

```python
my_generator = (i for i in range(10))
# <generator object my_generator at 0x1103335f0>
```

# 終わりに
内包表記初めてみたときは「読みづらい。.」と思ったのですが理解するとかなり便利に使えますね。
map 的な処理と filter 的な処理ができるのも良いです。使いこなしていきたいです。

# 参考

- [Python実践入門 ──言語の力を引き出し、開発効率を高める：書籍案内｜技術評論社](https://gihyo.jp/book/2020/978-4-297-11111-3)
