---
title: "わりと使うけど忘れがちなGitコマンドメモ"
emoji: "✍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "git"]
published: false
---

# rootコミットからのrebase

通常の rebase だと、2 つめのコミットからしか変更できない。`--root`オプションをつけることで最初のコミットから変更できる。

```bash
git rebase -i --root
```

# 一度コミット・プッシュしてしまった機密ファイルを後から完全に削除する

`git rm --cached`や`git filter-branch`を使う方法が紹介されてることが多いですが、[bfc](https://rtyley.github.io/bfg-repo-cleaner/)を使った方がより簡単に確実にできます。

bfgのインストール

```bash
brew install bfg
```

ファイル削除の実行

```bash
bfg --delete-files ファイル名
```


# 参考
- [機密データをリポジトリから削除する - GitHub Docs](https://docs.github.com/ja/enterprise-server@2.19/github/authenticating-to-github/removing-sensitive-data-from-a-repository)