---
title: "cobraでかんたんなCLIツールをつくった"
date: 2020-09-03T22:10:37+09:00
draft: false
tags: ["Go", "作業ログ"]
---

## Goでコマンドラインツール
golangでCLIツールを作るときに引数とかサブコマンドをいい感じにしてくれるライブラリのcobraを試した.  

<!--more-->
---

## やったことのまとめ

`cobra`を使ってコマンドラインであいさつするおもちゃ[greeting](https://github.com/uzimihsr/greeting)をつくった.  

{{< highlight bash >}}
# hello
$ greeting hello hoge fuga piyo --message 'Nice to meet you!'
Hello, hoge, fuga, and piyo!
Nice to meet you!

# こんにちは
$ greeting hello 鈴木 --message 'お会いできて嬉しいです!' -l ja
鈴木さん, こんにちは!
お会いできて嬉しいです!

# goodbye
$ greeting goodbye hoge fuga piyo
Goodbye, hoge, fuga, and piyo!

# さようなら
$ greeting goodbye A -l ja
Aさん, さようなら!
{{< /highlight >}}

## つかうもの
- macOS Mojave 10.14
- [go](https://github.com/golang/go) version go1.14.6 darwin/amd64
- [cobra](https://github.com/spf13/cobra) v1.0.0

## やったこと

- [cobraのインストールとサンプルコードの作成](#cobraのインストールとサンプルコードの作成)
- [サブコマンドの実装](#サブコマンドの実装)

### cobraのインストールとサンプルコードの作成

`cobra`はこれ自身がコマンドとしても提供されているので, まずはこれをインストールする.  
{{< highlight bash >}}
# cobraをインストール
$ go get -u github.com/spf13/cobra/cobra
$ which cobra
/GOPATH/bin/cobra
$ cobra
Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  cobra [command]

Available Commands:
  add         Add a command to a Cobra Application
  help        Help about any command
  init        Initialize a Cobra Application

Flags:
  -a, --author string    author name for copyright attribution (default "YOUR NAME")
      --config string    config file (default is $HOME/.cobra.yaml)
  -h, --help             help for cobra
  -l, --license string   name of license for the project
      --viper            use Viper for configuration (default true)

Use "cobra [command] --help" for more information about a command.
{{< /highlight >}}

`cobra init`を使うと動く状態のサンプルコードを生成してくれる.[^1]  
昔は空のディレクトリじゃないとできなかったみたいだけど, 今は気にしなくて大丈夫.  


{{< highlight bash >}}
# 新規ディレクトリを作成+go.modを作成
$ mkdir -p greeting && cd greeting
$ go mod init github.com/uzimihsr/greeting
go: creating new go.mod: module github.com/uzimihsr/greeting

# サンプルコードの作成
$ cobra init --pkg-name github.com/uzimihsr/greeting
Your Cobra application is ready at
/path/to/greeting
$ tree .
.
├── LICENSE
├── cmd
│   └── root.go
├── go.mod
└── main.go

1 directory, 4 files
{{< /highlight >}}

作成されたサンプルコードはこんな感じ.  

<details><summary>`main.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/447574ddce2b458a9ec0e9bbaced5f78.js"></script>
</div></details>
<details><summary>`cmd/root.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/e5e45083544edb48f9b8fb392a749886.js"></script>
</div></details>

試しに動かしてみる.  

{{< highlight bash >}}
# アプリの実行
$ go run main.go
A longer description that spans multiple lines and likely contains
examples and usage of using your application. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.
# root.goで設定されたヘルプメッセージ
{{< /highlight >}}

ちゃんと動いた.  

さらに, `cobra add`でサブコマンドを作成できる.  
追加したサブコマンドのファイルは`root.go`と同じく`cmd`配下に作成される.  

{{< highlight bash >}}
# サブコマンド(hello)の作成/実行
$ cobra add hello
hello created at /Users/ryota/Workspace/greeting
$ ls cmd
hello.go root.go

$ go run main.go hello
hello called
$ go run main.go hello -h
A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.

Usage:
  greeting hello [flags]

Flags:
  -h, --help   help for hello

Global Flags:
      --config string   config file (default is $HOME/.greeting.yaml)
{{< /highlight >}}

<details><summary>`cmd/hello.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/b1ee2c2e0140aa1fea0e5180374488f2.js"></script>
</div></details>

### サブコマンドの実装

あとはこのルートコマンド(`root.go`)とサブコマンド(`hello.go`)をいじっていく.  
どちらもinit()関数で引数やコマンドに必要な変数の設定を行い,  
Command.Runのfunc()で実際に行う処理を記述しているので, この形式に従っていろいろいじってみる.  


<details><summary>`cmd/root.go(編集後)`</summary><div>
<script src="https://gist.github.com/uzimihsr/b62f55b5e70a1b924c6576cf6f713b4e.js"></script>
</div></details>
<details><summary>`cmd/hello.go(編集後)`</summary><div>
<script src="https://gist.github.com/uzimihsr/0ee18d3e322b19e86277d98bf7fc1441.js"></script>
</div></details>
<details><summary>`cmd/goodbye.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/a0763f5d1295ed88747c09f54da91606.js"></script>
</div></details>

最終的なディレクトリ構造はこんな感じ.  
出会いと別れの挨拶をするサブコマンド`cmd/hello.go`と`cmd/goodbye.go`を実装した.  

{{< highlight bash >}}
$ tree .
.
├── cmd
│   ├── goodbye.go
│   ├── hello.go
│   └── root.go
├── go.mod
├── go.sum
└── main.go

1 directory, 6 files
{{< /highlight >}}

実際に動かしてみる.  
`go install`すると`PATH`が通ってすぐ使えるようになるのでべんり.  

{{< highlight bash >}}
# ビルドして実行
$ go build
$ ./greeting hello World
Hello, World!

# GOPATHが通っている場合はinstallするとパスが通る
$ go install
$ which greeting
$GOPATH/bin/greeting

# -l で言語(en/ja)の指定ができる
$ greeting goodbye A B -l ja
Aさん, Bさん, さようなら!

# --message で追加のメッセージを指定できる
$ greeting hello A B C --message 'Nice to meet you!'
Hello, A, B, and C!
Nice to meet you!

# goodbyeには敢えて --message のflagをつけていないので指定するとエラーになる
$ greeting goodbye A --message 'See you again!'
Error: unknown flag: --message
Usage:
  greeting goodbye [NAME] [flags]

Flags:
  -h, --help   help for goodbye

Global Flags:
  -l, --lang string   Language : en, ja (default "en")

unknown flag: --message

# 引数の数が不正でもエラーになる
$ greeting hello
Error: requires at least 1 arg(s), only received 0
Usage:
  greeting hello [NAME] [flags]

Flags:
  -h, --help             help for hello
  -m, --message string   Help message for toggle

Global Flags:
  -l, --lang string   Language : en, ja (default "en")

requires at least 1 arg(s), only received 0
{{< /highlight >}}

やったぜ.  
機能はしょぼいけど, サブコマンドごとに違う機能を持つCLIツールができた.  

## おわり
`Cobra`を使って簡単なCLIツールを作った.  
サブコマンドごとに`flag`(オプション)が管理できたり, `Args`で引数の条件を指定できたりするのがいいと思う.  
今まで`Go`でCLIを作るときは標準パッケージの[flag](https://godoc.org/flag)を使ってたけど,  
これからは`Cobra`でサクッと作るようにしたい.  

## おまけ
ねこのかわいい肉球  
![そとちゃん](/images/2020-09-03/sotochan.jpg)  

## 参考

[^1]: [Cobra Generator](https://github.com/spf13/cobra/blob/master/cobra/README.md)  
