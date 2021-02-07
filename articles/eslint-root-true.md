---
title: "同プロジェクトの複数の階層でeslintrcを持つ場合はroot:trueを設定"
emoji: "⚠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["eslint", "JavaScript"]
published: false
---

エラー対応のメモ。

# 何が起こった？

以下のような構成でアプリを作っていて Cloud Functions for Firebase のディレクトリで ESLint をかけたときに以下エラーが発生しました。

```bash
├── functions
│   ├── .eslintrc.js
│   ...
├── .eslintrc.js
...
```

```
ESLint couldn't determine the plugin "@typescript-eslint" uniquely.
```

# 対応

functions 配下の`.eslintrc.js`に以下を追記すれば OK です。

```js

module.exports = {
  //...
  root: true
}
```