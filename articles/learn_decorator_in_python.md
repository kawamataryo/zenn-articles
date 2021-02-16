---
title: "Pythonのデコレータを理解するまで"
emoji: "🍰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "django"]
published: true
---

Python のデコレータの学習メモです。

# デコレータとは？
デコレータは関数やクラスの前後に特定の処理を追加できる機能です。関数やクラス宣言の前に`@デコレータ名`を記述することで実現できます。クラスで静的メソッドを作る際の`@staticmethod`や Django の `@login_required`など何気なく使っていますね。

# 作り方
関数デコレータを例にデコレータの作り方を見ていきます。

## デコレータの実態

関数デコレータの実態は**関数を受け取り関数を返す関数**です。
例えば`@foo`というデコレータを`bar`関数に適応させるには以下のようにします。

```python
@foo
def bar():
    pass

bar()
```
これは実質以下と同義です。

```python
foo(bar)()
```

`@xxx`という書き方をみると何か変わった機能なのかと思うかもしれませんが、ただの関数を受け取り関数を返す関数というのを頭に入れておくと、デコレータの理解が進むと思います。

## 関数デコレータの作成
関数の開始時間・終了時間・処理結果をプリントする`my_logger`デコレータを作ってみます。

```python
import datetime

def my_logger(f):
    def _wrapper(*args, **keywords):
        # 前処理
        print(f'{f.__name__}の実行')
        print(f'開始: {datetime.datetime.now()}')

        # デコレート対象の関数の実行
        v = f(*args, **keywords)

        # 後処理
        print(f'終了: {datetime.datetime.now()}')
        print(f'実行結果: {v}')

        return v
    return _wrapper
```

my_logger の内部で`_wrapper`という関数を宣言し、my_logger の戻り値として_wrapper を返しています。そして_wrapper では、my_logger の引数として受け取った関数を実行し、その戻り値を返します。前処理と後処理については、_wrapper 内で引数として受け取った関数を実行する前後に記述すれば OK です。
このように、関数を受け取り関数を返す関数を作ればデコレータは完成です。

実効結果は以下となります。
しっかり前後の処理が実行されていますね。

```python
@my_logger
def return_one():
    return 1

return_one()
# return_oneの実行
# 開始: 2021-02-16 09:31:40.616712
# 終了: 2021-02-16 09:31:40.616742
# 実行結果: 1
```

## 引数を受け取る関数デコレータの作成

引数を受け取るデコレータは、関数を受け取り関数を返す関数をさらに関数でラップすることで実現できます。
デコレータの引数で指定されたタグ名で、関数の戻値をラップする`tag`デコレータを作ってみます。

```python
def tag(tag_name):
    def _tag(f):
        def _wrapper(*args, **keywords):
            v = f(*args, **keywords)
            return f'<{tag_name}>{v}</{tag_name}>'
        return _wrapper
    return _tag
```

デコレータの引数を受け取る`tag`、デコレートする関数を受け取る`_tag`、前後の処理を実行する`_wrapper`の三層構造になっています。
実行すると関数の戻り値を、デコレータの引数に渡したタグ名でラップしてくれます。

```python
@tag('h1')
def greet():
    return 'hello decorator'

print(greet())
# <h1>hello decorator</h1>
```

`@`を使わない形に書き直すと以下のようになります。
関数を受け取って関数を返す関数という部分は同様ですね。

```python
def greet():
    return 'hello decorator'

decorated_greet = my_tag('h1')(greet)

print(decorated_greet())# <h1>hello decorator</h1>
```

## @wrapsでメタデータを保持する
デコレータの実装時に使える便利なデコレータとして`@functools.wraps`があります。
`@functools.wraps`を使うとデコレータ適応時の関数のメタ情報書き換えを防ぐことができます。

例えば、先程作った`my_logger`を適用した return_one 関数の`__name__`を参照すると`return_one`ではなく`_wrapper`が返されてしまいます。
デコレータが適用された時点で return_one 関数は my_logger の _wrapper で置き換えられているのでこのような動きとなります。

```python
@my_logger
def return_one():
    return 1

print(return_one.__name__) # _wrappe
```

この状態を解消するのが`@functools.wraps`です。
`@functools.wraps`を使うと _wrapper のメタ情報へのアクセスで、デコレート対象の関数のメタ情報を返してくれるようになります。

my_loggeer を`@functools.wraps`を使って実装し直すと正しい関数名を取得出来ます。

```python
import datetime
import functools

def my_logger(f):
    @functools.wraps(f)
    def _wrapper(*args, **keywords):
      # ...
    return _wrapper

@my_logger
def return_one():
    return 1

print(return_one.__name__) # return_one
```

