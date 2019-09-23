---
title: "Go向けにAtomの設定をした"
date: 2019-09-07T11:47:25+09:00
draft: false
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

### atom-ide-uiのインストール
[atom-ide-ui](https://atom.io/packages/atom-ide-ui)  
AtomをIDEっぽくしてくれるパッケージ.  
Goの場合はgo-plusとの併用が必要になってくる.  
```
$ apm install atom-ide-ui
```

### scriptのインストール
[script](https://atom.io/packages/script)  
Atom上でコードを実行してくれるパッケージ.  
`⌘+i`で現在開いているコードを実行できる.  
.goファイルの場合は`go run`するのと同じことをしてくれる.  
他の言語で実行オプションを細かく設定したい場合などは`$HOME/.atom/packages/script/lib/grammars`にある設定ファイルを編集すれば良い.  
```
$ apm install script
```

### go-plusのインストール
[go-plus](https://atom.io/packages/go-plus)  
**本命**. AtomでGoを書くときのサポート(整形, 補完, docの簡易表示, 関数定義ファイルへのジャンプ)をだいたいやってくれる.  
Goのパッケージをいくつか必要とするので, それらもインストールする.  
```
$ go get -u golang.org/x/tools/cmd/goimports
$ go get -u golang.org/x/tools/cmd/gorename
$ go get -u github.com/sqs/goreturns
$ go get -u github.com/mdempsky/gocode
$ go get -u github.com/alecthomas/gometalinter
$ go get -u github.com/mgechev/revive
$ go get -u github.com/golangci/golangci-lint/cmd/golangci-lint
$ go get -u github.com/zmb3/gogetdoc
$ go get -u github.com/zmb3/goaddimport
$ go get -u github.com/rogpeppe/godef
$ go get -u golang.org/x/tools/cmd/guru
$ go get -u github.com/fatih/gomodifytags
$ go get -u github.com/tpng/gopkgs
$ go get -u github.com/ramya-rao-a/go-outline
$ apm install go-plus
```

### go-debugのインストール
[go-debug](https://atom.io/packages/go-debug)  
go-plusで使用する. Goのデバッガ.  
```
$ go get -u github.com/go-delve/delve/cmd/dlv
$ apm install go-debug
```

### go-signature-statusbarのインストール
[go-signature-statusbar](https://atom.io/packages/go-signature-statusbar)
go-plusで使用する. 画面下のステータスバーにいい感じの情報を載せてくれる.  
```
$ apm install go-signature-statusbar
```

## つかいかたの確認
現状使いこなせてるのはこの程度.  

|キーバインド|機能|
|---|---|
|`⌘+s`|保存時に自動で`goimports`でコード整形が行われる.|
|`⌘+i`|現在表示しているコードを実行する.|
|`⌘+click`|`godef`で関数定義へジャンプする.|

![screenshot](/images/2019-09-07-screenshot.png)

動的にlintしてくれないのが残念だけど, そもそもGoのコード整形自体が強力なのであまり気にならなさそう.
