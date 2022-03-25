---
template: post
title: rubocop-railsを使用したrailsプロジェクトでneovimのdiagnosticsが動作しない
slug: vim/rubocop-rails-diagnostics
socialImage: /media/printing-press.jpg
draft: false
date: 2022-03-25T04:44:55.443Z
description: rubocop-railsやrubocop-rspecのGemを使用したプロジェクトでneovimのdiagnosticsが表示されなくなったのでその解消方法の備忘録
category: neovim
tags:
  - neovim
---
## 結論

rubocop-railsやrubocop-rspecをローカルにインストールしよう。 以上。

- - -

1. 普段はこんな感じでDiagnosticsが表示されているのだが、仕事で使うrailsプロジェクトである日突然diagnosticsが表示されなくなった。

![neovim](/media/スクリーンショット-2022-03-25-14.20.20.png "neovim")

2. commitログを調べていくと、rubocop-railsやrubocop-rspecが追加されてから挙動がおかしくなったことがわかった
3. ただ調べても解決策には辿り着けなかった

* このへんのissueもあったけどちょっとちがった
* https://github.com/LunarVim/LunarVim/issues/945

4. `.cache/nvim/lsp.log`を確認
5. 「ruboco-railsがないよー」って記述を発見
6. `$ gem install rubocop-rails`をPCにインストールしたら解決