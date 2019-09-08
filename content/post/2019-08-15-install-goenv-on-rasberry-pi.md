---
title: "Raspberry PiにgoenvでGoの環境構築をした"
date: 2019-08-15T22:09:21+09:00
draft: false
tags: ["作業ログ", "Raspberry Pi", "Go"]
---

# Goに触りたい, だけどMacがない
Go(golang)を勉強したい.  
しかし先日愛用していたMacが[バッテリー自主回収プログラム](https://support.apple.com/ja-jp/15-inch-macbook-pro-battery-recall)に旅立ってしまったので当分帰ってこない...  
仕方がないので, だいぶ前に買って家に転がっていたラズパイを使ってGoを動かせるようにしてみた.  
需要は無いと思うが一応作業ログとして残しておく.  


<!--more-->
---
## なにができるようになるか
Raspberry Pi上で任意のversionのGoが動くようになる.

## なんでラズパイ?
Macが帰ってくるまで待つのも退屈だから.  

## なんでgoenv?
今日会社で教わったから.  
Goのバージョン管理ができるらしくて便利そうだから.  
ラズパイには少し重そうなので若干不安ではある.

## つかうもの
- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
    - shellはzsh

## いれるもの
- goenv (2.0.0beta11)
    - [repo](https://github.com/syndbg/goenv)

## 参考にしたもの
- [goenv Installation](https://github.com/syndbg/goenv/blob/master/INSTALL.md)
    - ほとんどこの通りにやっただけ

## やること
- goenvのインストール
- Goのインストール
- Hello World

### goenvのインストール
まずはgoenvのrepoを持ってくる.  
HOME直下に`.goenv`が作成され, その中にファイルが入ってくる.  

```
$ git clone https://github.com/syndbg/goenv.git ~/.goenv
```

次にgoenvを使うために必要な環境変数を設定ファイルに追加していく.  
今回はzshを使っているので`.zshenv`に書いていく.  
```
$ echo 'export GOENV_ROOT="$HOME/.goenv"' >> ~/.zshenv
$ echo 'export PATH="$GOENV_ROOT/bin:$PATH"' >> ~/.zshenv
$ echo 'eval "$(goenv init -)"' >> ~/.zshenv
$ echo 'export PATH="$GOPATH/bin:$PATH"' >> ~/.zshenv
$ exec $SHELL
```

`exec $SHELL`が問題なく動けばgoenvを使う準備は完了.  

<details><summary>`.zshenv`</summary><div>

```
#
# Defines environment variables.
#
...
# goenv
export GOPATH="$HOME/go"
export GOENV_ROOT="$HOME/.goenv"
export PATH="$GOENV_ROOT/bin:$PATH"
eval "$(goenv init -)"
export PATH="$GOPATH/bin:$PATH"
```
</div></details>

### Goのインストール
さっそくgoenvをつかって使用可能なGoのバージョン一覧を取得し, 最新の`1.12.7`をインストールした.  

```
$ goenv install -l

> Available versions:
>   1.2.2
>   1.3.0
>   1.3.1
>   ...
>   1.12.7 # これを入れる
>   1.13beta1

$ goenv install 1.12.7
$ goenv versions

> 1.12.7

$ goenv global 1.12.7 # すべての場所でGo 1.12.7を使うよう設定
$ go version

> go version go1.12.7 linux/arm
```

### Hello World
本当にGoが動くかどうか検証する.  

```
$ mkdir -p $GOPATH/bin/hello-world
$ cd $GOPATH/bin/hello-world
$ vim hello.go
$ go run hello.go

>Hello, World!
>from go(goenv) on Raspberry Pi
```

できた.  

<details><summary>`hello.go`</summary><div>

```
package main

import (
    "fmt"
)

func main() {
    fmt.Println("Hello, World!")
    fmt.Println("from go(goenv) on Raspberry Pi")
}
```
</div></details>

## おまけ
作業中ずっと邪魔してきたねこ  
![邪魔するそとちゃん](/images/2019-08-15-sotochan.jpg)  

次はねこについての記事を書きたい.
