---
title: "PushgatewayでCronJobの監視を行う"
date: 2020-04-30T21:40:42+09:00
draft: false
tags: ["作業ログ", "Raspberry Pi", "Prometheus", "Pushgateway", "Go", "Docker", "Kubernetes"]
---

## ジョブの監視
Pushgatewayを使った監視をやってみた.  

<!--more-->
---

## やったことのまとめ

- ラズパイに`Pushgateway`をインストールした
- `Kubernetes`の`CronJob`からメトリクスをPushしてみた
- 簡単な監視ルールを設定した

## つかうもの

- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
    - 監視サーバとして使用
- Prometheus
    - version="2.15.2"
    - [インストール済み](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)
- Pushgateway
    - version="1.2.0"
    - **今回入れる**
- macOS Mojave 10.14
    - Go, Docker, Kubernetesはこちらで実行
- Go(golang)
    - go version go1.13
- Docker
    - Version: 19.03.2
- Kubernetesクラスタ(Minikube)
    - GitVersion:"v1.16.2"
    - [MacBook上で構築](https://uzimihsr.github.io/post/2019-10-28-hello-minikube/)

## 構成
![component](/images/2020-04-30-component.png)  

[Pushgateway](https://github.com/prometheus/pushgateway)とは...  

>"The Pushgateway is an intermediary service which allows you to push metrics from jobs which cannot be scraped."  
>"Pushgatewayとは, スクレイプが不可能なジョブのメトリクスをプッシュするための仲介サービスです."  
>(https://prometheus.io/docs/practices/pushing/ より超意訳)  

`Node exporter`みたいなExporterは監視対象が動いてる間メトリクスを吐き出しつづけるから`Prometheus`で定期的にスクレイプ(pull)できるけど,  

バッチジョブみたいに動いている間しか情報を持たないものは`Prometheus`からスクレイプできず, 通常の方法では監視ができない.  

これを解決するため, `Pushgateway`を使ってジョブからのメトリクスのpushを受け付けて永続化し, `Prometheus`でこのメトリクスをpullすることでジョブの監視を行う.  

## やったこと

- [Pushgatewayのセットアップ](#pushgatewayのセットアップ)
- [メトリクスのpushとPrometheusによるスクレイプ](#メトリクスのpushとprometheusによるスクレイプ)
- [CronJobの監視](#cronjobの監視)

### Pushgatewayのセットアップ
まずはラズパイに`Pushgateway`をインストールしていく.  

[ダウンロードページ](https://prometheus.io/download/#pushgateway)で`Architecture`を`armv7`にした状態で  
**pushgateway-1.2.0.linux-armv7.tar.gz** のダウンロードリンクを確認する.  

ラズパイにSSHしてダウンロード, インストール, 起動までやってみる.  
```bash
# 以下はRaspberry Piで実行
# 任意のディレクトリで作業
$ cd workspace

# 確認したURLからダウンロードして展開, 移動
$ wget https://github.com/prometheus/pushgateway/releases/download/v1.2.0/pushgateway-1.2.0.linux-armv7.tar.gz
$ tar -xzf pushgateway-1.2.0.linux-armv7.tar.gz
$ sudo cp pushgateway-1.2.0.linux-armv7/pushgateway /usr/local/bin/

# 起動してみる
$ /usr/local/bin/pushgateway
level=info ts=2020-04-28T13:24:39.058Z caller=main.go:83 msg="starting pushgateway" version="(version=1.2.0, branch=HEAD, revision=b7e0167e9574f4f88404dde9653ee1d3c940f2eb)"
level=info ts=2020-04-28T13:24:39.058Z caller=main.go:84 build_context="(go=go1.13.8, user=root@0e823ccfff84, date=20200311-18:57:04)"
level=info ts=2020-04-28T13:24:39.062Z caller=main.go:137 listen_address=:9091
# 確認次第Ctrl+Cで終了する
```

**[ラズパイのIP]:9091** を開いてみると`Pushgateway`のUI画面が開く.  
今のところは何もメトリクスがないのでヘッダ以外は真っ白な画面.  
![Pushgateway](/images/2020-04-30-sc01.png)  

動作確認ができたので,  
サービス化してついでに`nginx`での[リバースプロキシ](https://uzimihsr.github.io/post/2020-01-29-nginx/)にも対応させておく.  

```bash
# 以下はRaspberry Piで実行
# service起動に必要なuser(pushgateway)を追加する
$ sudo useradd -U -s /sbin/nologin -M -d / pushgateway

# serviceファイルの作成
$ sudo vim /etc/systemd/system/pushgateway.service

# serviceの自動起動設定と起動
$ sudo systemctl daemon-reload
$ sudo systemctl enable pushgateway.service
Created symlink /etc/systemd/system/multi-user.target.wants/pushgateway.service → /etc/systemd/system/pushgateway.service.
$ sudo systemctl start pushgateway.service
● pushgateway.service - Pushgateway
   Loaded: loaded (/etc/systemd/system/pushgateway.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2020-04-28 22:41:04 JST; 28s ago
 Main PID: 30393 (pushgateway)
    Tasks: 8 (limit: 2200)
   Memory: 3.2M
   CGroup: /system.slice/pushgateway.service
           └─30393 /usr/local/bin/pushgateway --web.external-url=http://localhost:8080/pushgateway/ --web.route-prefix=/
...
Apr 28 22:41:04 raspberrypi pushgateway[30393]: level=info ts=2020-04-28T13:41:04.858Z caller=main.go:137 listen_address=:9091

# nginx設定ファイルの変更, 再起動
$ sudo vim /etc/nginx/conf.d/default.conf
$ sudo systemctl restart nginx
```

<details><summary>`pushgateway.service`</summary><div>
```service
[Unit]
Description=Pushgateway

[Service]
User=pushgateway
ExecStart=/usr/local/bin/pushgateway \
  --web.external-url=http://localhost:8080/pushgateway/ \
  --web.route-prefix=/

[Install]
WantedBy=multi-user.target
```
</div></details>
<details><summary>`default.conf`</summary><div>
```nginx
server {
	listen	8080;

	location /prometheus/ {
		proxy_pass	http://localhost:9090/;
	}

	location /alertmanager/ {
		proxy_pass	http://localhost:9093/;
	}

	location /pushgateway/ {
		proxy_pass	http://localhost:9091/;
	}

	location /grafana/ {
		proxy_pass	http://localhost:3000/;
	}
}
```
</div></details>

ここまで終わらせれば **[ラズパイのIP]:[nginxのポート]/pushgateway** で`Pushgateway`のUIが開けるようになっているはず.  

以上で`Pushgateway`のセットアップは完了.  

### メトリクスのpushとPrometheusによるスクレイプ
`Pushgateway`の準備ができたので, 実際にメトリクスをpushしてみる.  

各言語のクライアントを使った叩き方は[ここ](https://prometheus.io/docs/instrumenting/pushing/)にあるけど,  
今回は敢えて[API](https://github.com/prometheus/pushgateway/blob/master/README.md#api)を使ってコマンドラインからpushしてみる.  

```bash
# 以下はMacで実行
# job="some_job"のグループにsome_metricという名前のGaugeをpushする
$ echo "some_metric 3.14" | curl --data-binary @- http://<ラズパイのIP>:<nginxのポート>/pushgateway/metrics/job/some_job
```

Pushが成功していれば`Pushgateway`のUIにメトリクスの情報が表示される.  

`some_metric`以外の2つのメトリクス(`push_time_seconds`, `push_failure_time_seconds`)はそれぞれ  
メトリクスの更新に(成功|失敗)したUNIX時間を値として持つメトリクス.  

![Pushgateway](/images/2020-04-30-sc02.png)  

こんな感じでジョブ実行時にpushすることで, `Pushgateway`にジョブのメトリクスが貯まっていく.  

これらのメトリクスを時系列データとして扱うため,  
`Prometheus`で`Pushgateway`の持つメトリクスを定期的に収集(スクレイプ)する設定を追加してみる.  
ラベルの衝突を避けるため, `Pushgateway`のスクレイプ設定には`honor_labels`を追加しておく.  

```bash
# 以下はRaspberry Piで実行
# Prometheus設定ファイルを編集, 再起動
$ sudo vim /usr/local/prometheus/prometheus.yml
$ sudo systemctl restart prometheus.service
```

<details><summary>`prometheus.yml`</summary><div>
```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
rule_files:
  - rules.yml
alerting:
  alertmanagers:
    - static_configs:
      - targets: ['localhost:9093']
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
    - targets: ['localhost:9100']
  # ここから下を追加
  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
    - targets: ['localhost:9091'] # Pushgatewayが起動しているポートを指定
```
</div></details>

設定が問題なく反映されていれば,  
`Prometheus`から`Pushgateway`がスクレイプできているのを確認できる.  

![Prometheus](/images/2020-04-30-sc03.png)  

試しに何回かメトリクスをpushして, それから`Prometheus`で時系列データを確認してみる.  

```bash
# 以下はMacで実行
# 先程とは違う値をpush
$ echo "some_metric 1.41" | curl --data-binary @- http://<ラズパイのIP>:<nginxのポート>/pushgateway/metrics/job/some_job
# 少し待ってから再度push
$ echo "some_metric 2.71" | curl --data-binary @- http://<ラズパイのIP>:<nginxのポート>/pushgateway/metrics/job/some_job
```

`some_metric`の値を見てみると, 次のようになる.  
注意すべき点としては, 新たにメトリクスがpushされるまで以前の値が変わらず保持され続けること.  
`Pushgateway`はあくまでジョブのメトリクスを受け付けるためのものなので,  
ジョブが実行されていない間は最後にpushされた値を保持し続けて`Prometheus`がスクレイプできるようにしている.  

![Prometheus](/images/2020-04-30-sc04.png)  

これでジョブのメトリクスを`Pushgateway`経由で`Prometheus`がスクレイプできるようになった.  

### CronJobの監視
ジョブのメトリクスが収集できるようになったので, いよいよ簡単な監視をしてみる.  

まずは監視対象として`Go`で乱数のメトリクスをpushするだけのサンプルジョブを作成し,  
[この手順](https://uzimihsr.github.io/post/2020-03-15-golang-build-image/)で`Docker image`化してみる.  
`Prometheus`クライアントの使い方は[pushパッケージのGoDoc](https://godoc.org/github.com/prometheus/client_golang/prometheus/push#Pusher.Push)を参考にした.  

```bash
# 以下はMacで実行
# ジョブプログラムの作成
$ vim main.go

# 動作確認
$ go run main.go --endpoint=192.168.3.200:9091
192.168.3.200:9091
sample_job
Metrics pushed successfully.

# docker化
$ vim Dockerfile
$ docker image build -t golang-sample-job:latest .
...
Successfully built ff903de0d164
Successfully tagged golang-sample-job:latest
$ docker image ls golang-sample-job
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
golang-sample-job   latest              ff903de0d164        About a minute ago   12.6MB

# 動作確認
$ docker container run --rm golang-sample-job:latest --endpoint=192.168.3.200:9091
192.168.3.200:9091
sample_job
Metrics pushed successfully.

# タグをつけ直してDockerHubにアップロード
$ docker image tag golang-sample-job:latest uzimihsr/golang-sample-job:latest
$ docker image push uzimihsr/golang-sample-job:latest
```

<details><summary>`main.go`</summary><div>
```golang
package main

import (
	"flag"
	"fmt"
	"math/rand"
	"os"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/push"
)

func main() {
	// Pushgatewayのエンドポイントとジョブ名は実行時引数で渡す
	var pushgatewayEndpoint = flag.String("endpoint", "http://pushgateway:9091", "Pushgateway endpoint. default: http://pushgateway:9091")
	var jobName = flag.String("job", "sample_job", "Job name. default: sample_job")
	flag.Parse()
	if flag.NFlag() > 2 {
		fmt.Println("flags : --endpoint, --job")
		os.Exit(1)
	}
	fmt.Println(*pushgatewayEndpoint)
	fmt.Println(*jobName)

	// Gaugeの作成
	randomValue := prometheus.NewGauge(prometheus.GaugeOpts{
		Name: "random_value",
		Help: "Float64 random value generated by golang.",
	})

	// 乱数をGaugeにセット
	rand.Seed(time.Now().UnixNano())
	randomValue.Set(rand.Float64())

	// メトリクスをpush
	if err := push.New(*pushgatewayEndpoint, *jobName).
		Collector(randomValue).
		Grouping("sample_label", "sample_label_value").
		Push(); err != nil {
		fmt.Println("Could not push metrics to Pushgateway:", err)
		os.Exit(1)
	}
	fmt.Println("Metrics pushed successfully.")
}
```
</div></details>
<details><summary>`Dockerfile`</summary><div>
```docker
# build
FROM golang:1.13
COPY . ./goapp
WORKDIR ./goapp
RUN go mod download && \
    CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app .

# run
FROM scratch
LABEL maintainer="usimihsr"
WORKDIR goapp
COPY --from=0 /app ./
COPY --from=0 /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/ca-certificates.crt

ENTRYPOINT ["./app"]
```
</div></details>
実際にビルドした`image` : [uzimihsr/golang-sample-job](https://hub.docker.com/r/uzimihsr/golang-sample-job)  

これでサンプルジョブの`Docker image`が作成できたので,  
次にこれを`CronJob`化する.  

```bash
# 以下はMacで実行
# マニフェストの作成とCronJobの作成
$ vim cronjob.yaml
$ kubectl apply -f cronjob.yaml
$ kubectl get cronjob sample-cronjob
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
sample-cronjob   */1 * * * *   False     0        6m37s           3h12m
```

この`CronJob`では`Pod`定義の`.spec.initContainers`で  
メインコンテナの前にメトリクスをpushするコンテナを必ず実行するように定義している.  
こうすることでメインコンテナの成否に関わらずジョブの監視が可能になる.  

<details><summary>`cronjob.yaml`</summary><div>
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: sample-cronjob
spec:
  schedule: "*/1 * * * *"
  concurrencyPolicy: Allow
  startingDeadlineSeconds: 30
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      completions: 1
      parallelism: 1
      backoffLimit: 0
      template:
        spec:
          initContainers:
          - name: push-metrics
            image: uzimihsr/golang-sample-job:latest
            imagePullPolicy: IfNotPresent
            args: ["--endpoint=<ラズパイのIP>:9091"]
          containers:
          - name: main-batch
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: Never

```
</div></details>

`CronJob`が問題なく動作していれば1分ごとのジョブ実行時に乱数のメトリクス`random_value`がpushされるので,  
`Prometheus`で確認してみる.  

![Prometheus](/images/2020-04-30-sc05.png)  

これらのメトリクスを使って簡単な監視ができるので, 以下のアラートルールを作成してみる.  

```bash
# 以下はMacで実行
# アラートルールを編集して反映
$ vim /usr/local/prometheus/rules.yml
$ sudo systemctl restart prometheus.service
```

<details><summary>`rules.yml`</summary><div>
```yaml
groups:
  - name: instance
    rules:
    - alert: InstanceDown
      expr: up == 0
      for: 1m
  # 以下のルールを追加
  - name: pushgateway
    rules:
    - alert: CronJobNotScheduled
      expr: time() - push_time_seconds{job="sample_job"} > 60 * 2
    - alert: CronJobFailed
      expr: random_value{job="sample_job"} > 0.5
      for: 2m
```
</div></details>

`CronJobNotScheduled`は最後に`CronJob`が実行されてからの時間が2分以上になった場合に発火するアラート.  
pushの際に自動で更新されるメトリクス`push_time_seconds`の値と現在の時間を比較して条件判定している.  
クラスタに障害があったりして2分以上`CronJob`が実行されなかった場合はメトリクスがpushされないので,  
このルールで検知できる.  

![Prometheus](/images/2020-04-30-sc06.png)  

`CronJobFailed`は`CronJob`が2分以上失敗し続けた場合に発火する(ことを想定した)アラート.  
今回は例としてジョブ開始時にpushされる乱数のメトリクス`random_value`を条件判定に使っているが,  
ジョブのプロセスの終了ステータスをメトリクス化してジョブの最後にpushさせればジョブが連続して失敗した場合に検知できる.  

![Prometheus](/images/2020-04-30-sc07.png)  

試しにこのアラートルールが有効になっている状態で`CronJob`を停止してみる.  

```bash
# 以下はMacで実行
# CronJobを停止する
$ kubectl patch cronjob sample-cronjob -p '{"spec":{"suspend":true}}'
$ kubectl get cronjob sample-cronjob
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
sample-cronjob   */1 * * * *   True      0        49s             3h49m
```

`CronJob`を停止すると新たなメトリクスがpushされなくなるので,  
2分以上経過した時点で`CronJobNotScheduled`が発火し,  
停止した時点で最後にpushされた`random_value`がしきい値(**0.5**)より大きい値であったので`CronJobFailed`も発火した.  

![Prometheus](/images/2020-04-30-sc08.png)  

これで`Pushgateway`を利用した`CronJob`の監視ができるようになった.  
やったぜ.  

## おわり
`Pushgateway`のインストールから`CronJob`の監視までの流れを一通りやってみた.  
`CronJob`がちゃんと実行されているかを`Kubernetes API`や`kubectl`でいちいち確認するのは大変なので,  
こんな感じで監視できると便利だと思う.  

## おまけ
クッキーを焼くそとちゃん  
![クッキーを焼くそとちゃん](/images/2020-04-30-sotochan.jpg)  