デコレータを作るときは特別な理由がない限り`functools.wraps`を使ったほうが良さそうです。

# ライブラリの実装を読んでみる - Django login_required -

最後に、せっかくデコレータの仕組みを学んだので実コードではどのように利用されているのか Django の`login_required`の実装を読んでみます。

https://github.com/django/django/blob/5fcfe5361e5b8c9738b1ee4c1e9a6f293a7dda40/django/contrib/auth/decorators.py

login_required の処理は`django/contrib/auth/decorators.py`に記述されています。

:::details django/contrib/auth/decorators.py
```python:django/contrib/auth/decorators.py
def user_passes_test(test_func, login_url=None, redirect_field_name=REDIRECT_FIELD_NAME):
    """
    Decorator for views that checks that the user passes the given test,
    redirecting to the log-in page if necessary. The test should be a callable
    that takes the user object and returns True if the user passes.
    """

    def decorator(view_func):
        @wraps(view_func)
        def _wrapped_view(request, *args, **kwargs):
            if test_func(request.user):
                return view_func(request, *args, **kwargs)
            path = request.build_absolute_uri()
            resolved_login_url = resolve_url(login_url or settings.LOGIN_URL)
            # If the login url is the same scheme and net location then just
            # use the path as the "next" url.
            login_scheme, login_netloc = urlparse(resolved_login_url)[:2]
            current_scheme, current_netloc = urlparse(path)[:2]
            if ((not login_scheme or login_scheme == current_scheme) and
                    (not login_netloc or login_netloc == current_netloc)):
                path = request.get_full_path()
            from django.contrib.auth.views import redirect_to_login
            return redirect_to_login(
                path, resolved_login_url, redirect_field_name)
        return _wrapped_view
    return decorator


def login_required(function=None, redirect_field_name=REDIRECT_FIELD_NAME, login_url=None):
    """
    Decorator for views that checks that the user is logged in, redirecting
    to the log-in page if necessary.
    """
    actual_decorator = user_passes_test(
        lambda u: u.is_authenticated,
        login_url=login_url,
        redirect_field_name=redirect_field_name
    )
    if function:
        return actual_decorator(function)
    return actual_decorator
```
:::

login_required は内部で `user_passes_test` 関数を実行してその戻り値を返しています。

```python
def login_required(function=None, redirect_field_name=REDIRECT_FIELD_NAME, login_url=None):
# ...
    actual_decorator = user_passes_test(
        # ...
    )
    if function:
        return actual_decorator(function)
    return actual_decorator
```
処理の実態は user_passes_test の内部で定義されている`_wrapped_view`関数です。
この関数の一行目の`if test_func(request.user)`という条件分岐でログイン済みのリクエストかどうかを判定して、処理を分岐させているようです。

条件を通過する場合は、デコレート対象の view の関数を実効し、通過しない場合はログイン画面へのリダイレクトなどを行っています。

```python
def user_passes_test(test_func, login_url=None, redirect_field_name=REDIRECT_FIELD_NAME):
    # ...
    def decorator(view_func):
        @wraps(view_func)
        def _wrapped_view(request, *args, **kwargs):
            if test_func(request.user):
                return view_func(request, *args, **kwargs)
            # ...
            return redirect_to_login(
                path, resolved_login_url, redirect_field_name)
        return _wrapped_view
    return decorator
```

判定処理`test_func`の実態は、login_required の内部で user_passes_test の第一引数に渡している`lambda u: u.is_authenticated,`というラムダです。なので、login_required では、`request.user`に対して、`is_authenticated` 関数を呼び出してログイン判定を行っているということですね。

```python
def login_required(function=None, redirect_field_name=REDIRECT_FIELD_NAME, login_url=None):
# ...
    actual_decorator = user_passes_test(
        lambda u: u.is_authenticated,
        login_url=login_url,
        redirect_field_name=redirect_field_name
    )
# ...
```

# 終わりに
以上「Python のデコレータを理解するまで」でした。
デコレータは利用側から見るとわりとブラックボックスな処理だったので、この機会に勉強できてよかったなと思っています。使いこなしていきたいです。

# 参考
- [Pythonのデコレータを理解するための12Step - Qiita](https://qiita.com/_rdtr/items/d3bc1a8d4b7eb375c368)
- [Python実践入門 ──言語の力を引き出し、開発効率を高める：書籍案内｜技術評論社](https://gihyo.jp/book/2020/978-4-297-11111-3)