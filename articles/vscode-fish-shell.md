---
title: "VSCode のターミナルでデフォルト shell を fish に変更する"
emoji: "🐡"
type: "tech"
topics: ["fish"]
published: true
---

fish にする方法で詰まっていたのでメモ。

- VSCode Version: 1.68.1
- OS: Darwin arm64 21.4.0

## 誤った方法

大抵の記事でデフォルト shell の変更は、VSCode の setting.json`terminal.integrated.profiles.osx`に使いたいシェル名を設定するとあります。

```json:settings.json
{
  "terminal.integrated.defaultProfile.osx": "fish",
}
```

これだけだと fish シェルにはなりません。

## 正しい方法

fish にするには、`terminal.integrated.profiles.osx`を拡張して`fish`の profile を設定する必要があります。その上で、`terminal.integrated.defaultProfile.osx`に`fish`を設定します。

※ `args`に追加している`-l`はログインシェルで起動するオプションです。

```json:settings.json
{
  "terminal.integrated.profiles.osx": {
    "fish": {
        "path": "/opt/homebrew/bin/fish",
        "args": [
            "-l"
        ]
    }
  },
  "terminal.integrated.defaultProfile.osx": "fish",
}
```
