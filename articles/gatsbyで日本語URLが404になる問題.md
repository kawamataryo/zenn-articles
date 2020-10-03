---
title: "gatsbyで日本語URLが404になる問題"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

日本語URLがエンコードされたpathになっていた。

対策。デコードして記録する。

```
decodeURI(node.slug),
```