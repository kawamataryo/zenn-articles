---
title: "VanJS で素のDOM操作をリファクタしてみた"
emoji: "🍦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vanjs", "javascript", "typescript"]
published: true
---

VanJSを試してみたら開発体験が良かったので紹介します。

# 🍦 VanJSとは？

[VanJS](https://vanjs.org/)は、数ヶ月前にメジャーバージョンがリリースされた比較的新しいReactive UIフレームワークです。軽量、非依存、トランスパイル不要、シンプルなAPIという特徴があります。

https://vanjs.org/

gzip圧縮後で**0.9kbと非常に軽量**で、バンドルサイズの肥大化を気にすることなく手軽に導入できます。

![](/images/1ad6e51eed13ae/2023-08-17-07-34-28.png)
*他のUIフレームワークと比較しても圧倒的に軽量*

UIもJSXを使わず、関数ベースのAPIで宣言的に構築できます。

@[jsfiddle](https://jsfiddle.net/ryokawamata0425/fh7aLkcp/)

VanJSの開発秘話はとても考えさせられるものだったので、機会があればぜひ読んでみてください。
https://vanjs.org/about


# 🛠️ リファクタリング対象

[Sky Follower Bridge](https://chrome.google.com/webstore/detail/sky-follower-bridge/behhbpbpmailcnfbjagknjngnfdojpko)というX(Twitter)のFollower一覧から[Bluesky](https://bsky.app/)のユーザーを検索するChrome拡張を個人開発しています。今回はその拡張機能のcontent scriptの部分をリファクタリング対象としました。

https://www.youtube.com/watch?v=XcRcWjStIMc

https://github.com/kawamataryo/sky-follower-bridge

https://www.sky-follower-bridge.dev/

この拡張機能はX(Twitter)のDOM要素を基準に動的にBlueskyのユーザーセルを追加するのですが、対象要素にReactやVueのインスタンスを都度マウントするのも何か違和感があるなと、開発当初は素のDOM操作で実装していました。

![](/images/1ad6e51eed13ae/2023-08-17-08-54-41.png)

しかし、機能が増えるにつれコードが複雑化し可読性に問題が出てきました。そこで、軽量でシンプルなVanJSを使いリファクタリングしてみることにしました。


# 📝 結果

Blueskyのユーザーセルの挿入箇所を例にリファクタリング結果を紹介します。
実際のVanJSの導入PRはこちらです。

https://github.com/kawamataryo/sky-follower-bridge/pull/7

## Before

リファクタリング前のコードは以下のようになっていました。

```ts:src/lib/domHelpers.ts
export const insertBskyProfileEl = ({ dom, profile, statusKey, btnLabel, abortController, followAction, unfollowAction }: {
  dom: Element,
  profile: ProfileView,
  statusKey: keyof ViewerState,
  btnLabel: UserCellBtnLabel,
  abortController: AbortController,
  followAction: () => void,
  unfollowAction: () => void
}) => {
  const avatarEl = profile.avatar ? `<img src="${profile.avatar}" width="48" />` : "<div class='no-avatar'></div>"
  const actionBtnEl = profile.viewer[statusKey] ? `<button class='follow-button follow-button__following'>${btnLabel.progressive} on Bluesky</button>` : `<button class='follow-button'>${btnLabel.add} on Bluesky</button>`
  dom.insertAdjacentHTML('afterend', `
  <div class="bsky-user-content">
    <div class="icon-section">
      <a href="https://bsky.app/profile/${profile.handle}" target="_blank" rel="noopener">
        ${avatarEl}
      </a>
    </div>
    <div class="content">
      <div class="name-and-controller">
        <div>
          <p class="display-name"><a href="https://bsky.app/profile/${profile.handle}" target="_blank" rel="noopener">${profile.displayName ?? profile.handle}</a></p>
          <p class="handle">@${profile.handle}</p>
        </div>
        <div>
          ${actionBtnEl}
        </div>
      </div>
      ${profile.description ? `<p class="description">${profile.description}</p>` : ""}
    </div>
  </div>
  `)
  const bskyUserContentDom = dom.nextElementSibling as Element

  // register a click action
  bskyUserContentDom?.addEventListener('click', async (e) => {
    const target = e.target as Element
    const classList = target.classList

    // follow action
    if (classList.contains('follow-button') && !classList.contains('follow-button__following')) {
      target.textContent = "processing..."
      target.classList.add('follow-button__processing')
      await followAction()
      target.textContent = `${btnLabel.progressive} on Bluesky`
      target.classList.remove('follow-button__processing')
      target.classList.add('follow-button__following')
      target.classList.add('follow-button__just-followed')
      return
    }

    // unfollow action
    if (classList.contains('follow-button') && classList.contains('follow-button__following')) {
      target.textContent = "processing..."
      target.classList.add('follow-button__processing')
      await unfollowAction()
      target.textContent = `${btnLabel.add} on Bluesky`
      target.classList.remove('follow-button__processing')
      target.classList.remove('follow-button__following')
      return
    }
  }, {
    signal: abortController.signal
  })

  bskyUserContentDom?.addEventListener('mouseover', async (e) => {
    const target = e.target as Element
    const classList = target.classList
    if (classList.contains('follow-button') && classList.contains('follow-button__following')) {
      target.textContent = `${btnLabel.remove} on Bluesky`
    }
  }, {
    signal: abortController.signal
  })
  bskyUserContentDom?.addEventListener('mouseout', async (e) => {
    const target = e.target as Element
    const classList = target.classList
    if (classList.contains('follow-button__just-followed')) {
      target.classList.remove('follow-button__just-followed')
    }
    if (classList.contains('follow-button') && classList.contains('follow-button__following')) {
      target.textContent = `${btnLabel.progressive} on Bluesky`
    }
  }, {
    signal: abortController.signal
  })
}
```

`insertAdjacentHTML`で条件に合わせたDOMを構築、`addEventListener`でclickイベントと、hoverイベントを登録しています。

Reactiveな動作は主にボタンに集中しているのですが、このボタンの挙動を実現するためにDOM APIで直接クラスの付け変えやテキストの書き換えを行なっています。UIとイベントの関連が分かりにくく、書いた本人ながらあまり触りたくないコードです😅

![](/images/1ad6e51eed13ae/button.gif)



## After

VanJSでのリファクタリング後です。

・Mount

```ts:src/lib/domHelpers.ts
export const insertBskyProfileEl = ({ dom, profile, statusKey, btnLabel, addAction, removeAction }: {
  dom: Element,
  profile: ProfileView,
  statusKey: keyof ViewerState,
  btnLabel: UserCellBtnLabel,
  addAction: () => Promise<void>,
  removeAction: () => Promise<void>
}) => {
  van.add(dom.parentElement, BskyUserCell({
    profile,
    statusKey,
    btnLabel,
    addAction,
    removeAction,
  }))
}
```

・ActionButton

```ts:src/components/BskyUserCell.ts
const ActionButton = ({ statusKey, profile, btnLabel, addAction, removeAction }: {
  profile: ProfileView,
  statusKey: keyof ViewerState,
  btnLabel: UserCellBtnLabel,
  addAction: () => Promise<void>,
  removeAction: () => Promise<void>
}) => {
  const label = van.state(`${profile.viewer[statusKey] ? btnLabel.progressive : btnLabel.add} on Bluesky`)

  const isStateOfBeing = van.state(profile.viewer[statusKey])
  const isProcessing = van.state(false)
  const isJustApplied = van.state(false)

  const beingClass = van.derive(() => isStateOfBeing.val ? "action-button__being" : "")
  const processingClass = van.derive(() => isProcessing.val ? "action-button__processing" : "")
  const justAppliedClass = van.derive(() => isJustApplied.val ? "action-button__just-applied" : "")

  const onClick = async () => {
    if (isProcessing.val) return
    isProcessing.val = true
    label.val = "Processing..."

    if (isStateOfBeing.val) {
      await removeAction()
      label.val = `${btnLabel.add} on Bluesky`
      isStateOfBeing.val = false
    } else {
      await addAction()
      label.val = `${btnLabel.progressive} on Bluesky`
      isStateOfBeing.val = true
      isJustApplied.val = true
    }

    isProcessing.val = false
  }

  const onMouseover = () => {
    if(
      isProcessing.val ||
      isJustApplied.val ||
      !isStateOfBeing.val
    ) return

    label.val = `${btnLabel.remove} on Bluesky`
  }

  const onMouseout = () => {
    if (isJustApplied.val) {
      isJustApplied.val = false
    }
    if(!isStateOfBeing.val) return

    label.val = `${btnLabel.progressive} on Bluesky`
  }

  return button({
    class: () => `action-button ${beingClass.val} ${processingClass.val} ${justAppliedClass.val}`,
    onclick: onClick,
    onmouseover: onMouseover,
    onmouseout: onMouseout,
  },
    () => label.val,
  )
}
```

・BskyUserCell

```ts:src/components/BskyUserCell.ts
const Avatar = ({ avatar }: { avatar?: string }) => {
  return avatar ? img({ src: avatar, width: "40" }) : div({ class: "no-avatar" })
}

export const BskyUserCell = ({
  profile,
  statusKey,
  btnLabel,
  addAction,
  removeAction,
}: {
  profile: ProfileView,
  statusKey: keyof ViewerState,
  btnLabel: UserCellBtnLabel,
  addAction: () => Promise<void>,
  removeAction: () => Promise<void>
}) => {
  return div({ class: "bsky-user-content" },
    div({ class: "icon-section"},
      a({ href: `https://bsky.app/profile/${profile.handle}`, target: "_blank", rel: "noopener" },
        Avatar({ avatar: profile.avatar }),
      ),
    ),
    div({ class: "content" },
      div({ class: "name-and-controller" },
        div(
          p({ class: "display-name" },
            a({ href: `https://bsky.app/profile/${profile.handle}`, target: "_blank", rel: "noopener" },
              profile.displayName ?? profile.handle,
            ),
          ),
          p({ class: "handle" },
            `@${profile.handle}`,
          ),
        ),
        div(
          ActionButton({
            profile,
            statusKey,
            btnLabel,
            addAction,
            removeAction,
          })
        ),
      ),
      profile.description ? p({ class: "description" }, profile.description) : "",
    ),
  )
}
```

マウント処理は、対象のDOMに`van.add`でコンポーネントをマウントするのみ。VanJSのコンポーネント側も、親しみあるVueやReactのようにコンポーネントを分割し宣言的にUIを構築できました。

Reactiveな挙動を持つボタンも、`van.state`で状態を管理することでイベントとUIの関連が一目でわかり、だいぶ可読性は良くなった気がします。

今後の機能拡張のモチベーションも上がりました💪

# 👋 おわりに
VanJSは `VanJS is the scripting language for UI, just like bash is the scripting language for terminal.`  という説明の通り、手軽にReactiveなUIが構築できて最高でした。

今回の例のようなChrome Extensionのcontent scriptや、静的サイトのちょっとしたUIの実装など、ピンポイントでReactiveな動きを実現したい時には、とても良い選択肢になると思います。ぜひ使って見てください。
