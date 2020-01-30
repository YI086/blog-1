---
title: "AlertmanagerからGmailを使って通知するようにした"
date: 2020-01-27T22:52:41+09:00
draft: false
tags: ["作業ログ", "Raspberry Pi", "Prometheus"]
---

## やばい時には通知する
監視の練習として, Alertmanagerを使ってラズパイから通知をGmailで送るようにした.  

<!--more-->
---

## やったことのまとめ

- Googleアカウントの設定をしてSMTPサーバーを使えるようにした
- ラズパイにAlertmanagerを突っ込んでGmail経由で通知を送るようにした

## つかうもの

- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
- Node exporter
    - version="0.18.1"
- Prometheus
    - version="2.15.2"
- Alertmanager
    - version="0.20.0"
    - 今回入れる
- Googleのアカウント

Node exporterとPrometheusは[ラズパイにインストール済み](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)  

## 構成

![構成図](/images/2020-01-27-component.png)  
<div>Icons made by <a href="https://www.flaticon.com/authors/smashicons" title="Smashicons">Smashicons</a> from <a href="https://www.flaticon.com/" title="Flaticon">www.flaticon.com</a></div>
<br>

今回は図のような構成で通知を送る.  
**Prometheus** は各 **exporter** を常に監視しており,  
監視対象(ラズパイ)に何か問題が発生した場合 **Node exporter** のメトリクスが変化する.  
**Prometheus** では監視対象のメトリクスについて設定した条件(放っておいたらまずい状態)を満たした際にアラートを発火させる.  
**Alertmanager** は発火したアラートに応じた処理を行うが,  
今回はGmail経由で管理者に通知するようにする.  

## やったこと

