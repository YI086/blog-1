---
title: "Goでコマンドラインサイコロを作った"
date: 2019-09-04T22:15:18+09:00
draft: false
tags: ["作業ログ", "Go"]
---

## なんか作りたい
[A Tour of Go](https://tour.golang.org/list)を一通りなぞったので, なんか作りたくなった.  

<!--more-->
---

## やったこと
Goの練習として, コマンドラインで動くかんたんなサイコロを作ってみた.  

動作のイメージ :
```
$ dice # 1~6の範囲でランダムに生成された整数を1つ出力
3

$ dice -f 10 # -fオプションでサイコロの最大値を変更
9

$ dice -d 2 # -dオプションで振るサイコロの数を変更
5 4

$ dice -f 10 -d 2 # -fオプションと-dオプションの併用も可能
6 9
```

## つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- Go
    - go version go1.12.9 darwin/amd64
    - [インストール手順](https://github.com/uzimihsr/setup/blob/master/go.md)

## 手順
- dice.goの作成
- dice.goをインストール

### dice.goの作成
とりあえず標準パッケージを使って作ってみた.  

```
$ mkdir -p $GOPATH/src/github.com/uzimihsr/dice
$ cd $GOPATH/src/github.com/uzimihsr/dice
$ vim dice.go
```

<details><summary>`dice.go`</summary><div>

```
package main

import (
	"flag"
	"fmt"
	"math/rand"
	"time"
)

func main() {
	// コマンドラインオプションで与える値の変数定義
	var (
		faces uint
		dices uint
	)

	// コマンドラインオプションの設定
	flag.UintVar(&faces, "f", 6, "The number of dice faces")
	flag.UintVar(&dices, "d", 1, "The number of dices to throw")
	flag.Parse()

	// サイコロを振り, 出目を出力
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < int(dices); i++ {
		fmt.Printf("%d ", (rand.Intn(int(faces)) + 1))
	}
	fmt.Println()
}
```
</div></details>
<br>

Goでコマンドラインツールを作る場合には[urfave/cli](https://github.com/urfave/cli)とか[cobra](https://github.com/spf13/cobra)が良いらしいんだけど,  
今回はそこまで高機能なものはつくらないので標準パッケージ[flag](https://godoc.org/flag)を使ってオプションの処理をした.  

サイコロの処理(乱数の生成)にはこちらも標準の[math/rand](https://godoc.org/math/rand)パッケージを使った.  
そのまま使うと乱数のSeedが1で固定され, 毎回同じ値が生成されてしまうので[time](https://godoc.org/time)パッケージで呼び出したUnix時間を乱数Seedとして使うようにした.  

### dice.goをインストール
`go install`を使ってdice.goの実行ファイルを`$GOPATH/bin`にインストールする.  

```
$ cd $GOPATH/src/github.com/uzimihsr/dice
$ go install
$ which dice
{$GOPATH}/bin/dice

$ dice -f 12 -d 6
1 6 8 9 11 3
```

いい感じ.  

つくったやつはここ.  

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:500px;" title="uzimihsr/dice" src="https://hatenablog-parts.com/embed?url=https://github.com/uzimihsr/dice" width="300" height="150" frameborder="0" scrolling="no"></iframe>

## おまけ
スマホをけとばすねこ with エリザベスカラー  
![スマホをけとばすそとちゃん](/images/2019-09-04-sotochan-omake.jpg)
