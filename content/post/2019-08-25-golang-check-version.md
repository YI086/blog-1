---
title: "コード中でGoのバージョンを確認する"
date: 2019-08-25T22:13:29+09:00
draft: false
tags: ["Go", "メモ"]
---

## go version は使えるけども
pythonの`sys.version`みたいに`Go(golang)`でもコード中でバージョンを確認する方法があったのでメモ.  
<!--more-->
---
## なにができるようになるか
`Go`のソースコード内でバージョン情報をstringで扱えるようになる.  

## やりかた  

[runtime.Version()](https://godoc.org/runtime#Version)を使えば良い. かんたん.

試しに`goenv`でGoのバージョンを変えながらコマンドライン上でバージョンを確認する`go version`とコード中でバージョンを確認する`check-version.go`を交互に動かしてみる.  

<details><summary>`check-version.go`</summary><div>

```
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("Go version :", runtime.Version())
}
```
</div></details>
<br>

```
$ goenv versions

>1.11.13
>* 1.12.9 (set by /Users/username/.goenv/version)

$ goenv global 1.12.9 # すでに設定されてるので意味無し
$ go version

>go version go1.12.9 darwin/amd64

$ go run check-version.go

>Go version : go1.12.9

$ goenv global 1.11.13
$ go version

>go version go1.11.13 darwin/amd64

$ go run check-version.go

>Go version : go1.11.13
```

できた. サンプルコードで実行環境吐き出させるときとかにつかいたい.  

## おまけ
パソコンばっかいじってる飼い主に愛想をつかしておしりを向けて寝るねこ  
![おしりを向けて寝るそとちゃん](/images/2019-08-25-sotochan-omake.jpg)  
