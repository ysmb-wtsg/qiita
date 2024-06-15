---
title: Neovimでソースコードレビュー！！！
tags:
  - レビュー
  - プラグイン
  - ソースコード
  - neovim
private: false
updated_at: '2024-03-20T23:14:55+09:00'
id: ca55bc5a41ece779df7f
organization_url_name: null
slide: false
ignorePublish: false
---
最近neovimでソースコードレビューの環境が整ってきたので世に広めたいと思います。
というかほぼプラグインの紹介です。

## 目次

[1. プラグインのインストール](#1-プラグインのインストール)
[2. keymapsの設定](#2-keymapsの設定)
[3. 使ってみる](#3-使ってみる)

## 1. プラグインのインストール
[diffview](https://github.com/sindrets/diffview.nvim)というプラグインを使います。
gitの差分表示に特化したプラグインです。デモとか使い方とかはリンク先のREADME見てください。
ちなみにnvimにおいてgitの主力なプラグインとしては[gitsigns](https://github.com/lewis6991/gitsigns.nvim)というプラグインがありますが、差分表示の仕方にあまりしっくりこなかったです。

packerを使っている方は以下でインストールできます。
セットアップのスクリプトも特に不要です。
```plugin-setup.lua
use("sindrets/diffview.nvim")
```

## 2. keymapsの設定
パパッと差分を表示できるようにkeymapも設定しておきます。

```keymaps.lua
keymap.set("n", "<leader>hd", "<cmd>DiffviewOpen HEAD~1<CR>", silent)
keymap.set("n", "<leader>hf", "<cmd>DiffviewFileHistory %<CR>", silent)
```

軽く説明をすると、
`DiffviewOpen HEAD~1`というコマンドは、HEADと、HEADから一個前のコミットの差分を表示するコマンドです。私はgit系のコマンドは`h(unk)`から始めるように統一してるので、`hd`でマップしてます。
`DiffviewFileHistory %`というコマンドは、開いているファイルの変更履歴を表示するコマンドです。

## 3. 使ってみる
今回は私の[nvimのconfigリポジトリ](https://github.com/ysmb-wtsg/nvim)を使ってソースレビューをしてみます。

![スクリーンショット 2024-03-20 23.01.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/b962e419-9c5c-69de-995e-d700ac10a74e.png)

ここで、`<leader>hd`を実行すると、

![スクリーンショット 2024-03-20 23.03.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/9ea1ee4a-7ebf-af36-e493-4f21364df070.png)

こんな感じでGitHubやGitLabで見れるような差分ビューアが表示されます。

ちなみにファイルを選択して`<leader>hf`を実行すると、

![スクリーンショット 2024-03-20 23.06.55.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/10a49da0-9bb7-eef8-4d1c-ea43b35df80c.png)

こんな感じで、ファイルに関連する過去のコミットと、その差分が表示されます。

## ちなみに
GitHubのPRをレビューするためのプラグインもあるのですが(詳しくはググってください)、GitLabなどの他のサービスでもレビューできるようなプラグインとして今回はdiffviewを紹介しました。

それではより良いnvimライフを！！！
