---
title: "[Vue] ctrl + EnterでのkeypressイベントがWindowsの場合のみ発火しない"
emoji: "⌨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue"]
published: false
---

地味に詰まったのでメモ

# 起こったこと
Windowsの場合のみ`ctrl + enter`でのkeypressイベントが発火しないという事象にあたりました。

サンプルコード

```

```

# 回避策
ワークアラウンド的ですが、keypress単体ならイベントが発火することがわかったので、一旦以下で対応しました。


もし他に良い案あればお願いします。