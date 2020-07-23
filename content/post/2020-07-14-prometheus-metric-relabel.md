---
title: "Prometheusで不要なメトリクスを除外する"
date: 2020-07-14T22:49:32+09:00
draft: false
tags: ["メモ", "Raspberry Pi", "Prometheus"]
---

## いらないメトリクスは拾いたくない
Prometheusでスクレイプする対象にいらないメトリクスがあったときに無視したかった.  

<!--more-->
---

## まとめ
特定のメトリクスだけ除外したいときは`metric_relabel_configs`で除外したいメトリクスのラベルを正規表現で記述してdropするのが良さそう.  


{{< highlight yaml >}}
# メトリクス名がnode_cpu_seconds_totalでmode="idle"のラベルを持つメトリクスを除外する例
scrape_configs:
  - job_name: 'node'
    static_configs:
    - targets: ['localhost:9100']
    metric_relabel_configs:
    - source_labels: [__name__, mode]
      regex: 'node_cpu_seconds_total;idle'
      action: drop
{{< /highlight >}}

## 環境
- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
    - 監視サーバとして使用
- [Prometheus](https://github.com/prometheus/prometheus) version 2.15.2
    - [インストール済み](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)
- [Node exporter](https://github.com/prometheus/node_exporter) version 0.18.1
    - [インストール済み](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)

## やりかた

例えば`Node exporter`で取れるCPU時間のメトリクス`node_cpu_seconds_total`を`Prometheus`で見ているとき,  
普通はこんな感じで`mode`ラベルの値が異なる複数のメトリクスが取れる.  

```
node_cpu_seconds_total{cpu="0"}
```

![Prometheus](/images/2020-07-14/sc01.png)  

メトリクスの用途はさておき, この中で`mode="idle"`のラベルがついているメトリクスだけを除外したいとする.  

そんなときは`Prometheus`のスクレイプ設定で`metric_relabel_configs`の設定を追加する.  
この設定により, `__name__`ラベル(メトリクス名)が`node_cpu_seconds_total`かつ`mode`ラベルが`idle`であるメトリクスだけが除外(drop)される.  

<script src="https://gist.github.com/uzimihsr/47cbde957d57c5fc8a78d02e681460f3.js"></script>

設定変更後, `Prometheus`を再起動する.  

{{< highlight bash >}}
# prometheusのスクレイプ設定を変更して再起動
$ vim /usr/local/prometheus/prometheus.yml
$ sudo systemctl restart prometheus
{{< /highlight >}}

再度`Prometheus`で先程と同じクエリを投げると, `mode="idle"`のラベルがついたメトリクスだけがなくなっていることがわかる.  

![Prometheus](/images/2020-07-14/sc02.png)  

今度は逆に`mode="idle"`のラベルがついているメトリクスだけを残して, 他のメトリクスを拾わないようにしてみる.  
この設定により, `__name__="node_cpu_seconds_total"`と`mode="idle"`の両方のラベルを持つメトリクスのみが保持(keep)され,  
それ以外のメトリクスがすべて除外される.  
<script src="https://gist.github.com/uzimihsr/cb14c21ba82985b58834994f20f3f100.js"></script>

![Prometheus](/images/2020-07-14/sc03.png)  

これでも問題なさそうだけど, 実はこの設定を入れるとメトリクス名(`__name__`)が`node_cpu_seconds_total`でないメトリクスがすべて捨てられてしまうので,  
`node_filesystem_free_bytes`(ファイルシステムの空き容量)のような`Node exporter`の他のメトリクスが取れなくなってしまう.  

![Prometheus](/images/2020-07-14/sc04.png)  

うまいこと`__name__="node_cpu_seconds_total"`かつ`mode!="idle"`のような条件を正規表現で書ければいいんだけど,  
`Prometheus`で使うRE2正規表現は`(?!idle)`みたいな否定先読みに対応していないらしいのでちょっと難しい.  

よくわかんなかったので苦肉の策として除外したい個別のラベルを列挙してみた.  
<script src="https://gist.github.com/uzimihsr/ee45b5d477ac9f415db214cca5db9f54.js"></script>

これなら`__name__="node_filesystem_free_bytes"`のような他のメトリクスを捨てずに`__name__="node_cpu_seconds_total"`のメトリクスの中で`mode="idle"`なものだけを残せるけど,  
除外するラベルを全部列挙するのはあんまり現実的じゃないような気がする...  

![Prometheus](/images/2020-07-14/sc03.png)  
![Prometheus](/images/2020-07-14/sc05.png)  

## おわり
正規表現にマッチするメトリクスだけ除外するdropとその逆のkeepだと本当は除外対象を列挙しないで済むkeepを使いたくなるけど,  
現状は正規表現の仕様でdropのほうが使いやすいという感じがする.  
そもそもそこまで排除対象のメトリクスが多いようなスクレイプ対象だったらそれ自体を見直したほうがいいのだろうか.  

とりあえずはdropで頑張ってみて, keepでいい感じに他のメトリクスを残せるような方法があったら試してみたい.  

## おまけ
人間用まくらが好きなねこ  
![そとちゃん](/images/2020-07-14/sotochan.jpg)  

## 参考

- https://www.robustperception.io/dropping-metrics-at-scrape-time-with-prometheus
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config
- https://github.com/google/re2/wiki/Syntax