- [SMTPサーバーの設定](#smtpサーバーの設定)
- [Alertmanagerのインストール](#alertmanagerのインストール)
- [通知設定とテスト](#通知設定とテスト)

### SMTPサーバーの設定

まずはラズパイ上のアプリケーションからメールを送れるようにSMTPサーバーの設定を行う.  
今回はGmailをSMTPサーバーとして使用するが, 他に利用可能なメールサーバーがあれば飛ばしても良い.  

[ここ](https://support.google.com/mail/answer/185833)を参考に, Googleアカウントでアプリ用パスワードを設定する.  
https://myaccount.google.com/u/1/security を開き, `セキュリティ`->`2段階認証プロセス`と進む.  

![sc](/images/2020-01-27-sc01.png)  

もろもろの設定をして2段階認証を有効化する.  
細かい設定手順は[ここ](https://support.google.com/accounts/answer/185839)を参考にする.  

![sc](/images/2020-01-27-sc02.png)  

再度 https://myaccount.google.com/u/1/security を開くと`アプリパスワード`の項目が増えているので進む.  
`アプリを選択`から`その他(名前を入力)`を選び, 適当な名前(alertmanager)を入力して`生成`を押す.  

![sc](/images/2020-01-27-sc03.png)  

16文字の`アプリパスワード`が生成されるので覚えておく.  
このパスワードを使ってラズパイからメールを送る.  

![sc](/images/2020-01-27-sc04.png)  

SMTPサーバーの設定は[ここ](https://support.google.com/a/answer/176600)を参考に後ほどAlertmanagerの設定に記述する.  

|項目|値|
|---|---|
|SMTPホスト名|`smtp.gmail.com`|
|ポート番号|`587`|
|アカウント|`アプリパスワードを発行したアカウント名`|
|パスワード|`アプリパスワード`|

以上でGmailをSMTPサーバーとして使うための準備は完了.  

### Alertmanagerのインストール

次にPrometheusで発火したアラートを処理するためのAlertmanagerをインストールする.  
ファイルは[前回](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)と同じように[ここ](https://prometheus.io/download/#alertmanager)からダウンロードする. `Architecture`は`armv7`であることに注意.  

```bash
# 任意のディレクトリで実行
$ cd /path/to/workspace

# Alertmanagerをダウンロードして展開, フォルダを移動
$ wget https://github.com/prometheus/alertmanager/releases/download/v0.20.0/alertmanager-0.20.0.linux-armv7.tar.gz
$ tar -xzf alertmanager-0.20.0.linux-armv7.tar.gz
$ sudo cp -a alertmanager-0.20.0.linux-armv7 /usr/local/alertmanager

# service用ユーザの作成, データ保存用ディレクトリの作成
$ sudo useradd -U -s /sbin/nologin -M -d / alertmanager
$ sudo mkdir -p /var/lib/alertmanager/data
$ sudo chown -R alertmanager:alertmanager /var/lib/alertmanager

# serviceファイルを作成
$ sudo vim /etc/systemd/system/alertmanager.service

# serviceを起動
$ sudo systemctl daemon-reload
$ sudo systemctl enable alertmanager
Created symlink /etc/systemd/system/multi-user.target.wants/alertmanager.service → /etc/systemd/system/alertmanager.service.
$ sudo systemctl start alertmanager
$ systemctl status alertmanager
● alertmanager.service - Alertmanager
   Loaded: loaded (/etc/systemd/system/alertmanager.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2020-01-27 00:10:38 JST; 4s ago
 Main PID: 9413 (alertmanager)
    Tasks: 13 (limit: 2200)
   Memory: 7.7M
   CGroup: /system.slice/alertmanager.service
           └─9413 /usr/local/alertmanager/alertmanager --config.file=/usr/local/alertmanager/alertmanager.yml --storage.path=/var/lib/alertmanager/data

...
Jan 27 00:10:38 raspberrypi alertmanager[9413]: level=info ts=2020-01-26T15:10:38.829Z caller=main.go:497 msg=Listening address=:9093
...
```

<details><summary>`alertmanager.service`</summary><div>
```service
[Unit]
Description=Alertmanager

[Service]
User=alertmanager
ExecStart=/usr/local/alertmanager/alertmanager \
  --config.file=/usr/local/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager/data

[Install]
WantedBy=multi-user.target
```
</div></details>

`<ラズパイのIP>:9093`をブラウザで開くとAlertmanagerのUI画面が表示される.  
(ラズパイのブラウザで開く場合は`localhost:9093`)  

![AlertmanagerのUI画面](/images/2020-01-27-sc05.png)  

これでAlertmanagerのインストールと起動ができた.  

### 通知設定とテスト

次にPrometheusでアラートを発生させるための設定を行う.  
アラートルールの設定ファイル`rules.yml`で監視対象がダウンしたときに発火するInstanceDownアラートを定義し,  
`prometheus.yml`でこれを参照するよう設定する.  

```bash
# アラートの条件を記述するファイルを作成
$ vim /usr/local/prometheus/rules.yml

# prometheusの設定ファイルを修正
$ vim /usr/local/prometheus/prometheus.yml

# prometheusのserviceを再起動
$ sudo systemctl restart prometheus
```

<details><summary>`rules.yml`</summary><div>
```yml
groups:
  - name: instance
    rules:
    - alert: InstanceDown
      expr: up == 0       # 監視対象のupが0(down)になり
      for: 1m             # かつその状態が1分以上続くと発火
```
</div></details>

<details><summary>`prometheus.yml`</summary><div>
```yml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
rule_files:
  - rules.yml # アラート設定を読み込む
alerting:
  alertmanagers:
    - static_configs:
      - targets: ['localhost:9093'] # 9093番ポートで起動しているAlertmanagerにアラートを送る
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
    - targets: ['localhost:9100']
```
</div></details>

この時点で`<ラズパイのIP>:9090`をブラウザで開き, `Alerts`を選択するとアラート設定が有効になっていることがわかる.  

![PrometheusのUI画面](/images/2020-01-27-sc06.png)  

ここで試しに監視対象のNode exporterを止めてアラートを発火させてみる.  

```bash
# Node exporterのサービスを止める
$ sudo systemctl stop node_exporter

# アラートの動作確認(発火)が完了した後で再度サービスを起動する
$ sudo systemctl start node_exporter
```

`rules.yml`で定義したとおり,  
監視対象(Node exporter)がダウンして1分経つとアラートが発火する.  

![Prometheusのアラートが発火](/images/2020-01-27-sc07.png)  

また, PrometheusからAlertmanagerにアラートが飛んできていることが確認できる.  

![Alertmanagerの画面](/images/2020-01-27-sc08.png)  

以上でPrometheusのアラート設定は完了.  

次にAlertmanagerでアラートを処理する設定を行う.  
Alertmanagerの設定ファイル`alertmanager.yaml`を修正し,  
アラートが飛んできた際の処理を設定する.  
メール設定の部分に[SMTPサーバーの設定](#smtpサーバーの設定)で確認した項目を書き込む.  
設定の書き方は[ここ](https://prometheus.io/docs/alerting/configuration/#email_config)を参考にする.  

```bash
# Alertmanagerの設定ファイルを編集
$ vim /usr/local/alertmanager/alertmanager.yml

# サービスを再起動
$ sudo systemctl restart alertmanager
```

<details><summary>`alertmanager.yml`</summary><div>
```yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'   # SMTPサーバーのホスト名
  smtp_from: 'xxxxxxxxx@gmail.com'       # 先程設定したアカウントのメールアドレスを送信元に設定
  smtp_auth_username: 'xxxxxxxxx'        # アカウント名
  smtp_auth_password: 'wwwwwwwwwwwwwwww' # アプリパスワード
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: alert-email
receivers:
  - name: alert-email
    email_configs:
      - to: 'yyyyyyyyy@gmail.com'    # 通知を受け取るメールアドレスを設定
```
</div></details>

以上でAlertmanagerからGmail経由で通知メールを送る設定は完了.  

動作確認のため, 先程と同様にNode exporterを再度落としてみる.  

```bash
# Node exporterのサービスを止める
$ sudo systemctl stop node_exporter

# アラートの動作確認(発火)が完了したら再度サービスを起動する
$ sudo systemctl start node_exporter
```

宛先に指定したメールアドレス(`yyyyyyyyy@gmail.com`)の受信トレイを確認すると,  
アラートメールが送信されていることがわかる.  

![届いたメール](/images/2020-01-27-sc09.png)  

やったぜ.  
これでPrometheusのアラートをGmail経由で通知できるようになった.  

## おわり
以上でラズパイ監視のアラートを通知できるようになった.  
いよいよ監視っぽくなってきた...  

## おまけ
かわいいちゃんちゃんこを着るそとちゃん(嫌すぎて2秒で脱いだ)  
![そとちゃん](/images/2020-01-27-sotochan.jpg)  
