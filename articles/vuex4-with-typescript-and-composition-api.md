---
title: "Vuex4 から始める Vuex 〜TSとComposition API を添えて〜"
emoji: "4️⃣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vue", "vuex", "typescript"]
published: false
---

## 疑問

- getters は Composition API でどう使うの？
  - mapGetters は CompositionAPI で使えない。
- Composition API で書いた場合と Optional API で書いた場合の比較
- mutations, actions 違いは何？
  - dispatch と commit は？
  - mutations の実行は commit
  - actions の実行は dispatch

# 参考

- [vuex/examples/composition at 4.0 · vuejs/vuex](https://github.com/vuejs/vuex/tree/4.0/examples/composition)

useXXX を作りたいための Issue

- [Add useXXX helpers · Issue #1725 · vuejs/vuex](https://github.com/vuejs/vuex/issues/1725)

Vuex で compositionAPI を使うためのライブラリ

- [greenpress/vuex-composition-helpers: A util package to use Vuex with Composition API easily.](https://github.com/greenpress/vuex-composition-helpers#typescript-mappings)
