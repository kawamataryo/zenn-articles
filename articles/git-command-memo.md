---
title: "わりと使うけど忘れがちなGitコマンドメモ[随時更新]"
emoji: "✍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub", "git"]
published: false
---

# プルリクエストのブランチへのcheckout

プルリクエストもらったものの「ブランチ名で checkout できない。.?」となったのでメモ。

`ID`はプルリクエストの番号
`BRANCHNAME`はプルリクエストのブランチ名（作成ユーザー名を除く）

```
git fetch origin pull/ID/head:BRANCHNAME
git checkout BRANCHNAME
```

# 一度コミット・プッシュしてしまった機密ファイルを後から完全に削除する

`git rm --cached`や`git filter-branch`を使う方法が紹介されてることが多いですが、[bfc](https://rtyley.github.io/bfg-repo-cleaner/)を使ったほうがより簡単に確実にできます。

bfg のインストール

```bash
brew install bfg
```

ファイル削除の実行

```bash
bfg --delete-files ファイル名
```

# rootコミットからのrebase

通常の rebase だと、2 つめのコミットからしか変更できませんが、`--root`オプションをつけることで最初のコミットから変更できます。

```bash
git rebase -i --root
```

# 参考
- [機密データをリポジトリから削除する - GitHub Docs](https://docs.github.com/ja/enterprise-server@2.19/github/authenticating-to-github/removing-sensitive-data-from-a-repository)
- [Checking out pull requests locally - GitHub Docs](https://docs.github.com/en/free-pro-team@latest/github/collaborating-with-issues-and-pull-requests/checking-out-pull-requests-locally)