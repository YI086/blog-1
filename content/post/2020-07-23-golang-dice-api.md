---
title: "GoでサイコロAPIを作る"
date: 2020-07-23T14:10:10+09:00
draft: false
tags: ["Go", "作業ログ"]
---

## Web APIつくってみる
Web APIを自分で最初から作ったことがなかったので, Goの練習も兼ねてやってみた.  

<!--more-->
---

## やったことのまとめ

以下のようにランダムなサイコロの出目(`number`)とサイコロの面の数(`faces`)を`JSON`で返すだけのWeb APIを`Go`で作った.  

{{< highlight bash >}}
# 普通に叩くと6面サイコロを振る
$ curl -X GET "localhost:8080"
{"number":1,"faces":6}
$ curl -X GET "localhost:8080"
{"number":5,"faces":6}

# クエリパラメータで面の数を指定できる
$ curl -X GET "localhost:8080?faces=100"
{"number":71,"faces":100}

# ズルをするためのエンドポイントも用意
$ curl -X GET "localhost:8080/cheat"
{"number":6,"faces":6}
$ curl -X GET "localhost:8080/cheat?faces=100&number=99"
{"number":99,"faces":100}
{{< /highlight >}}

## つかうもの
- macOS Mojave 10.14
- [anyenv](https://github.com/anyenv/anyenv) 1.1.1
  - インストール済み
- [goenv](https://github.com/syndbg/goenv) 2.0.0beta11
  - インストール済み
- [go](https://github.com/golang/go) version go1.14.6 darwin/amd64
  - 今回入れる

## やったこと

- [準備](#準備)
- [サイコロの作成](#サイコロの作成)
- [HTTPハンドラの作成](#httpハンドラの作成)
- [テストの作成](#テストの作成)

### 準備
まずは開発環境の準備.  

[公式](https://golang.org/doc/devel/release.html)によると現在(2020/07/16)`Go`の最新版が **1.14.6** なので, これを使うようにする.  

{{< highlight bash >}}
# goenvの更新
$ anyenv install goenv
anyenv: /Users/uzimihsr/.anyenv/envs/goenv already exists
Reinstallation keeps versions directories
continue with installation? (y/N) y
...

Install goenv succeeded!
Please reload your profile (exec $SHELL -l) or open a new session.
$ exec $SHELL -l

# Go 1.14.6のインストール
$ cd
$ goenv install 1.14.6
$ goenv global 1.14.6
$ goenv rehash
$ exec $SHELL -l

# 確認
$ goenv version
1.14.6 (set by /Users/uzimihsr/.anyenv/envs/goenv/version)
$ go version
go version go1.14.6 darwin/amd64
$ echo $GOPATH
/Users/uzimihsr/go/1.14.6
{{< /highlight >}}

次にGitHubリポジトリを[新規作成](https://github.com/new)する.  
![GitHub](/images/2020-07-23/sc01.png)  
作ったリポジトリ : https://github.com/uzimihsr/dice-api  

作成したリポジトリをローカルにcloneして, `Go Module`とか`.gitignore`の準備をする.  
`Go Module`の使い方は[公式](https://github.com/golang/go/wiki/Modules#example)を参考にする.  
`.gitignore`は[gitignore.io](https://www.toptal.com/developers/gitignore)を使って作るのが楽.  

{{< highlight bash >}}
# 適当なディレクトリで作業
$ cd workspace
$ git clone https://github.com/uzimihsr/dice-api.git
Cloning into 'dice-api'...
warning: You appear to have cloned an empty repository.
$ cd dice-api

# Go Moduleの初期化
$ go mod init github.com/uzimihsr/dice-api
go: creating new go.mod: module github.com/uzimihsr/dice-api
$ ls
go.mod

# gitignore.ioのAPIを利用して.gitignoreを作成する
$ curl -o ./.gitignore https://www.toptal.com/developers/gitignore/api/go,macos,linux

# いったんpushしておく
$ echo "# dice-api" > README.md
$ git add .
$ git commit -m "first commit"
$ git push origin master
{{< /highlight >}}

このときのリポジトリは[こんなかんじ](https://github.com/uzimihsr/dice-api/tree/f484784bcd1b569fcf1bc77cd882263ea809b715).  

### サイコロの作成
まずはサイコロの動作を作ってみる.  
[以前作ったもの](https://uzimihsr.github.io/post/2019-09-04-golang-dice/)を流用する.  

{{< highlight bash >}}
$ mkdir dice && cd dice
$ vim dice.go
{{< /highlight >}}

今回は普通にサイコロの出目を返すメソッド`Roll()`の他に指定した出目を返す`Cheat()`を作ってみた.  
<details><summary>`dice.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/6f9b3cc44bae4b4b0738bbe29585e390.js"></script>
</div></details>

ためしに動かしてみる.  

{{< highlight bash >}}
$ cd ..
$ vim main.go
$ go run main.go
6
5
$ go run main.go
1
5
{{< /highlight >}}

<details><summary>`main.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/d3d4cf0ef43f6de5faee53417c6c2aa6.js"></script>
</div></details>

いい感じ.  

### HTTPハンドラの作成
次にこれをWeb APIとして動かすためのハンドラ関数を作る.  

{{< highlight bash >}}
$ mkdir handler && cd handler
$ vim handler.go
{{< /highlight >}}

ポイントはハンドラ関数`DiceHandler()`, `CheatDiceHandler()`の引数で`interface`を受けるようにしているところ.  
こうしておくと後でテストを書くときにmockを使いやすくなる.  
<details><summary>`handler.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/f5029b7b42805a0ea6c5820aa6df099c.js"></script>
</div></details>

これらのハンドラ関数を扱ってHTTPサーバーを動かすため, `main.go`を修正する.  
{{< highlight bash >}}
$ cd ..
$ vim main.go
$ go run main.go
# 動作を確認したら Ctrl+C で終了
{{< /highlight >}}

こちらは`http.NewServeMux()`を使ってリクエストパスごとに異なるハンドラを呼び出すようにしているのがポイント.  
<details><summary>`main.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/d56241bed0c022585885812c438d068f.js"></script>
</div></details>

`main.go`を実行するとMacの警告が出るので, `許可`を選択するとアプリが動く.  
![GitHub](/images/2020-07-23/sc02.png)  

試しに別のターミナルからAPIを叩いてみる.  

{{< highlight bash >}}
# 正常系
$ curl "localhost:8080"
{"number":1,"faces":6}
$ curl "localhost:8080?faces=12"
{"number":10,"faces":12}
$ curl "localhost:8080/cheat"
{"number":6,"faces":6}
$ curl "localhost:8080/cheat?number=1"
{"number":1,"faces":6}
$ curl "localhost:8080/cheat?number=1&faces=18"
{"number":1,"faces":18}

# 異常系
$ curl -i -X POST "localhost:8080"
HTTP/1.1 404 Not Found
Content-Type: application/json
Date: Thu, 23 Jul 2020 08:12:39 GMT
Content-Length: 24

{"message": "not found"}
$ curl -i -X GET "localhost:8080?faces=hoge"
HTTP/1.1 500 Internal Server Error
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Thu, 23 Jul 2020 08:13:04 GMT
Content-Length: 45

strconv.Atoi: parsing "hoge": invalid syntax
$ curl -i -X POST "localhost:8080/cheat"
HTTP/1.1 404 Not Found
Content-Type: application/json
Date: Thu, 23 Jul 2020 08:13:58 GMT
Content-Length: 24

{"message": "not found"}
$ curl -i -X GET "localhost:8080/cheat?number=hoge"
HTTP/1.1 500 Internal Server Error
Content-Type: text/plain; charset=utf-8
X-Content-Type-Options: nosniff
Date: Thu, 23 Jul 2020 08:14:12 GMT
Content-Length: 45

strconv.Atoi: parsing "hoge": invalid syntax
{{< /highlight >}}

サイコロの出目(`number`)と面の数(`faces`)が`JSON`で返ってきて,  
異常なクエリパラメータやHTTPメソッドでリクエストした場合には指定したステータスコード(**404**, **500**)が返ってくることが確認できた.  
やったぜ.  

このときのリポジトリは[こんなかんじ](https://github.com/uzimihsr/dice-api/tree/6048bc4d6a97fdadf30feea5408b0e981335c0ec).  


### テストの作成

作りたいものは作れたのでここで終わってもいいけど, せっかくなのでテストコードを書いてみる.  

まずは`dice`パッケージから.  

{{< highlight bash >}}
$ cd ./dice
$ vim dice_test.go
{{< /highlight >}}

カバレッジ100%になるように書いたけど, 無駄なテストも含まれている.  
<details><summary>`dice_test.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/bd3cfcba754fa444a9cd84f69ff96bbc.js"></script>
</div></details>

次に`handler`パッケージのテストを作る.  
`handler`パッケージは自作の`dice`パッケージに依存しているので, `dice`パッケージのmockを用意したい.  
(今回はDBとかに接続してないので`dice`を直接呼び出してもいいんだけど, サイコロの出目が確率で変わっちゃうのでテストがしづらい)  

こんなときには[gomock](https://github.com/golang/mock)を使う.  
mockしたい`interface`のファイルを指定するだけでmock用のコードを作成してくれるのでめちゃ便利.  

{{< highlight bash >}}
# mockパッケージのインストール
$ cd ..
$ go get github.com/golang/mock/mockgen

# diceのmockを作成
$ mockgen -source ./dice/dice.go -destination ./mock_dice/mock_dice.go
{{< /highlight >}}

<details><summary>`mock_dice.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/9d9ffc5382c8597eeee941e1058ebf10.js"></script>
</div></details>

このmockを使ったテストを書いてみる.  

{{< highlight bash >}}
$ cd handler
$ vim handler_test.go
{{< /highlight >}}

準備したmockを各ハンドラ関数の引数に渡してやることで, 実際の`dice`を呼び出すことなくハンドラ関数のテストができるようになる.  
ハンドラ関数を作るときに`dice`の`interface`を引数で受けるようにしたのはこのため.  
<details><summary>`handler_test.go`</summary><div>
<script src="https://gist.github.com/uzimihsr/9fdf8338eb7487cf58da661766b98b88.js"></script>
</div></details>

テストコードが作れたので, 実際にテストを実行する.  

{{< highlight bash >}}
$ cd ..
# diceパッケージのテスト
$ go test -cover ./dice
ok  	github.com/uzimihsr/dice-api/dice	0.006s	coverage: 100.0% of statements
# handlerパッケージのテスト
$ go test -cover ./handler
ok  	github.com/uzimihsr/dice-api/handler	0.016s	coverage: 100.0% of statements
# 全部まとめてテスト
$ go test -cover ./...
?   	github.com/uzimihsr/dice-api	[no test files]
ok  	github.com/uzimihsr/dice-api/dice	(cached)	coverage: 100.0% of statements
ok  	github.com/uzimihsr/dice-api/handler	(cached)	coverage: 100.0% of statements
?   	github.com/uzimihsr/dice-api/mock_dice	[no test files]
{{< /highlight >}}

これでテストも(一応)できた.  

最終的なディレクトリの構成はこんなかんじ.  
テストを実行したのでパッケージの依存関係を管理するためのファイル`go.mod`と`go.sum`が修正/追加されている.  
こいつらも一緒に`Git`で管理しておくと, 別の環境でこれをビルドするときに便利だったりする. らしい.  

{{< highlight bash >}}
$ tree .
.
├── README.md
├── dice
│   ├── dice.go
│   └── dice_test.go
├── go.mod
├── go.sum
├── handler
│   ├── handler.go
│   └── handler_test.go
├── main.go
└── mock_dice
    └── mock_dice.go

3 directories, 9 files
{{< /highlight >}}

最終的なリポジトリは[こんなかんじ](https://github.com/uzimihsr/dice-api/tree/07e945bed60205b936e9ca3511cf1dafe8881ab8).  

## おわり
Web APIっぽいものを0から作ってみた. テストまでやったのでけっこうしんどかった.  
あとは[ビルド](https://uzimihsr.github.io/post/2020-03-15-golang-build-image/)すれば普通に動くはずなので,  
暇があったら`Kubernetes`で動かすところまでやってみたい.  

## おまけ
おすわりするねこ(ごはんがほしい)  
![そとちゃん](/images/2020-07-23/sotochan.jpg)  

## 参考にしたもの

- https://github.com/golang/go/wiki/Modules#example
- [Goプログラミング実践入門 標準ライブラリでゼロからWebアプリを作る](https://book.impress.co.jp/books/1115101145)
- https://github.com/golang/mock/blob/master/README.md
