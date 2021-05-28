---
title: "[Python] ネストされた辞書と配列の各値に対して再帰的に処理をしたい場合のコードサンプル"
emoji: "🐍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "再帰"]
published: true
---

ネストされた dict に対して再帰的に処理をしたい場面があったのでメモ。

# コードサンプル

最初に高階関数を作ります。
`create_recursive_action` は値の判定関数とその判定関数が適応した場合に行う関数を受け取り、再帰的に処理を行う関数を作る関数です。

```python
def create_recursive_action(is_target: Callable[[any], bool], action: Callable[[any], any]) -> Callable[[Union[dict, list]], Union[dict, list]]:
    def is_dict_or_list(obj: any) -> bool:
        return isinstance(obj, dict) or isinstance(obj, list)

    def inner(obj: Union[dict, list]) -> Union[
    dict, list]:
        if isinstance(obj, dict):
            return {
                k: action(v) if is_target(v) else inner(v) if is_dict_or_list(v) else v
                for k, v in obj.items()
            }
        if isinstance(obj, list):
            return [
                action(v) if is_target(v) else inner(v) if is_dict_or_list(v) else v
                for v in obj
            ]
    return inner
```

あとは、`lambda` を渡して目的の関数を作ります。

以下は与えられた辞書を再帰的に走査して、値が文字列の場合は upper() を適用する場合の例です。

```python
recursive_upper_case = create_recursive_action(lambda obj: isinstance(obj, str), lambda obj: obj.upper())

recursive_upper_case({
    'dict': {
        'str': 'aaa',
        'dict': {
            'str': 'aaa',
            'num': 1,
            'bool': True

        },
    },
    'list': [
        'bbb',
        {
            'dict': {
                'str': 'aaa',
                'num': 1,
                'bool': True
            },
        }
    ]
})
# {'dict': {'str': 'AAA',  'num': 1,  'bool': True,  'dict': {'str': 'AAA', 'num': 1, 'bool': True}}, 'list': ['BBB', {'dict': {'str': 'AAA', 'num': 1, 'bool': True}}]}
```

ユーザー入力のタグの除去などにも便利です。

```python
from django.utils.html import strip_tags

recursive_strip_tags = create_recursive_action(lambda obj: isinstance(obj, str), lambda obj: strip_tags(obj))

recursive_strip_tags({
  'payload': [
    { 'text': '<b>aaa</b>' }
    { 'text': '<b>bbb>' }
    [
      { 'text': '<b>ccc</b>' }
      { 'text': '<b>ddd</b>' }
    ]
  ]
})
# { 'payload': [ { 'text': 'aaa', { 'text': 'bbb' }, [ { 'text': 'ccc' }, { 'text', 'ddd' }]}]}