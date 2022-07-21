---
title: "Flask-Caching で Flask の View をキャッシュする"
emoji: "🦩"
type: "tech"
topics: ["flask", "flask-caching", "python"]
published: false
---

[ISUCON12](https://isucon.net/) に向けて flask のキャッシュについて調べたメモ。

# Flask-Caching とは？

[Flask-Caching](https://flask-caching.readthedocs.io/en/latest/) は Flask の View やその他関数にて容易にキャッシュ機能を実装するためのライブラリです。

キャッシュのバックエンドとして、Redis や Memcached、UWSGICache など、さまざまなミドルウェアを指定することが出来、それぞれを統一したインターフェイスとして扱えるという特徴があります。

https://flask-caching.readthedocs.io/en/latest/

[Flask の公式ドキュメントでも紹介](https://flask.palletsprojects.com/en/2.1.x/patterns/caching/?highlight=cache)されているので、Flask でキャッシュを使う際の第 1 選択肢として考えて良いと思います。

# セットアップ

本記事では Redis でのセットアップを紹介します。

Flask の環境を作ったら、pip でライブラリをインストールします。Redis をバックエンドとして使うため [redis-py](https://github.com/redis/redis-py) も合わせて追加します。

```bash
pip install Flask-Caching redis
```

次に、Flask のエントリーポイントに Flask-Cache の設定を追記します。

```python:app.py
from flask import Flask
from flask_caching import Cache

config = {
    "CACHE_TYPE": "RedisCache",
    "CACHE_REDIS_HOST": "localhost",
    "CACHE_REDIS_PORT": 6379,
}

app = Flask(__name__)
app.config.from_mapping(config)
cache = Cache(app)
```

config で `CACHE_TYPE` に `RedisCache` を指定することで Redis のキャッシュを有効化して、`CACHE_REDIS_HOST`と`CACHE_REDIS_PORT`で Redis の接続先を指定しています。もし Redis でパスワード認証を行っている場合は、`CACHE_REDIS_PASSWORD` でパスワードの指定が可能です。

# キャッシュ API

Flask-Caching が提供するキャッシュ API をみていきます。

## View のキャッシュ

View のキャッシュは`@cache.cached`デコレータで行います。

以下`/heavy_view`は内部で 10 秒間 sleep する View ですが、初回アクセス以降は Flask-Cache のキャッシュがヒットして、即時にレスポンスを返すことができます。

```python
@app.route('/heavy_view/<string:name>')
@cache.cached()
def heavy_view(name):
    time.sleep(10)
    return f'hello {name} !!!!!!'
```

標準でキャッシュのキーはリクエストのパスごとに異なります。
一度 `/heavy_view/foo` にアクセスしたあと再度`/heavy_view/foo`へアクセスするとキャッシュがヒットしますが、`/heavy_view/bar`にアクセスした場合はキャッシュがヒットせず内部の処理が実行されます。


また、`@cache.cached`のデコレータに引数を指定することでキャッシュの制御が行えます。

- `timeout` : キャッシュの保存時間。秒数で指定。config で`CACHE_DEFAULT_TIMEOUT`を指定することで標準の保存時間を指定することが可能。もしどちらも指定がない場合は無期限でキャッシュを返す。
- `key_prefix` : キャッシュキーのプレフィクス。無指定の場合は、`view/<リクエストのpath>`が設定される（上記サンプルコードだと`view//heavy_view/xx`。

詳細な引数は[API リファレンス](https://flask-caching.readthedocs.io/en/latest/api.html#cache-api)を参照してください。

キャッシュの削除は`cache.clear()`または`cache.delete()`で行えます。

```python
# 存在するすべてのキャッシュを削除
cache.clear()

# 引数で指定したキーのキャッシュを削除
cache.delete('key_name')
```

`cache.delete` で前述のサンプルコードのキャッシュのみ削除する場合は、`cache.delete('view//heavy_view')` です。

## View 以外の関数のキャッシュ

View 以外の関数のキャッシュには `@cache.memoize` デコレータが使えます。

`@cache.memoize`は関数名と、引数をキーとしてキャッシュを作成する点が、`@cache.cached`とは異なります。
以下サンプルコードで、`heavy_func(1, 2)`を 2 回実行した場合、初回は 10 秒かかりますが、2 回目以降は瞬時に値が返ります。

```python
@cache.memoize()
def heavy_func(num1, num2):
    time.sleep(10)
    return num1 + num2
```

`@cache.memoize` でも、`timeout`や`key_prefix`を引数で指定できます。
詳細な引数は[API リファレンス](https://flask-caching.readthedocs.io/en/latest/api.html#cache-api)を参照してください。

キャッシュの削除は`cache.delete_memoized` で行います。
`cache.delete_memoized` の第二引数以降に値を指定することで、特定の引数で memoize した関数が呼ばれた場合のキャッシュのみを削除することも可能です。

```python
# heavy_func関数のキャッシュをすべて削除
cache.delete_memoized(heavy_func)
# heavy_func(1, 2)で呼ばれた場合のキャッシュのみを削除
cache.delete_memoized(heavy_func, 1, 2)
```

## 任意の値のキャッシュ

Flask-Caching はデコレータでの利用以外でも、`cache.set`と`cache.get`を使うことでキャシュバックエンドとして指定したミドルウェアへのクライアントとしても使うことができます。

```python
# cache_keyというキー名で、'cached value'という文字列を保存
cache.set('cache_key', 'cached value')
# 'cached_value' が返却される
cache.get('cache_key')
```

# サンプルアプリ

最後に少しだけ現実的な例を。RESTful API のサンプルアプリです。

```python:app.py
from flask import Flask, jsonify, request
import time

app = Flask(__name__)

book_list = [{'name': 'foo', 'price': 100}]


@app.get('/books')
def books():
    _super_heavy_func()
    return jsonify(book_list)


@app.post('/books')
def add_book():
    book_list.append(request.json)
    return jsonify(book_list)


@app.get('/books/<string:name>')
def get_book(name):
    _super_heavy_func()
    book = next((book for book in book_list if book['name'] == name), None)
    return jsonify(book) if book else 'Book not found', 404


@app.put('/books/<string:name>')
def update_book(name):
    book = next((book for book in book_list if book['name'] == name), None)
    if not book:
        return 'Book not found', 404
    book['price'] = request.json['price']
    return '', 204


def _super_heavy_func():
    time.sleep(10)
```

本の一覧を取得する`/books`と本の詳細を取得する`/books/<name>`の GET API では、内部でとても重い処理（`_super_heavy_func`）を実行しているので、毎回レスポンスに 10 秒以上かかります。
これではとても体験が悪いので、キャッシュを導入することとします。以下、Flask-Caching を導入した場合の例です。

```python
from flask import Flask, jsonify, request
from flask_caching import Cache
import time


config = {
    "DEBUG": True,
    "CACHE_TYPE": "RedisCache",
    "CACHE_REDIS_HOST": "localhost",
    "CACHE_REDIS_PORT": 6379,
}
app = Flask(__name__)
app.config.from_mapping(config)
cache = Cache(app)

book_list = [{'name': 'foo', 'price': 100}]


@app.get('/books')
@cache.cached()
def books():
    _super_heavy_func()
    return jsonify(book_list)


@app.post('/books')
def add_book():
    book_list.append(request.json)
    cache.delete('view//books')
    return jsonify(book_list)


@app.get('/books/<string:name>')
@cache.cached()
def get_book(name):
    _super_heavy_func()
    book = next((book for book in book_list if book['name'] == name), None)
    return jsonify(book) if book else 'Book not found', 404


@app.put('/books/<string:name>')
def update_book(name):
    book = next((book for book in book_list if book['name'] == name), None)
    if not book:
        return 'Book not found', 404
    book['price'] = request.json['price']
    cache.delete(f'view//books/{request.json["name"]}')
    return '', 204


def _super_heavy_func():
    time.sleep(10)
```

`/books`と`/books/<name>`の GET API で`@cache.cached`を使ってレスポンスをキャッシュしています。2 回目以降のアクセスではキャッシュがヒットするため、高速に動作します。

ただ重い API に`@cache.cached`を追加しただけだと、本の変更が反映されず困るのですが、本の追加の`/books`の POST API と、本の内容更新の`/books/<name>`の PUT API にて対応するキャッシュを削除するこで変更が反映されるようにしています。

これでパフォーマンスは向上させつつ、仕様を満たすことができます。

# おわりに

ISUCON12 頑張るぞい！！
