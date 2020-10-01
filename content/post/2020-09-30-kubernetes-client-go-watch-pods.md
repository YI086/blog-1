---
title: "client-goを使ってKubernetesのPodをクラスタ外からwatchする"
date: 2020-09-30T10:23:13+09:00
draft: false
tags: ["Go", "Kubernetes", "作業ログ"]
---

## Podを見張りたい
Go用のKubernetesクライアントを使ってクラスタ内のPodの様子を見張るやつを作った.  

<!--more-->
---

## やったことのまとめ

- `Kubernetes`クラスタ内の`Pod`をwatchし, `Pod`の状態変化に応じて任意の処理を行う`observer.go`を作った
- `RetryWatcher`を使う方法と`Informer`を使う方法を試したが, `Informer`を使うほうがよさそう

![gif](/images/2020-09-30/sc01.gif)  

[kubernetes-pod-observer](https://github.com/uzimihsr/kubernetes-pod-observer/tree/144aec7d3e38562bd53dae2a5cf5052b086a7533)  

## つかうもの
- macOS Mojave 10.14
- [go](https://github.com/golang/go) version go1.14.6 darwin/amd64
- [kubernetes/client-go](https://github.com/kubernetes/client-go) v0.19.2
- `Kubernetes`クラスタ v1.15.12-gke.20

## やったこと

- [RetryWatcherでwatchする](#retrywatcherでwatchする)
- [Informerを使ってみる](#informerを使ってみる)

### RetryWatcherでwatchする

`Pod`をListするAPI[^1]で`watch`オプションを指定すると, `Pod`の情報(`event`)がリアルタイムで流れてくる.  
この`event`を見ることで, `Pod`が作成/更新/削除されたタイミングを把握することができる.  

{{< highlight bash >}}
# PodのList APIにwatchオプションをつけて実行(kubectlは内部でAPIを叩いている)
$ kubectl get pods -w

# この状態で別ターミナルからJobを実行して削除
$ kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello'
$ kubectl delete job busybox

# kubectl get pods -w の出力
NAME            READY   STATUS    RESTARTS   AGE
busybox-8kzpz   0/1     Pending   0          1s             # Pod作成予約
busybox-8kzpz   0/1     Pending   0          1s             # Pod作成
busybox-8kzpz   0/1     ContainerCreating   0          1s   # コンテナ起動
busybox-8kzpz   0/1     Completed           0          3s   # Pod(コンテナ)完了
busybox-8kzpz   0/1     Terminating         0          12s  # 削除直前の状態
busybox-8kzpz   0/1     Terminating         0          12s  # Pod削除完了
{{< /highlight >}}

今回は`Go`純正の`Kubernetes`クライアントライブラリ[client-go](https://github.com/kubernetes/client-go)を使い,  
`watch`用のAPIを利用して`Pod`をひたすら見張りつづけて状態変化に応じた処理を行うようなしくみを作ってみた.  

{{< highlight bash >}}
# 作業用ディレクトリとgo.modの作成
$ pwd
/path/to/workspace
$ mkdir kubernetes-pod-observer && cd kubernetes-pod-observer
$ go mod init github.com/uzimihsr/kubernetes-pod-observer
$ touch observer.go
{{< /highlight >}}

`observer.go`はこんな感じで作った.  
`Kubernetes API`クライアントの認証情報は`kubeconfig`(~/.kube/config)を参照して,  
`Pod`を`watch`し, `watch`した`event`の内容を元に処理を分けるような形になっている.  
`kubeconfig`読み込みからクライアント立ち上げまでの流れは公式のサンプルコード[^2]を参考にした.  
<script src="https://gist.github.com/uzimihsr/ba86023e96fea6b451339418fa0a3630.js"></script>

少しこだわったのは`Pod`を`watch`する際に`RetryWatcher`[^3]を使用しているところ.  
普通に`watch`用のAPIを叩く場合一定時間が経つとAPIサーバー側が接続を切ってしまう仕様なのだが[^4][^5],  
これを使うと`watch`の接続が切れたときにクライアント側で勝手に再接続してくれる.  

また, `watch`を再開するとすでに観測したことのある`event`も拾ってしまうので,  
`event`に紐づく`ResourceVersion`を参照して既に見たことのある`event`は無視するようにした.  

実際に`observer.go`が動いている状態で, `Pod`を手で作ったり, `Job`から作成してみる.  

{{< highlight bash >}}
# observer.goを実行
$ go run observer.go

# 以下は別のターミナルから実行
# Podを作成(1)
$ kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c 'echo hello'

# Podを削除(2)
$ kubectl delete pod busybox

# Job経由でPodを作成(3)
$ kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello'

# Jobを削除(4)
$ kubectl delete job busybox

{{< /highlight >}}

このときの`observer.go`の出力は次のようになる.  

```
# observer.goの出力

# (1)
ResourceVersion: 60844265
the Pod < busybox > has been created.
ResourceVersion: 60844266
the Pod < busybox >'s phase is < Pending >.
ResourceVersion: 60844267
the Pod < busybox >'s phase is < Pending >.
ResourceVersion: 60844271
the Pod < busybox >'s phase is < Succeeded >.

# (2)
ResourceVersion: 60844359
the Pod < busybox >'s phase is < Succeeded >.
ResourceVersion: 60844360
the Pod < busybox > has been deleted.

# (3)
ResourceVersion: 60844936
the Pod < busybox-dx4f8 > has been created by the Job < busybox >.
ResourceVersion: 60844937
the Pod < busybox-dx4f8 >'s phase is < Pending >.
ResourceVersion: 60844940
the Pod < busybox-dx4f8 >'s phase is < Pending >.
ResourceVersion: 60844948
the Pod < busybox-dx4f8 >'s phase is < Succeeded >.

# (4)
ResourceVersion: 60845063
the Pod < busybox-dx4f8 >'s phase is < Succeeded >.
ResourceVersion: 60845064
the Pod < busybox-dx4f8 > has been deleted.
```

こんな感じで, `watch`した`Pod`の状態に応じた処理を行うことができた.  

### Informerを使ってみる

...実は同じようなことをする公式のサンプルコード[^7]がある,  
こっちは`event`を`watch`する部分で`RetryWatcher`の代わりに, `Informer`[^8]というやつを使っている.  
`Informer`は`watch`に関する処理をさらに抽象化したものらしくて, `watch`が途切れたときに再度接続を復活させたり,  
`watch`した`event`の種類で処理を分けたりと`watch`対象のリソース(`Pod`)の状態変化に応じた処理がしやすくなっている. らしい.  
(`observer.go`では自分で記述していたような処理も内部でやってくれている)  

サンプルコードを真似て`observer.go`を書き直してみる.  
<script src="https://gist.github.com/uzimihsr/21972a6ca3cb66c0f4ff366f8407e545.js"></script>

ポイントは`Informer`の作成時に`event`のType(`ADDED`, `MODIFIED`, `DELETED`)に応じた処理をそれぞれ`AddFunc`, `UpdateFunc`, `DeleteFunc`で設定するところ.  
こうしておくことで, 対象のリソース(`Pod`)の状態が変化したときにそれに応じた処理を設定することができる.  

実際に動かしてみる.  

{{< highlight bash >}}
# observer.goを実行
$ go run observer.go

# 以下は別のターミナルから実行
# Job経由でPodを作成(1)
$ kubectl create job busybox --image=busybox -- /bin/sh -c 'echo hello'

# Jobを削除(2)
$ kubectl delete job busybox
{{< /highlight >}}

{{< highlight bash >}}
# observer.goの出力

# (1)
Pod <busybox-m5cd6> was added by Job <busybox>.
Pod <busybox-m5cd6> is Pending phase.
Pod <busybox-m5cd6> is Pending phase.
Pod <busybox-m5cd6> is Succeeded phase. previous: Pending phase

# (2)
Pod <busybox-m5cd6> is Succeeded phase.
Pod <busybox-m5cd6> was deleted.
{{< /highlight >}}

![gif](/images/2020-09-30/sc01.gif)  

`observer.go`を起動した状態で`Job`を実行し, `Pod`の作成, 状態変化, 削除のタイミングで任意の処理を行うことができた.  
今回はデフォルト`Namespace`の`Pod`名を標準出力しただけだが, 頑張れば特に重要な`Namespace`の`Pod`については変化のタイミングで特定の相手に通知を送ったりとかもできるはず.  

今回は`Pod`の状態変化を見張る部分とそれに応じた処理を1つにしてしまっているが, もちろんサンプルコード[^7]のように`AddFunc`などでキューにデータを送り, 別の`go routine`でキューからデータを取り出して処理することもできる. はず.  
(ここで力尽きたのでやめた)  

## おわり
`Kubernetes`クラスタの外から`Pod`を`watch`して, その状態変化に応じた任意の処理を行うやつを作った.  
`RetryWatcher`を使う方法と`Informer`を使う方法の2通りを試してみたけど, `Informer`を使うほうが`watch`を行う部分について難しいことを考える必要がなくて楽な感じがした.  
どっちを使うべきか迷ったら`Informer`を使うほうがおすすめ?らしい[^9].  

`Informer`はカスタムコントローラとかでも使われているみたいなので[^10], 時間があったらもう少し勉強してみたい.  

## おまけ

ソファを占拠するねこ  
![そとちゃん](/images/2020-09-30/sotochan.jpg)  

## 参考
[^1]: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.18/#list-pod-v1-core  
[^2]: https://github.com/kubernetes/client-go/tree/master/examples/out-of-cluster-client-configuration  
[^3]: https://pkg.go.dev/k8s.io/client-go/tools/watch#RetryWatcher  
[^4]: https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes  
[^5]: https://github.com/kubernetes/client-go/issues/623#issuecomment-506822043  
[^7]: https://github.com/kubernetes/client-go/tree/master/examples/workqueue  
[^8]: https://pkg.go.dev/k8s.io/client-go/tools/cache#NewInformer  
[^9]: https://stackoverflow.com/questions/59544139/kubernetes-client-go-watch-interface-vs-cache-newinformer-vs-cache-newsharedi  
[^10]: https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md  
