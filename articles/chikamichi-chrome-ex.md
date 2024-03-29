---
title: "閲覧履歴・ブックマーク・タブを横断的に検索し移動できるChrome拡張を作ってみた"
emoji: "🛣"
type: "tech"
topics: ["vue","typescript","extension","chrome"]
published: true
---

個人的に超便利な Chrome 拡張を作ってみたので紹介です。

# 作ったもの

Chikamichi（近道）という閲覧履歴・ブックマーク・タブを横断的に検索し移動できる Chrome 拡張です。

|機能|
|---|
|履歴・ブックマーク・現在開いているタブを横断的にインクリメンタルサーチ|
|クリック・エンターでの対象への移動|
|キーボードショートカットでのスムーズな操作|

デザイン・機能は[Sidekick](https://www.meetsidekick.com/)のサーチダイアログを参考にしました。このデモのように、直感的にサクサク操作できるのでお試しください〜

https://www.youtube.com/watch?v=Oi8MlZeaa4Y


技術スタックは`TypeScript`、`Vue.js`、`Windi.css`です。コードも GitHub 上に公開しています。
コントリビューションも大歓迎です。

https://github.com/kawamataryo/chikamichi

:::message
`Chikamichi` という名前は[エンジニアと人生](https://community.camp-fire.jp/projects/view/280040)というコミュニティで[@にっしー](https://twitter.com/paranishian)さんに案を頂きました。とても気に入っています。感謝🙏
:::

# 使い方
Chrome Web Store 又は Firefox Add on からインストールできます。

https://chrome.google.com/webstore/detail/chikamichi/gkhobepjbiepngbeikhbpnfgjcjgmgha?hl=ja&authuser=1

https://addons.mozilla.org/en-GB/firefox/addon/chikamichi/

マウスを使わずキーボードだけで操作可能です。

| コマンド                       | 動作                                  |
|-------------------------------|------------------------------------------|
| Alt + k                       | サーチダイアログの起動                       |
| ↓ or ↑ (Ctrl + n or Ctrl + p) | 検索結果の選択                    |
| Enter                         | 対象のサイトを現在のタブで開く（タブの場合はタブへの移動）            |
| Ctrl + Enter                  | 対象のサイトを新しいタブで開く（タブの場合はタブへの移動） |

:::message alert
chrome の new tab ページ及び Chrome Web Store では、Chrome Extension の content_script の読み込みが出来ず、`Alt + k` でも起動しないので注意です。Chrome の仕様のようです。

https://stackoverflow.com/questions/30474892/chrome-extension-content-script-not-working-on-chrome-webstore-page
:::

**2022/01/17 追記**

prefixで検索対象を絞り込める機能も追加しました！
`/b`でブックマークだけの検索、`/t`でタブだけの検索、`/h`で履歴だけの検索となります。

https://twitter.com/KawamataRyo/status/1481784597117472768


# 工夫したところ

## Fuese.jsでのFuzzy Search
ただの完全一致や部分一致だと良い検索結果が得られなかったので、Fuse.js というライブラリを利用して、サイトタイトルと URL を対象に Fuzzy Search を実装しています。入力値との一致具合によってソートされるので、わずかな入力でもより良い検索結果が得られます。

https://github.com/krisk/fuse

## Dark modeへの対応
Windi CSS の Dark mode 機能を使って Dark mode に対応しています。本当は Dark mode の ON・OFF も設定できればよいのですが、現状は OS の設定によって切り替える形としています。

https://windicss.org/features/dark-mode.html

|light|dark|
|---|---|
|![](https://i.gyazo.com/23d7607222a999effb1ad85b4a4870bb.png)|![](https://i.gyazo.com/82be770e2ce6dfa792c62468e6a5d0d4.png)|

## キーボードショートカットでの操作
このプラグインのモチベーションはマウスをなるべく使わず、キーボードだけで移動したいというものだったので、ほぼ全ての動作をキーボードショートカットで行えるようにしました。Vue の key イベントを使って実装しています。

https://github.com/kawamataryo/history-search/blob/72fdf41fe4d45dc0d4626b2792aafa192c446043/src/contentScripts/views/App.vue#L16-L33

```html:main.vue
<!-- ... -->
  <input
    id="username"
    ref="searchInput"
    v-model="searchWord"
    class="shadow appearance-none border border-gray-400 rounded-5px w-full py-12px px-12px text-gray-700 leading-tight focus:outline-none focus:shadow-outline box-border bg-white pl-43px text-16px dark:bg-gray-700 dark:text-gray-300 dark:focus:shadow-none dark:border-0"
    type="search"
    placeholder="Search for.."
    @keydown.stop.exact
    @keypress.stop.exact
    @keyup.stop.exact
    @keypress.ctrl.enter.exact.prevent="onEnterWithControl"
    @keydown.down.prevent="onArrowDown"
    @keydown.up.prevent="onArrowUp"
    @keypress.enter.exact.prevent="onEnter"
    @keydown.ctrl.n.prevent="onArrowDown"
    @keydown.ctrl.p.prevent="onArrowUp"
    @keydown.esc.prevent="onCloseModal"
  >
<!-- ... -->
```

## viteese-webextでの爆速開発
Chrome Extensions の開発は、環境構築が面倒なのですが、今回は viteese-webext というボイラープレートを使いました。Vite、TypeScript、WindiCSS などの環境が整えられた状態で開発をスタートできます。そのおかげで、すぐ主要な機能の開発に着手できました。開発効率にかなり影響したと思います。

https://github.com/antfu/vitesse-webext

# 仕組み
Chrome の [History API](https://developer.chrome.com/docs/extensions/reference/history/)、[Bookmark API](https://developer.chrome.com/docs/extensions/reference/bookmarks/)、[Tab API](https://developer.chrome.com/docs/extensions/reference/tabs/) を使いデータを集計。集計結果を content_script.js にて読み込み Vue.js で描画するというシンプルな構造です。サーバーのやり取りは存在せず、すべてブラウザの JS で完結しているため履歴の漏洩などのリスクはありません。

onCommand のリスナーで各種 API を呼び出しています。
https://github.com/kawamataryo/history-search/blob/72fdf41fe4d45dc0d4626b2792aafa192c446043/src/background/main.ts#L67-L95

```ts:background/main.ts
// ...

// コマンドでの起動スクリプト
browser.commands.onCommand.addListener(async() => {
  const tabs = await browser.tabs.query({})
  const bookmarks = await browser.bookmarks.getTree()
  const histories = await browser.history.search({
    text: '',
    maxResults: 10000,
    // Search up to 30 days in advance.
    startTime: new Date().setDate(new Date().getDate() - 30),
  })
  const [tab] = await browser.tabs.query({
    active: true,
    currentWindow: true,
  })

  await sendMessage(
    'history-search',
    {
      result: JSON.stringify(removeDeprecatedItem([
        ...convertToSearchItemsFromHistories(histories),
        ...convertToSearchItemsFromBookmarks(bookmarks),
        ...convertToSearchItemsFromTabs(tabs),
      ])),
    },
    {
      context: 'content-script',
      tabId: tab.id!,
    },
  )
})
// ...
```

そして、background.js からのメッセージ経由で受け取った値は、簡易的な Store を利用して Vue コンポーネントに渡しています。

https://github.com/kawamataryo/history-search/blob/72fdf41fe4d45dc0d4626b2792aafa192c446043/src/contentScripts/index.ts#L10-L17

```ts:contentScripts/index.ts
(() => {
  // initialise store
  const store = useStore()

  onMessage('history-search', ({ data }) => {
    store.changeSearchWord('')
    store.changeHistories(JSON.parse(data.result!))

    store.toggleModal()
  })

  // ...
})
```

コンポーネントでは store から取得したデータをもとに Fuse.js を実行して検索結果を作っています。
https://github.com/kawamataryo/history-search/blob/72fdf41fe4d45dc0d4626b2792aafa192c446043/src/contentScripts/views/App.vue#L154-L167

```ts:contentScripts/views/App.vue
// ...
const searchResult = computed(() => {
  if (!searchItems.value) return []
  const fuse = new Fuse(searchItems.value, {
    keys: [
      'title',
      'url',
    ],
    threshold: FUSE_THRESHOLD_VALUE,
  })
  return fuse.search(searchWord.value, { limit: 10 }).map(result => result.item)
})
// ...
```
# 終わりに

正月の勢いで実装した割には便利な拡張機能が作れたかなと思っています。
多くの人に使ってもらえると嬉しいです。また、何か不具合あればお気軽に Twitter や GitHub Issue でご連絡ください。
コントリビューションも大歓迎です！

https://twitter.com/KawamataRyo
https://github.com/kawamataryo/chikamichi/issues