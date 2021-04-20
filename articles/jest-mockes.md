---
title: "Jestのmockいろいろ"
emoji: "🍥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "JavaScript", "Jest", "Test"]
published: false
---

# ユーザー定義モジュールのClassのモック
`jest.mock(<module_path>)`でモジュールをモックする。`MockedClass<T>`を使うと楽。

```ts
import FooClass from './lib/fooClass'

// モック化
jest.mock('./lib/fooClass')
const FooClassMock = FooClass as jest.MockedClass<typeof FooClass>
```

## クラスメソッドのモック

```ts
describe('fooClass', () => {
  it('foo', () => {
    // クラスメソッドのモック
    FooClass.staticMethod = jest.fn().mockResolvedValue(true)

    // assert
  })
})
```

## コンストラクタ関数をモック

```ts
import FooClass from './lib/fooClass'

// モック化
jest.mock('./lib/fooClass', () => {
  return { instanceMethod: jest.fn() }
})

```

# 参考
https://jestjs.io/ja/docs/es6-class-mocks