---
title: "GoアプリをDockerのscratchイメージで動かす"
date: 2020-03-15T13:36:21+09:00
draft: false
tags: ["Go", "Docker", "作業ログ"]
---

## 軽いイメージをつくる
Go(golang)のDocker buildの練習をしてみた.  

<!--more-->
---

## やったことのまとめ

- Go(golang)で作ったアプリをDockerイメージにした
- マルチステージビルドを使って軽いイメージを作った

scratchを使う場合は次のような`Dockerfile`を書けばいい.  

```dockerfile
# ビルド用イメージ
FROM golang:1.13

# mainパッケージがあるディレクトリ(.)をまるごとコピー
COPY . ./goapp
WORKDIR ./goapp

# goapp内のgo.mod, go.sumで依存関係を管理している場合に使用
RUN go mod download

# クロスコンパイル
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .

# バイナリを載せるイメージ
FROM scratch
WORKDIR goapp

# ビルド済みのバイナリをコピー
COPY --from=0 /app ./

# httpsで通信を行う場合に使用
COPY --from=0 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

ENTRYPOINT ["./app"]
```

## つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- Go(golang)
    - go version go1.13 darwin/amd64
- Docker
    - Docker version 19.03.2, build 6a30dfc
- なんかのGoアプリ
    - [前回作ったやつ](https://uzimihsr.github.io/post/2020-03-09-golang-api-client/)を使用

## やったこと

- [普通にDockerイメージをビルド](#普通にdockerイメージをビルド)
- [マルチステージビルドを試す](#マルチステージビルドを試す)
- [scratchを使う](#scratchを使う)

### 普通にDockerイメージをビルド
Goのアプリはなんでもいいはずなので,  
前回作ったwikipedia検索するおもちゃをそのまま使ってみる.  
https://github.com/uzimihsr/wikipedia-search  

`Dockerfile`を書くとこんな感じ.  
ベースイメージは[golang:1.13](https://hub.docker.com/layers/golang/library/golang/1.13/images/sha256-be37fd7a30b94a720a45ba5dcc0cf386d043acc5f4f61db7c6736fd10ee621bd?context=explore)でビルドしてみる.  

`Dockerfile-golang`  
```dockerfile
# ベースイメージが大きいパターン
FROM golang:1.13
LABEL maintainer="usimihsr"

# ファイルを全部コピー
COPY . ./goapp
WORKDIR ./goapp

# go.modとgo.sumを使って管理している依存関係をダウンロードしてビルド
RUN go mod download && \
    go build -o ./app .

# バイナリをエントリーポイントに指定
ENTRYPOINT ["./app"]
```

ビルドして動かしてみる.  

```bash
$ git clone https://github.com/uzimihsr/wikipedia-search.git
$ cd wikiepdia-search
# Dockerイメージをビルドする
$ docker image build -t wikipedia-search:golang -f Dockerfile-golang .
Sending build context to Docker daemon  87.55kB
Step 1/6 : FROM golang:1.13
 ---> 3a7408f53f79
Step 2/6 : LABEL maintainer="usimihsr"
 ---> Using cache
 ---> b5ba8c17244f
Step 3/6 : COPY . ./goapp
 ---> d2bffdafe570
Step 4/6 : WORKDIR ./goapp
 ---> Running in def85395de98
Removing intermediate container def85395de98
 ---> d215e88c95ba
Step 5/6 : RUN go mod download &&     go build -o ./app .
 ---> Running in 307004980e61
Removing intermediate container 307004980e61
 ---> 9873d8f458a6
Step 6/6 : ENTRYPOINT ["./app"]
 ---> Running in 4f189dba7fa1
Removing intermediate container 4f189dba7fa1
 ---> f75198355e97
Successfully built f75198355e97
Successfully tagged wikipedia-search:golang

# ENTRYPOINTを無視してbashを起動
# 指定したディレクトリにコピーしたファイルとビルドしたバイナリが確認できる
$ docker container run --rm -it --entrypoint='' wikipedia-search:golang /bin/bash
root@6f62f4307560:/go/goapp$ pwd
/go/goapp
root@6f62f4307560:/go/goapp$ ls
Dockerfile-golang  README.md  app  go.mod  main.go
root@6f62f4307560:/go/goapp$ exit

# コンテナを動かしてみる
# <-srlimit=5 'イチロー'>はENTRYPOINTのバイナリ(./app)実行時に渡すパラメータ
$ docker container run wikipedia-search:golang -srlimit=5 'イチロー'
---------------------------------------------------
イチロー
https://ja.wikipedia.org/?curid=1432262
---------------------------------------------------
首位打者 (日本プロ野球)
https://ja.wikipedia.org/?curid=38085
---------------------------------------------------
国道262号
https://ja.wikipedia.org/?curid=126147
---------------------------------------------------
河上イチロー
https://ja.wikipedia.org/?curid=3682529
---------------------------------------------------
新井宏昌
https://ja.wikipedia.org/?curid=688515
---------------------------------------------------
```

問題なく動いた.  

### マルチステージビルドを試す
上記の手順でGoアプリのイメージが問題なくビルドできた. が,  
ベースイメージに使わないファイルが大量にあるため,  
かなり重いイメージ(810MB)が出来上がってしまった.  

```bash
$ docker image ls wikipedia-search
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
wikipedia-search    golang              f75198355e97        6 minutes ago       810MB
```

実際に使いたいのはバイナリ(app)だけなので,  
[マルチステージビルド](https://docs.docker.com/develop/develop-images/multistage-build/)を使ってビルド済みのバイナリだけを軽いイメージに載せてみる.  
試しに軽量Linuxイメージの[alpine:latest](https://hub.docker.com/layers/alpine/library/alpine/latest/images/sha256-ddba4d27a7ffc3f86dd6c2f92041af252a1f23a8e742c90e6e1297bfa1bc0c45?context=explore)を使ってみる.  

`Dockerfile-alpine`  
```dockerfile
# golangをビルド用イメージとして使うパターン
FROM golang:1.13
COPY . ./goapp
WORKDIR ./goapp

# ビルド時にクロスコンパイルのオプションを指定
RUN go mod download && \
    CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .

# ベースイメージはalpineを指定
FROM alpine:latest
LABEL maintainer="usimihsr"
WORKDIR goapp

# ビルド用イメージからバイナリをコピー
COPY --from=0 /app ./

# httpsで通信するのに必要なCA証明書を用意する
RUN apk --no-cache add ca-certificates

ENTRYPOINT ["./app"]
```

```bash
# ビルド
$ docker image build -t wikipedia-search:alpine -f Dockerfile-alpine .
Sending build context to Docker daemon   98.3kB
Step 1/10 : FROM golang:1.13
 ---> 3a7408f53f79
Step 2/10 : COPY . ./goapp
 ---> 2ce3dd5f2452
Step 3/10 : WORKDIR ./goapp
 ---> Running in 66c34bbf39c1
Removing intermediate container 66c34bbf39c1
 ---> 6876fbc22a61
Step 4/10 : RUN go mod download &&     CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .
 ---> Running in 98346481243b
Removing intermediate container 98346481243b
 ---> 3bc411f60c04
Step 5/10 : FROM alpine:latest
 ---> 961769676411
Step 6/10 : LABEL maintainer="usimihsr"
 ---> Running in dbddd972c873
Removing intermediate container dbddd972c873
 ---> cb8ff63adce6
Step 7/10 : WORKDIR goapp
 ---> Running in cad6ec5d31df
Removing intermediate container cad6ec5d31df
 ---> d88c328fe670
Step 8/10 : COPY --from=0 /app ./
 ---> 5a8738b30e43
Step 9/10 : RUN apk --no-cache add ca-certificates
 ---> Running in c271e71b475e
fetch http://dl-cdn.alpinelinux.org/alpine/v3.10/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.10/community/x86_64/APKINDEX.tar.gz
(1/1) Installing ca-certificates (20190108-r0)
Executing busybox-1.30.1-r2.trigger
Executing ca-certificates-20190108-r0.trigger
OK: 6 MiB in 15 packages
Removing intermediate container c271e71b475e
 ---> 7a22400f095e
Step 10/10 : ENTRYPOINT ["./app"]
 ---> Running in a3f75ef34223
Removing intermediate container a3f75ef34223
 ---> b9635a037fe6
Successfully built b9635a037fe6
Successfully tagged wikipedia-search:alpine

# マルチステージビルドで置かれたバイナリを確認
$ docker container run --rm -it --entrypoint='' wikipedia-search:alpine /bin/ash
/goapp $ pwd
/goapp
/goapp $ ls
app
/goapp $ exit

# アプリを実行
$ docker container run wikipedia-search:alpine -srlimit=5 'イチロー'
---------------------------------------------------
イチロー
https://ja.wikipedia.org/?curid=1432262
---------------------------------------------------
首位打者 (日本プロ野球)
https://ja.wikipedia.org/?curid=38085
---------------------------------------------------
国道262号
https://ja.wikipedia.org/?curid=126147
---------------------------------------------------
河上イチロー
https://ja.wikipedia.org/?curid=3682529
---------------------------------------------------
新井宏昌
https://ja.wikipedia.org/?curid=688515
---------------------------------------------------
```

問題なく動いた.  

イメージもだいぶ軽くなった(810MB -> 13.5MB).  
```bash
$ docker image ls wikipedia-search
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
wikipedia-search    alpine              9fc6e1d19baf        18 seconds ago      13.5MB
wikipedia-search    golang              f75198355e97        52 minutes ago      810MB
```
やったぜ.  

### scratchを使う
なんと! 世の中にはalpineよりもっと軽いイメージがあるらしい.  
[scratch](https://hub.docker.com/_/scratch?tab=description)はDockerの最小イメージで, 中にはなんにも入っていない.  
こいつを使えばめちゃめちゃ軽いイメージが作れるのでは?  

やってみる.  

`Dockerfile-scratch`  
```Dockerfile
# golangをビルド用イメージとして使うパターン
FROM golang:1.13
COPY . ./goapp
WORKDIR ./goapp
RUN go mod download && \
    CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .

# ベースイメージはscratchを指定
FROM scratch
LABEL maintainer="usimihsr"
WORKDIR goapp
COPY --from=0 /app ./

# httpsで通信するのに必要なCA証明書を用意する
COPY --from=0 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

ENTRYPOINT ["./app"]
```

```bash
# ビルド
$ docker image build -t wikipedia-search:scratch -f Dockerfile-scratch .
Sending build context to Docker daemon   98.3kB
Step 1/10 : FROM golang:1.13
 ---> 3a7408f53f79
Step 2/10 : COPY . ./goapp
 ---> Using cache
 ---> 2ce3dd5f2452
Step 3/10 : WORKDIR ./goapp
 ---> Using cache
 ---> 6876fbc22a61
Step 4/10 : RUN go mod download &&     CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .
 ---> Using cache
 ---> 3bc411f60c04
Step 5/10 : FROM scratch
 --->
Step 6/10 : LABEL maintainer="usimihsr"
 ---> Running in ca17656408ee
Removing intermediate container ca17656408ee
 ---> a27e151d506a
Step 7/10 : WORKDIR goapp
 ---> Running in 6f5f0ef76f49
Removing intermediate container 6f5f0ef76f49
 ---> 19dcf5fdbbb4
Step 8/10 : COPY --from=0 /app ./
 ---> 76bbcf3bc727
Step 9/10 : COPY --from=0 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt
 ---> 0f5a0c01f2c0
Step 10/10 : ENTRYPOINT ["./app"]
 ---> Running in fdaa63f93c93
Removing intermediate container fdaa63f93c93
 ---> eafc764f5a6e
Successfully built eafc764f5a6e
Successfully tagged wikipedia-search:scratch

$ docker container run wikipedia-search:scratch -srlimit=5 'イチロー'
---------------------------------------------------
イチロー
https://ja.wikipedia.org/?curid=1432262
---------------------------------------------------
首位打者 (日本プロ野球)
https://ja.wikipedia.org/?curid=38085
---------------------------------------------------
国道262号
https://ja.wikipedia.org/?curid=126147
---------------------------------------------------
河上イチロー
https://ja.wikipedia.org/?curid=3682529
---------------------------------------------------
新井宏昌
https://ja.wikipedia.org/?curid=688515
---------------------------------------------------
```

これまでと同じように動かせた.  

さらにイメージも軽くなった(13.5MB -> 7.52MB).  
```bash
$ docker image ls wikipedia-search
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
wikipedia-search    scratch             eafc764f5a6e        9 seconds ago       7.52MB
wikipedia-search    alpine              b9635a037fe6        35 seconds ago      13.5MB
wikipedia-search    golang              f75198355e97        About an hour ago   810MB
```
やったぜ.  
Goのアプリを軽いDocker imageにすることができた.  

作ったイメージはここ.  
https://hub.docker.com/r/uzimihsr/wikipedia-search/tags  

## おまけ
ひざの上で寝るねこ  
![そとちゃん](/images/2020-03-15-sotochan.jpg)  
