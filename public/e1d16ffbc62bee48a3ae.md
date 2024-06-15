---
title: VSCodeでJavaの開発環境をサクッと構築する
tags:
  - Java
  - 環境構築
  - 新人教育
  - VSCode
private: false
updated_at: '2024-01-09T14:49:30+09:00'
id: e1d16ffbc62bee48a3ae
organization_url_name: null
slide: false
ignorePublish: false
---
:::note info
本記事は新人向け社内研修資料の一部として作成したものです。
一部冗長な説明をしている箇所がありますがあしからず。
:::

## はじめに
javaの開発には、**テキストエディタ**と**javaのコンパイラ**と**javaの実行環境**が必要です。

どういうことかというと、例えばあなたが知らない国の舞台の脚本を任されたとします。そのとき、まずはあなたは自国語で台本を作成する必要がありますね。そして台本を作成したら、それを通訳に翻訳してもらって翻訳版の台本ができあがります。そして翻訳版の台本を知らない国の劇団に渡せば、知らない国で舞台が行われます。[^1]
上記の
- 自国語で台本を作成
- 通訳に翻訳してもらう
- 翻訳された台本をもとに舞台が行われる

がそれぞれ

- テキストエディタでソースコード作成
- コンパイラでソースコードをコンパイル
- コンパイルされたファイルを実行環境で実行

にあたります。

軽くそれぞれについて説明をすると、

テキストエディタはjavaのソースコードを作成するのに使います。
windowsユーザはメモ帳、macユーザはテキストエディットを思い浮かべていただくとわかりやすいと思います。
今回はおそらくエンジニア界隈でトップシェアを誇るであろうVSCodeというエディタを使います。

そしてjavaのコンパイラとjavaの実行環境として今回はopenJDKというオープンソースを使います。openJDKをインストールすればコンパイラと実行環境がついてくるのでopenJDKだけインストールすればokです。

## やること
1. インストーラの準備
2.  vscのインストール
3.  openJDKのインストール

## for windows
### 1. インストーラの準備
今回はwindowsで最近主流な[scoop](https://scoop.sh/)を使います。
powerShellを開いて以下を実行します。
```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser # Optional: Needed to run a remote script the first time
irm get.scoop.sh | iex
```

### 2. vscのインストール
scoopを使ってvscをインストールします。
```powershell
scoop install git
scoop bucket add extras
scoop install vscode
```

### 3. openJDKのインストール
scoopを使ってopenJDKをインストールします。
```powershell
scoop bucket add java
scoop install openjdk
```

## for mac
### 1. インストーラの準備
今回はmacのインストーラといえばの[homebrew](https://brew.sh/ja/)を使います。
ターミナルを立ち上げて以下をコピペして実行します。
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. vscのインストール
homebrewを使ってvscをインストールします。
```bash
brew install --cask visual-studio-code
```

### 3. openJDKのインストール
homebrewを使ってopenJDKをインストールします。
```bash
brew install openjdk
```

※macOS>=10.15の場合はinstall直後に以下を実行してください。
```bash
sudo ln -sfn $(brew --prefix)/opt/openjdk/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk.jdk
```

 ## 動作確認
 1. VSCodeを立ち上げます。
 ![スクリーンショット 2023-12-01 9.20.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/97c4d5e5-b4ee-2dac-9370-0b88a2d24168.png)

2. 左上のメニューから「Terminal」->「New Terminal」を選択肢します。
![スクリーンショット 2023-12-01 9.21.35.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/b924ba4b-f391-d209-2049-e26c435e807a.png)

3. 画面下部にターミナルが開くので「javac --version」と実行します。
画像のようになればOK(細かい数字が違っててもNPです)
![スクリーンショット 2023-12-01 9.23.47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/77eef68b-f84b-aab8-b6fb-d722b786b15b.png)

4. ターミナルで「java --version」と実行します。
画像のようになればOK(細かい数字が違っててもNPです)
![スクリーンショット 2023-12-01 9.25.22.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/2947418/9340db34-dd39-68bf-962d-4150a22186d3.png)

[^1]:こんな簡単に舞台できるわけないです。
