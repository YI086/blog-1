---
title: "cobraでCLIを作る"
date: 2020-08-05T22:10:37+09:00
draft: true
tags: ["Go", "作業ログ"]
---

## cobraを試す
GoでCLIを作るときに引数とかサブコマンドをいい感じにしてくれるライブラリのcobraを試した.  

<!--more-->
---

## やったことのまとめ

cobraを使ってあいさつするおもちゃをつくった.  

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
昔は空のディレクトリじゃないとできなかったみたいだけど, 今はファイルが合っても大丈夫.  


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



## おわり


## おまけ
ねこのかわいい肉球  
![そとちゃん](/images/2020-08-05/sotochan.jpg)  

## 参考

[^1]: [Cobra Generator](https://github.com/spf13/cobra/blob/master/cobra/README.md)  
