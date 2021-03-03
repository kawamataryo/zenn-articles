---
title: "NeovimのTerminalモードをちょっと使いやすくする"
emoji: "🏴"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vim", "neovim", "terminal"]
published: true
---

Neovim の Terminal モードをちょと便利にする設定をみつけたのでメモ。

# Terminalのインサートモードからの離脱を`esc`キーにマッピング

Terminal のインサートモードを抜けるコマンド（`<C-\><C-n>`）を毎回忘れるので、いつもの`esc`にマッピングする。

```vim:.config/nvim/init.vim
:tnoremap <Esc> <C-\><C-n>
```

# TerminalをVSCodeのように現在のウィンドウの下に開く

普通に`:term`コマンドで Terminal を開くと現在のウィンドウで開かれるが、前の画面に戻る際、 buffer の切り替えが必要なので微妙に不便。
`:T`コマンドで Terminal を開くと下部別ウィンドウで表示されるようにする。

```vim:config/nvim/init.vim
command! -nargs=* T split | wincmd j | resize 20 | terminal <args>
```

# 常にインサートモードでTerminalを開く

Terminal を開くときは、Terminal を使いたいときなので、いちいちノーマルモードからインサートモードに変更するのが面倒。
Terminal を開くとデフォルトでインサートモードにする。

```vim:config/nvim/init.vim
autocmd TermOpen * startinsert
```