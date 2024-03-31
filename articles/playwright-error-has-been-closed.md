---
title: "PlaywrightでTarget page, context or browser has been closed"
emoji: "🚨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["playwright"]
published: true
---

とても単純な理由なのに、ハマったのでメモ。


# 🚨 エラー内容

Playwrightでテストを書いたていた時に、なぜかlocatorのaut-waitが効かず、以下エラーが発生しました。

```test.ts
test('some test', async (page) => {
  await page.goto('https://example.com')

  // 操作
  await page.locator('button').click()

  // ここでエラーが発生: Error: expect.toVisible: Target page, context or browser has been closed
  expect(page.getByRole('header', { name: 'Success' })).toBeVisible()
})
```

エラー内容

```
Error: expect.toBeHidden: Target page, context or browser has been closed
```

locatorを使っているので、Playwrightのauto-waitingが効くはずなのに、なぜ・・？と30分くらい時間を費やしてしまいました😇

https://playwright.dev/docs/actionability#assertions


# 💡 解決策

expectにawaitを追加するだけです。よくよく考えれば当たり前ですが、当時はexpectをawaitするという発想がなく、ハマってしまいました。

```test.ts
test('some test', async (page) => {
  await page.goto('https://example.com')

  // 操作
  await page.locator('button').click()

  // awaitするだけ
  await expect(page.getByRole('header', { name: 'Success' })).toBeVisible()
})
```

# 📚 学び

冷静にドキュメントを読もう。

https://playwright.dev/docs/test-assertions#auto-retrying-assertions
