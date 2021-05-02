---
title: "Mock Service Worker でjest.mockを使わず非同期リクエストのテストを書く"
emoji: "🍊"
type: "tech"
topics: ['mockserviceworker', "typescript", "mock", "jest"]
published: false
---
[Mock Service Worker](https://mswjs.io/) を使ってみたらかなり良かったので紹介です。

# Mock Service Worker とは？
Mock Service Worker（以下 msw）は、ネットワークレベルで API リクエストをインターセプトして mock のデータを返すためのライブラリです。API リクエストを含む処理のテストや、SPA 開発時の mock サーバーとして利用出来ます。

https://mswjs.io/

以下テスト利用の場合のサンプルコードです。
`setupServer`でインターセプト用のサーバーを定義し、`listen()`でインターセプトをスタート、`close()`でインターセプトをストップします。

:::message
`setupServer`という関数名からは実際にサーバーを立てるように思われるのですが、内部的には Node.js の API リクエスト用のネイティブモジュールである`https`や`XMLHttpRequest`を補強することで動作しているようです。
https://mswjs.io/docs/api/setup-server#operation
:::

```ts
import { rest } from "msw"
import { setupServer } from "msw/node";
import axios from "axios";


// mockサーバー
const mockServer = setupServer(
  rest.get('/greeting', (req, res, ctx) => {
    return res(ctx.status(200), ctx.json('Hello'))
  })
)

// テスト対象の関数
const greeting = async (name: string) => {
  const word =  await axios.get('/greeting')
  return `${word.data} ${name}`
}

describe('greeting', () => {
  beforeAll(() => {
    // インターセプトスタート
    mockServer.listen()
  })
  afterAll(() => {
    // インターセプトストップ
    mockServer.close()
  })

  test('挨拶を返す', async () => {
    const result = await greeting('ryo')

    expect(result).toEqual('Hello ryo') // Green
  })
})
```

## なぜネットワークレベルでのmockが必要なの？

なぜ、`jest.mock`ではなくネットワークレベルのモックが必要かというと、mock のレイヤーが低レイヤーになるほどテストコードがより安全になるからです。次章で詳細な例を書きますが、非同期リクエストを投げるモジュールをモックしてしまうと、そのモジュール自体のバグにはそのテストで担保できなくなってしまいます。
より安全なテストコードを書くためには、モックをなるべく使わない or モックのレイヤーを下げることが必要です。

※ モックを使うことでテスト実行のパフォーマンスを向上させるというメリットもあるのでそこはトレードオフです。


# Vueコンポーネントのテストでの利用例

実際のユースケースでありそうなログイン処理についてのテストコードを書いていきます。
テストフレームワークは Jest と、Vue Testing Library を使います。

https://github.com/facebook/jest

https://github.com/testing-library/vue-testing-library

## テスト対象のコード

以下ログインフォームのコンポーネントを対象にテストを書いていきます。
このコンポーネントの機能は以下です。

- ユーザー名とパスワードを入力出来る
- `submit` ボタンを押すとユーザー名とパスワードを使ってログイン API をコールする
- ログインに成功すると`Hello <user name>`を表示する
- ログインに失敗すると`Error try again.`と表示する

```html:Login.vue
<template>
  <h1 v-if="user">
    Hello, {{ user.name }}
  </h1>
  <form @submit.prevent="handleAuth">
    <label for="username">Username:
      <input v-model="formData.username" id="username" name="username"/>
    </label>
    <label for="password">Password:
      <input v-model="formData.password" id="password" name="password"/>
    </label>
    <button>submit</button>
  </form>
  <span v-if="error" data-testid="error">{{ error }}</span>
</template>

<script lang="ts">
import { defineComponent, reactive, ref } from "@vue/runtime-core";
import axios from 'axios';

type User = {
  name: string,
}

export default defineComponent({
  setup() {
    const formData = reactive({
      username: '',
      password: '',
    })
    const user = ref<null | User>(null)
    const error = ref<null | string>(null)

    const handleAuth = async () => {
      try {
        const response = await axios.post('/login', {
          username: formData.username,
          password: formData.password
        })
        user.value = response.data
      } catch (e) {
        error.value = e.response.data.error
      }
    }
    return {
      formData,
      handleAuth,
      user,
      error
    };
  }
});
</script>
```


# Jest.mockを使ってaxiosをモックしたコード

まず最初に、よくある Jest.mock を使って axios をモックする例です。
`jest.mock('axios')`で axios のモジュールをモックして、`mockResolvedValue`、`mockRejectedValue`でモックの戻り値を定義することで正常系と異常系の UI をテストしています。

```ts:Login.vue.test
import { render, screen, fireEvent } from '@testing-library/vue'
import Login from "../../components/Login.vue"
import axios from 'axios'

jest.mock('axios')
const mockedAxios = axios as jest.Mocked<typeof axios>

describe('Login', () => {
  test('ログインが成功した場合にユーザー名を表示する', async () => {
    mockedAxios.post.mockResolvedValue(  { data: { name: 'validUsername'}})

    render(Login)

    expect(screen.queryByText('Hello, validUsername')).toBeFalsy()

    await fireEvent.update(screen.getByLabelText(/username/i), 'validUser')
    await fireEvent.update(screen.getByLabelText(/password/i), 'validPassword')
    await fireEvent.click(screen.getByText('submit'))

    expect(await screen.findByText('Hello, validUsername')).toBeTruthy()
    expect(screen.queryByTestId('error')).toBeFalsy()
  })

  test('ログインが失敗した場合にエラーを表示する', async () => {
    mockedAxios.post.mockRejectedValue( { response: { data: { error: 'error: invalid username or password'}} })

    render(Login)

    expect(screen.queryByText('Hello, validUsername')).toBeFalsy()

    await fireEvent.update(screen.getByLabelText(/username/i), 'invalidUser')
    await fireEvent.update(screen.getByLabelText(/password/i), 'invalidPassword')
    await fireEvent.click(screen.getByText('submit'))

    expect(await screen.findByTestId('error')).toBeTruthy()
    expect(screen.queryByText('Hello')).toBeFalsy()
  })
})
```


一見問題はないのですが、このテストの場合 axios をそのまま mock しているので axios のモジュール自体にバグが入って機能しなくなった場合や、BREAKING CHANGE で axios の戻り値が変化した場合（例えば`data`でのラップがなくなるなど）でもテストは通貨してしまいます。
# mswを使ってnetworkレベルでモックしたコード

続いて msw を使った例です。`setupServer`で`/login`への POST リクエストに対してインターセプトの定義を行っています。内部でリクエストの username と password の比較を行い正常系と異常系のレスポンスを分けています。
テストコードはとてもシンプルです。特に`jest.mock`を使わずユーザー入力と同じようにただフォームに値を入れて送信しているだけです。

```ts:Login.test.ts
import { rest } from "msw"
import { render, screen, fireEvent } from '@testing-library/vue'
import { setupServer } from "msw/node";
import Login from "../../components/Login.vue"


const VALID_USER = {
  username: 'validUsername',
  password: 'validPassword'
}

const mockServer = setupServer(
  rest.post<Record<string, any>>('/login', (req, res, ctx) => {
    const { username, password } = req.body

    if (username !== VALID_USER.username && password !== VALID_USER.password) {
      return res(
        ctx.status(403),
        ctx.json({
          error: 'error: invalid username or password'
        })
      )
    }
    return res(
      ctx.status(200),
      ctx.json({
        name: 'validUsername',
      })
    )
  })
)

describe('Login', () => {
  beforeAll(() => mockServer.listen())
  afterEach(() => mockServer.resetHandlers())
  afterAll(() => mockServer.close())

  test('ログインが成功した場合にユーザー名を表示する', async () => {
    render(Login)

    expect(screen.queryByText('Hello, validUsername')).toBeFalsy()

    await fireEvent.update(screen.getByLabelText(/username/i), VALID_USER.username)
    await fireEvent.update(screen.getByLabelText(/password/i), VALID_USER.password)
    await fireEvent.click(screen.getByText('submit'))

    expect(await screen.findByText('Hello, validUsername')).toBeTruthy()
    expect(screen.queryByTestId('error')).toBeFalsy()
  })

  test('ログインが失敗した場合にエラーを表示する', async () => {
    render(Login)

    expect(screen.queryByText('Hello, validUsername')).toBeFalsy()

    await fireEvent.update(screen.getByLabelText(/username/i), 'invalid user')
    await fireEvent.update(screen.getByLabelText(/password/i), 'invalid password')
    await fireEvent.click(screen.getByText('submit'))

    expect(await screen.findByTestId('error')).toBeTruthy()
    expect(screen.queryByText('Hello')).toBeFalsy()
  })
})
```

これなら、`jest.mock`を使った場合と違って、axios のモジュール自体のバグや、BREAKING CHANGE で戻り値が変わった場合にはテストが失敗するのでテストレベルで動作を担保することが出来ます。

# 他フレームワークとの比較
msw のようなネットワークレベルでのインターセプトライブラリとしては`nock`も有名です。

https://github.com/nock/nock

msw のドキュメントに nock との比較があったので記載します。



# 終わりに

以上、「Mock Service Worker で jest.mock を使わず非同期リクエストのテストを書く」でした。`jest.mock`を使う場合と比べ少し記述量は増えますが、よりテストでの安全性を担保できるので良さそうです。また、パスの間違えなどで地味にモックされず困ることもなくなります。業務コードでの導入も検討したいです。