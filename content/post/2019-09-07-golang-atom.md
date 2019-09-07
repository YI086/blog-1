---
title: "Go向けにAtomの設定をした"
date: 2019-09-07T11:47:25+09:00
draft: true
tags: ["作業ログ", "Go", "Atom"]
---

## AtomってIDEじゃなくね?  
最近Goに触りだしたので, 普段使ってるエディタ([Atom](https://atom.io/))をGo向けにセットアップしてみた.  

<!--more-->
---

## やったこと
AtomをGoのIDE**っぽく**する.  
IDEにするって言い切ると詳しい人に怒られそう.  

## つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- Atom
    - 1.40.1
    - [インストール手順](https://github.com/uzimihsr/setup/blob/master/atom.md)
- Go
    - go version go1.13 darwin/amd64
    - [インストール手順](https://github.com/uzimihsr/setup/blob/master/go.md)

## 手順
- atom-ide-uiのインストール
- scriptのインストール
- go-plusのインストール
- go-debugのインストール
- go-signature-statusbarのインストール
- 動作確認

### atom-ide-uiのインストール
[atom-ide-ui](https://atom.io/packages/atom-ide-ui)  
AtomをIDEっぽくしてくれるパッケージ.  

### scriptのインストール
[script](https://atom.io/packages/script)  
Atom上でコードを実行してくれるパッケージ.  

### go-plusのインストール
[go-plus](https://atom.io/packages/go-plus)  
**本命**. AtomでGoを書くときのサポートをだいたいやってくれる.  

### go-debugのインストール
[go-debug](https://atom.io/packages/go-debug)  
go-plusで使用する. Goのデバッガ.  

### go-signature-statusbarのインストール
[go-signature-statusbar](https://atom.io/packages/go-signature-statusbar)
go-plusで使用する. 画面下のステータスバーにいい感じの情報を載せてくれる.  

### 動作確認
