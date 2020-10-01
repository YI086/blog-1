---
title: "GoでPrometheus用のExporterをつくる"
date: 2020-09-16T22:32:00+09:00
draft: false
tags: ["作業ログ", "Go", "Prometheus", "Docker"]
---

## 自作Exporter
Prometheusでスクレイプする用のExporterを自作してみた.  

<!--more-->
---

## やったことのまとめ

- `Go`の`Prometheus`クライアント[^1]を使って, 自分で設定したメトリクスを表示できる`Exporter`を作成した
- 作成した`Exporter`と`Prometheus`を`Docker`で動かし, スクレイプできることを確認した

![Exporter](/images/2020-09-17/sc01.png)  

https://github.com/uzimihsr/example-exporter  

## つかうもの
- macOS Mojave 10.14
- [go](https://github.com/golang/go) version go1.14.6 darwin/amd64
- [prometheus/client_golang](https://github.com/prometheus/client_golang) v1.7.1
- [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac) 2.1.0.3
  - Docker version 19.03.2
  - docker-compose version 1.24.1

## やったこと

- [Exporterの作成](#exporterの作成)
- [Prometheusでスクレイプ](#prometheusでスクレイプ)

### Exporterの作成

{{< highlight bash >}}
# 作業用ディレクトリとgo.modの作成
$ pwd
/path/to/workspace
$ mkdir example-exporter && cd example-exporter
$ go mod init github.com/uzimihsr/example-exporter
go: creating new go.mod: module github.com/uzimihsr/example-exporter
$ touch main.go
{{< /highlight >}}

`main.go`はこんな感じで作る.  
書き方はDocsの例[^2]と`Prometheus`クライアントのサンプルコード[^3]を参考にした.  

<script src="https://gist.github.com/uzimihsr/4610ea7923274336898733c6588ef962.js"></script>

ポイントは`/metrics`へのリクエストをハンドリングする処理と別に`go routine`を使ってメトリクスを更新する処理を入れているところ.  
こうすることで各メトリクスが並行に更新され, リクエストが来るたびにメトリクスの最新の値を返すことができる.  

試しに動かしてみる.  

{{< highlight bash >}}
# exporterの実行
$ go run main.go
{{< /highlight >}}

ブラウザで http://localhost:2112/metrics を開く.  

![Exporter](/images/2020-09-17/sc01.png)  

こんな感じで`Prometheus`が読めるフォーマットのメトリクスが表示された.  
メトリクス定義に記述したメトリクス名, 説明文, ラベルが反映されていることがわかる.  

```
# HELP example_number Example Gauge
# TYPE example_number gauge
example_number{fuga="fugafuga"} -0.0056473753420378525
# HELP example_total Example Counter
# TYPE example_total counter
example_total{hoge="hogehoge"} 13
```

### Prometheusでスクレイプ

せっかくなので, `Prometheus`でスクレイプしてみる.  
今回は試しに`Docker`上で`Prometheus`のコンテナと今回作った **example-exporter** のコンテナを同時に動かす.  

**example-exporter** 用の`Dockerfile`の書き方は[GoアプリをDockerのscratchイメージで動かす](https://uzimihsr.github.io/post/2020-03-15-golang-build-image/)を参考にする.  

<script src="https://gist.github.com/uzimihsr/7fbec7bd2acbe4ecb050f37e7f70420d.js"></script>

さらに`Prometheus`の設定ファイル`prometheus.yml`と,  
2つのコンテナを動かすために`docker-compose.yml`を作成する.  

<script src="https://gist.github.com/uzimihsr/548f1a8ebae74fb84f1ee6aabed3e1f3.js"></script>

<script src="https://gist.github.com/uzimihsr/e08901b5093d810d68bd933149d1230b.js"></script>

この状態で`docker-compose`で2つのコンテナを起動する.  

{{< highlight bash >}}
# ディレクトリの状態
$ tree .
.
├── Dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
├── main.go
└── prometheus.yml

# docker-composeでコンテナを起動
$ docker-compose up -d
Creating network "example-exporter_default" with the default driver
Creating example-exporter ... done
Creating prometheus       ... done
{{< /highlight >}}

コンテナが起動した状態で http://localhost:9090/graph をホストのブラウザで開くと, `Prometheus`の画面が表示される.  
無事に起動できていれば **example-exporter** がスクレイプできている.  

![Prometheus](/images/2020-09-17/sc02.png)  

試しに **example-exporter** のメトリクス名でクエリを投げてみると, ちゃんと時系列の値が表示される.  
![Prometheus](/images/2020-09-17/sc03.png)  
![Prometheus](/images/2020-09-17/sc04.png)  

**example_total** は`main.go`で設定した通り30秒ごとに値が加算され,  
**example_number** は`main.go`で設定した値の変化の間隔(10秒)が`Prometheus`の`scrape_interval`(15秒)より短いために15秒ごとに値が変化している.  

やったぜ.  
自作の`Exporter`をスクレイプして, 設定したとおりにメトリクスが変化していることが確認できた.  

## おわり
`Prometheus`でスクレイプできるかんたんな`Exporter`を自作してみた.  
`Prometheus`自体が`Go`で書かれているだけあって, `Go`だとかなり楽に記述できた. と思う.  
特に`go routine`のおかげでメトリクスとHTTPハンドラの並行処理が簡単に書けるのがいいと思った.  
自作のAPIとかにこんな感じで`Exporter`を実装すれば監視がめちゃめちゃ楽になりそう.  

## おまけ
あたらしいボールをもらってごきげんのねこ  
![そとちゃん](/images/2020-09-17/sotochan.jpg)  

## 参考

[^1]: [prometheus/client_golang](https://github.com/prometheus/client_golang)  
[^2]: [Instrumenting a Go application](https://prometheus.io/docs/guides/go-application/)  
[^3]: [client_golang/examples](https://github.com/prometheus/client_golang/tree/master/examples)  
