---
title: "Mock Service Worker でJest.mockを使わず非同期リクエストのテストを書く"
emoji: "🍊"
type: "tech"
topics: ['mockserviceworker', "typescript", "mock", "jest"]
published: false
---

# Mock Service Worker とは？
Mock Service Worker は、ネットワークレベルで API リクエストをインターセプトして mock のデータを返すためのライブラリです。API リクエストを含む処理のテストや、SPA 開発時の mock サーバーとして利用出来ます。

https://mswjs.io/

今回は、テストでの利用に着目してサンプルコードを書きます。

# テストでの利用例
## テスト対象コード

以下ログインフォームのコードを対象にテストを書いていきます。
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
        console.log('response', e)
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

# mswを使ってnetworkレベルでモックしたコード

```ts:Login.test.ts
import { rest } from "msw"
import { render, screen, fireEvent } from '@testing-library/vue'
import { setupServer } from "msw/node";
import Login from "../../components/Login.vue"


const VALID_USER = {
  username: 'validUsername',
  password: 'validPassword'
}

export const mockServer = setupServer(
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

# 他フレームワークとの比較

# 終わりに