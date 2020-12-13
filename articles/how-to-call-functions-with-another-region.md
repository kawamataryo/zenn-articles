---
title: "[Firebase] us-central1以外にデプロイしたhttpsCallable関数を呼び出す際には.."
emoji: "🤙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase", "gcp"]
published: true
---

ちょっと詰まったのでメモ。

# 別リージョンにデプロイしたhttpsCallable関数の呼び出しに失敗する？

以下のように`asia-northeast1`に httpsCallable 関数をデプロイしてアプリ側から呼ぶコードを書きました。

◆function 側

```js
export const foo = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    // ...
  })
```

◆アプリ側

```js
const functions = firebase.functions();

// fooはasia-notheast1のregionのhttpsCallable関数で呼び出される
const foo = functions.httpsCallable("foo");
foo({ /* ... */ }).then(/* ... */)
```

特になにも問題はないように見えるのですが、アプリ側から Cloud Functions for Firebase の httpsCallable 関数を呼び出したとき以下エラーで失敗してしまいました。

```
Access to fetch at 'https://us-central1-trigger-email-sample.cloudfunctions.net/foo' from origin 'http://localhost:8080' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: Redirect is not allowed for a preflight request.
```

一見 CORS っぽいエラーなのですが、よく見ると関数の実行先が`https://us-central1-xxx`となっています。
これでは region が違うので関数が見つかりません。

ドキュメントにも以下の記載がありました。どうやら`us-central1`以外にデプロイしている場合はアプリ側で region の設定が必要なようです。

> Note: To call a function running in any location other than the default us-central1, you must set the appropriate value at initialization. For example, on Android you would initialize with getInstance(FirebaseApp app, String region).

https://firebase.google.com/docs/functions/callable


# 解決策

以下のように firebase 初期化時に region をつけることで解決しました。
通常のように`firebase.functions()`ではなく`firebase.app().functions()`とする必要があるので注意です。

```js
const functions = firebase.app().functions('asia-northeast1');

// fooはasia-notheast1のregionのhttpsCallable関数で呼び出される
const foo = functions.httpsCallable("foo");
foo({ /* ... */ }).then(/* ... */)
```

# 参考

- [Call functions from your app  |  Firebase](https://firebase.google.com/docs/functions/callable#web_2)
- [firebase - How to use httpsCallable on a region other then us-central1 for web - Stack Overflow](https://stackoverflow.com/questions/57547745/how-to-use-httpscallable-on-a-region-other-then-us-central1-for-web)
