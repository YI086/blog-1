---
title: "Kubernetes完全に理解したい 5章"
date: 2020-02-26T22:10:56+09:00
draft: false
tags: ["Kubernetes", "メモ"]
---

## PodとかDeploymentとか

[Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055)の続き.  
Podとかの話.  

<!--more-->
---

## 読んだもの

- [Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055) 5章(Workloadsリソース)

重要そうなところとかよく使いそうなところだけまとめる.  

## 読んだことのまとめ

- [Pod](#pod)
- [ReplicaSet](#replicaset)
- [Deployment](#deployment)
- [DaemonSet](#daemonset)
- [StatefulSet](#statefulset)
- [Job](#job)
- [CronJob](#cronjob)

### Pod
`Kubernetes`の最小単位となるリソース.  
コンテナの起動を担当する.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.containers[].name`|任意のコンテナ名|
|`.spec.containers[].image`|使用するイメージ|
|`.spec.containers[].command`|イメージの`ENTRYPOINT`を上書きできる|
|`.spec.containers[].args`|イメージの`CMD`を上書きできる|
|`.spec.dnsPolicy`|クラスタ外のDNSを利用する場合のみ<br>`"None"`を指定する|
|`.spec.dnsConfig`|`.spec.dnsPolicy`が`"None"`の場合の詳細な設定<br>(`/etc/resolv.conf`に相当)|
|`.spec.hostAliases[]`|全コンテナの`/etc/hosts`を書き換える|

1つの`Pod`には複数のコンテナを内包することができ, それらは同じIPを共有する.  
それぞれのコンテナにはポート番号を変えてアクセスできる.  
また, `Pod`内のコンテナに入って直接操作することもできる.  
```bash
# sample-2podはnginxとredisのコンテナを持つPod
# 2つのコンテナは同じIP(10.4.2.6)を共有している
$ kubectl get pods -o wide
NAME          READY   STATUS    RESTARTS   AGE   IP         NODE                                                  NOMINATED NODE   READINESS GATES
sample-2pod   2/2     Running   0          94s   10.4.2.6   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>

# 確認のためポート転送する
# localhost:8080 -> sample-2pod:80(nginxが使用)
# localhost:8080 -> sample-2pod:6379(redisが使用)
$ kubectl port-forward sample-2pod 8080:80 8081:6379
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Forwarding from 127.0.0.1:8081 -> 6379
Forwarding from [::1]:8081 -> 6379
# 以下の確認が終わり次第Ctrl+Cで終了

# nginxへの疎通を確認
$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

# redisへの疎通を確認
$ curl -s telnet://localhost:8081
set key01 value01
+OK
get key01
$7
value01
quit
+OK

# Pod内のnginxコンテナに入って環境変数を表示
$ kubectl exec -it sample-2pod -c nginx-container -- /bin/sh -c "printenv"
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.7.240.1:443
HOSTNAME=sample-2pod
HOME=/root
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.7.240.1
NGINX_VERSION=1.12.2-1~stretch
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
NJS_VERSION=1.12.2.0.1.14-1~stretch
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.7.240.1:443
KUBERNETES_SERVICE_HOST=10.7.240.1
PWD=/
```

[sample-2pod.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/be0185d3970f97b3b8d7aaa2a51c07144ab54171/samples/chapter05/sample-2pod.yaml)  


コンテナで使用する`Docker image`の`ENTRYPOINT`と`CMD`は`Pod`定義で上書きできる.  

```bash
# .spec.containers[].commandと.spec.containers[].argsを設定したPod
# Dockerfile: CMD:["nginx", "-g", "daemon off;"]
# 上書きした内容: command:["/bin/sleep"] args:["3600"]
# この場合はnginxが立ち上がらずに3600秒sleepする
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
sample-entrypoint   1/1     Running   0          45s

# ポート転送を行った状態(localhost:8080 -> sample-entrypoint:80)でアクセスしてもnginxが起動していないため何も返らない
$ curl localhost:8080
curl: (52) Empty reply from server
```

[sample-entrypoint.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/be0185d3970f97b3b8d7aaa2a51c07144ab54171/samples/chapter05/sample-entrypoint.yaml)  
[Dockerfile(nginx)](https://github.com/nginxinc/docker-nginx/blob/e3bbc1131a683dabf868268e62b9d3fbd250191b/stable/alpine/Dockerfile)  

`Pod`内の全コンテナの`/etc/resolv.conf`, `/etc/hosts`も`Pod`定義で上書きできる.  

```bash
# .spec.dnsConfigを設定したPod
$ kubectl exec -it sample-externaldns cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
search example.com
options ndots:5

# .spec.hostAliasesを設定したPod
$ kubectl exec -it sample-hostaliases cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
fe00::0	ip6-mcastprefix
fe00::1	ip6-allnodes
fe00::2	ip6-allrouters
10.4.2.9	sample-hostaliases

# Entries added by HostAliases.
8.8.8.8	google-dns	google-public-dns
```

[sample-externaldns.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-externaldns.yaml)  
[sample-hostaliases.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-hostaliases.yaml)  

### ReplicaSet
`Pod`のレプリカ(複製)を指定した数だけ維持するリソース.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.replicas`|維持する`Pod`の数|
|`.spec.selector.matchLabels`|維持する`Pod`のラベル<br>基本的に`.spec.template.metadata.labels`と同じものを指定する|
|`.spec.template`|維持する`Pod`の定義|

`ReplicaSet`は生成時に指定した数の`Pod`を作成し, それらをラベルで管理する.  
指定したラベルを持つ`Pod`の数が`ReplicaSet`定義で指定した数より少なくなったときは追加し(セルフヒーリング),  
それよりも多くなった場合は削除することで`Pod`数を維持する.  

```bash
# ReplicaSetが存在している状態
$ kubectl get replicasets -o wide
NAME        DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
sample-rs   3         3         3       72s   nginx-container   nginx:1.12   app=sample-app
# ReplicaSetによって管理されているPod
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS    RESTARTS   AGE
sample-rs-9vh82   1/1     Running   0          6s
sample-rs-f8922   1/1     Running   0          6s
sample-rs-ql2fq   1/1     Running   0          6s

# Podを手動で削除してみる
$ kubectl delete pod sample-rs-9vh82
pod "sample-rs-9vh82" deleted
# Pod数が3より少なくなったためPodが追加される(セルフヒーリング)
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS              RESTARTS   AGE
sample-rs-8f7tm   0/1     ContainerCreating   0          1s
sample-rs-9vh82   0/1     Terminating         0          28s
sample-rs-f8922   1/1     Running             0          28s
sample-rs-ql2fq   1/1     Running             0          28s
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS        RESTARTS   AGE
sample-rs-8f7tm   1/1     Running       0          3s
sample-rs-9vh82   0/1     Terminating   0          30s
sample-rs-f8922   1/1     Running       0          30s
sample-rs-ql2fq   1/1     Running       0          30s
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS    RESTARTS   AGE
sample-rs-8f7tm   1/1     Running   0          9s
sample-rs-f8922   1/1     Running   0          36s
sample-rs-ql2fq   1/1     Running   0          36s

# ReplicaSetの.spec.replicasを2に減らす
$ kubectl scale rs sample-rs --replicas 2
# Pod数が2より多いので削除される
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS        RESTARTS   AGE
sample-rs-8f7tm   0/1     Terminating   0          9m41s
sample-rs-f8922   1/1     Running       0          10m
sample-rs-ql2fq   1/1     Running       0          10m
$ kubectl get pods -l app=sample-app
NAME              READY   STATUS    RESTARTS   AGE
sample-rs-f8922   1/1     Running   0          10m
sample-rs-ql2fq   1/1     Running   0          10m
```
[sample-rs.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-rs.yaml)

### Deployment
複数の`ReplicaSet`を管理するリソース.  
`Pod`と`ReplicaSet`の機能を内包しているので基本的にコンテナを扱うときは`Deployment`を使用する.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.replicas`|`ReplicaSet`と同じ|
|`.spec.selector.matchLabels`|`ReplicaSet`と同じ|
|`.spec.template`|維持する`Pod`の定義|
|`.spec.strategy.type`|ローリングアップデートを使用しない場合のみ<br>`Recreate`を指定する|
|`.spec.strategy.rollingUpdate.maxUnavailable`|ローリングアップデート時に許容できる<br>`Pod`の不足数|
|`.spec.strategy.rollingUpdate.maxSurge`|ローリングアップデート時に許容できる<br>`Pod`の超過数|

`Pod`の更新を行う際は`Deployment`定義で指定した数に対して  
許容できる`Pod`の不足数~超過数の範囲で`Pod`の追加と削除を行う(ローリングアップデート).  
これにより常にいくつかの`Pod`が稼働しつづけるため,  
更新時にダウンタイムが発生しないメリットがある.  
このとき, 新しい`Pod`は同じタイミングで作成される新しい`ReplicaSet`によって管理される.  

```bash
# Deploymentが存在している状態
$ kubectl get deployment -o wide
kubectl get deployment -o wide
NAME                READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES       SELECTOR
sample-deployment   3/3     3            3           11m   nginx-container   nginx:1.12   app=sample-app
$ kubectl get rs -o wide
NAME                           DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES       SELECTOR
sample-deployment-6c5948bf66   3         3         3       17s   nginx-container   nginx:1.12   app=sample-app,pod-template-hash=6c5948bf66
$ kubectl get pods -l app=sample-app
NAME                                 READY   STATUS    RESTARTS   AGE
sample-deployment-6c5948bf66-4tk5h   1/1     Running   0          29s
sample-deployment-6c5948bf66-6stnf   1/1     Running   0          29s
sample-deployment-6c5948bf66-72flm   1/1     Running   0          29s

# コンテナイメージを更新する
$ kubectl set image deployment sample-deployment nginx-container=nginx:1.13
deployment.extensions/sample-deployment image updated
# Podが段階的に更新される(ローリングアップデート)
# 新たにReplicaSet(7b4f67c7bc)が作成され, そこに新しいPodが作成される
$ kubectl get rs --watch
NAME                           DESIRED   CURRENT   READY   AGE
sample-deployment-6c5948bf66   3         3         3       46s
sample-deployment-7b4f67c7bc   1         0         0       0s
sample-deployment-7b4f67c7bc   1         0         0       0s
sample-deployment-7b4f67c7bc   1         1         0       0s
sample-deployment-7b4f67c7bc   1         1         1       2s
sample-deployment-6c5948bf66   2         3         3       55s
sample-deployment-7b4f67c7bc   2         1         1       2s
sample-deployment-6c5948bf66   2         3         3       55s
sample-deployment-6c5948bf66   2         2         2       55s
sample-deployment-7b4f67c7bc   2         1         1       2s
sample-deployment-7b4f67c7bc   2         2         1       2s
sample-deployment-7b4f67c7bc   2         2         2       3s
sample-deployment-6c5948bf66   1         2         2       56s
sample-deployment-7b4f67c7bc   3         2         2       3s
sample-deployment-6c5948bf66   1         2         2       56s
sample-deployment-6c5948bf66   1         1         1       56s
sample-deployment-7b4f67c7bc   3         2         2       4s
sample-deployment-7b4f67c7bc   3         3         2       4s
sample-deployment-7b4f67c7bc   3         3         3       5s
sample-deployment-6c5948bf66   0         1         1       58s
sample-deployment-6c5948bf66   0         1         1       58s
sample-deployment-6c5948bf66   0         0         0       59s
# 更新後
$ kubectl get rs -o wide
NAME                           DESIRED   CURRENT   READY   AGE    CONTAINERS        IMAGES       SELECTOR
sample-deployment-6c5948bf66   0         0         0       108s   nginx-container   nginx:1.12   app=sample-app,pod-template-hash=6c5948bf66
sample-deployment-7b4f67c7bc   3         3         3       55s    nginx-container   nginx:1.13   app=sample-app,pod-template-hash=7b4f67c7bc

# ReplicaSetと同様にスケーリングも可能
$ kubectl scale deployment sample-deployment --replicas 5
deployment.extensions/sample-deployment scaled
$ kubectl get pods -l app=sample-app
NAME                                 READY   STATUS    RESTARTS   AGE
sample-deployment-7b4f67c7bc-28rq4   1/1     Running   0          12m
sample-deployment-7b4f67c7bc-ctzmp   1/1     Running   0          19s
sample-deployment-7b4f67c7bc-gmcvc   1/1     Running   0          12m
sample-deployment-7b4f67c7bc-knx5z   1/1     Running   0          12m
sample-deployment-7b4f67c7bc-vr5m4   1/1     Running   0          19s
```
[sample-deployment.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-deployment.yaml)

### DaemonSet
`Pod`を各Nodeに1個ずつ作成, 管理するリソース.  
監視などに使用される.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.selector.matchLabels`|`ReplicaSet`と同じ|
|`.spec.template`|維持する`Pod`の定義|
|`.spec.updateStrategy.type`|ローリングアップデートを使用しない場合のみ<br>`OnDelete`を指定する|
|`.spec.updateStrategy.rollingUpdate.maxUnavailable`|ローリングアップデート時に許容できる<br>`Pod`の不足数|

各Nodeに`Pod`を1つずつしか配置できないというルール以外,  
`Pod`の管理方法は`ReplicaSet`と同じ.  
```bash
# Nodeの確認
$ kubectl get nodes
NAME                                                  STATUS   ROLES    AGE   VERSION
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   Ready    <none>   66d   v1.13.11-gke.14
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   Ready    <none>   66d   v1.13.11-gke.14
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   Ready    <none>   66d   v1.13.11-gke.14

# DaemonSetがある状態でPodを確認するとPodが各Nodeに1個ずつ存在している
$ kubectl get ds
NAME        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
sample-ds   3         3         3       3            3           <none>          4s
$ kubectl get pods -o wide -l app=sample-app
NAME              READY   STATUS    RESTARTS   AGE     IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-ds-5s6gx   1/1     Running   0          4m16s   10.4.1.10   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-9qs6j   1/1     Running   0          4m16s   10.4.2.17   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-ds-srblh   1/1     Running   0          4m16s   10.4.0.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>

# 試しにPodを削除する
$ kubectl delete pod sample-ds-5s6gx
pod "sample-ds-5s6gx" deleted
# DaemonSetと同様にセルフヒーリングが起こる
$ kubectl get pods -o wide -l app=sample-app --watch
NAME              READY   STATUS              RESTARTS   AGE    IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-ds-5s6gx   1/1     Running             0          6m1s   10.4.1.10   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-9qs6j   1/1     Running             0          6m1s   10.4.2.17   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-ds-srblh   1/1     Running             0          6m1s   10.4.0.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
sample-ds-5s6gx   1/1     Terminating         0          6m14s  10.4.1.10   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-5s6gx   0/1     Terminating         0          6m15s  <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-5s6gx   0/1     Terminating         0          6m21s  <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-5s6gx   0/1     Terminating         0          6m21s  <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-bg2bp   0/1     Pending             0          0s     <none>      <none>                                                <none>           <none>
sample-ds-bg2bp   0/1     Pending             0          0s     <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-bg2bp   0/1     ContainerCreating   0          0s     <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-bg2bp   1/1     Running             0          1s     10.4.1.11   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
$ kubectl get pods -o wide -l app=sample-app
NAME              READY   STATUS    RESTARTS   AGE     IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-ds-9qs6j   1/1     Running   0          11m     10.4.2.17   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-ds-bg2bp   1/1     Running   0          4m50s   10.4.1.11   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-ds-srblh   1/1     Running   0          11m     10.4.0.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
```
[sample-ds.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-ds.yaml)

### StatefulSet
複数の`Pod`に番号をつけて管理し, データの永続化を行うリソース.  
DBなどに使用される.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.replicas`|`ReplicaSet`と同じ|
|`.spec.selector.matchLabels`|`ReplicaSet`と同じ|
|`.spec.template`|維持する`Pod`の定義|
|`.spec.updateStrategy.type`|ローリングアップデートを使用しない場合のみ<br>`OnDelete`を指定する|
|`.spec.updateStrategy.rollingUpdate.partition`|ローリングアップデート時に更新しない<br>`Pod`の数|
|`.spec.volumeClaimTemplates[]`|各`Pod`に紐づく`PersistentVolumeClaim`の定義|

データの永続化には`PersistentVolume`を使用する.  
`PersistentVolume`はPodが無くなっても残り続け,  
セルフヒーリングによって同じ番号のPodが作成された場合は自動でマウントされる.  
```bash
# StatefulSetを作成
$ kubectl apply -f sample-statefulset.yaml
statefulset.apps/sample-statefulset created
# Podが順番に作成される
$ kubectl get pods -o wide -l app=sample-app --watch
NAME                   READY   STATUS              RESTARTS   AGE   IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-statefulset-0   0/1     Pending             0          1s    <none>      <none>                                                <none>           <none>
sample-statefulset-0   0/1     Pending             0          1s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-statefulset-0   0/1     ContainerCreating   0          1s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-statefulset-0   1/1     Running             0          12s   10.4.1.13   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-statefulset-1   0/1     Pending             0          0s    <none>      <none>                                                <none>           <none>
sample-statefulset-1   0/1     Pending             0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     ContainerCreating   0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   1/1     Running             0          10s   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-2   0/1     Pending             0          0s    <none>      <none>                                                <none>           <none>
sample-statefulset-2   0/1     Pending             0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
sample-statefulset-2   0/1     ContainerCreating   0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
sample-statefulset-2   1/1     Running             0          11s   10.4.0.21   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
$ kubectl get pods -o wide -l app=sample-app
NAME                   READY   STATUS    RESTARTS   AGE     IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-statefulset-0   1/1     Running   0          5m58s   10.4.1.13   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-statefulset-1   1/1     Running   0          5m46s   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-2   1/1     Running   0          5m36s   10.4.0.21   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>

# 同時にPersistentVolume(永続化領域)が作成されている
$ kubectl get persistentvolumeclaims
NAME                       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-sample-statefulset-0   Bound    pvc-c325de12-53e6-11ea-9451-42010a8a005a   1Gi        RWO            standard       8m36s
www-sample-statefulset-1   Bound    pvc-cba6642f-53e6-11ea-9451-42010a8a005a   1Gi        RWO            standard       8m21s
www-sample-statefulset-2   Bound    pvc-d3f4bbce-53e6-11ea-9451-42010a8a005a   1Gi        RWO            standard       8m7s
$ kubectl get persistentvolumes
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                              STORAGECLASS   REASON   AGE
pvc-c325de12-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-0   standard                8m48s
pvc-cba6642f-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-1   standard                8m34s
pvc-d3f4bbce-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-2   standard                8m20s

# 試しにPodにマウント(/usr/share/nginx/html)されたPersistentVolumeにファイルを作成
$ kubectl exec -it sample-statefulset-1 touch /usr/share/nginx/html/hoge.txt
$ kubectl exec -it sample-statefulset-0 ls /usr/share/nginx/html
lost+found
$ kubectl exec -it sample-statefulset-1 ls /usr/share/nginx/html
hoge.txt  lost+found
$ kubectl exec -it sample-statefulset-2 ls /usr/share/nginx/html
lost+found
# Podを削除してみる
$ kubectl delete pod sample-statefulset-1
pod "sample-statefulset-1" deleted
# ReplicaSetと同様にセルフヒーリングが起こる
$ kubectl get pods -o wide -l app=sample-app --watch
NAME                   READY   STATUS              RESTARTS   AGE   IP          NODE                                                  NOMINATED NODE   READINESS GATES
sample-statefulset-0   1/1     Running             0          15m   10.4.1.13   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   <none>           <none>
sample-statefulset-1   1/1     Running             0          14m   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-2   1/1     Running             0          14m   10.4.0.21   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   <none>           <none>
sample-statefulset-1   1/1     Terminating         0          15m   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     Terminating         0          15m   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     Terminating         0          15m   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     Terminating         0          15m   10.4.2.19   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     Pending             0          0s    <none>      <none>                                                <none>           <none>
sample-statefulset-1   0/1     Pending             0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   0/1     ContainerCreating   0          0s    <none>      gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-statefulset-1   1/1     Running             0          11s   10.4.2.20   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
# 新しく作成されたPodに同じPersistentVolumeがマウントされるのでファイルが残っている
$ kubectl exec -it sample-statefulset-1 ls /usr/share/nginx/html
hoge.txt  lost+found

# StatefulSetごとPodを全削除してみる
$ kubectl delete statefulsets sample-statefulset
statefulset.apps "sample-statefulset" deleted
# StatefulSetが消えてもPersistentVolumeは残る
$ kubectl get persistentvolumes
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                              STORAGECLASS   REASON   AGE
pvc-c325de12-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-0   standard                22m
pvc-cba6642f-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-1   standard                22m
pvc-d3f4bbce-53e6-11ea-9451-42010a8a005a   1Gi        RWO            Delete           Bound    default/www-sample-statefulset-2   standard                22m
```
[sample-statefulset.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-statefulset.yaml)  

### Job
使い切りの`Pod`を作成するリソース.  
バッチ処理などに使用される.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.completions`|`Pod`が正常終了する回数の上限|
|`.spec.parallelism`|並列に動かす`Pod`の数|
|`.spec.backoffLimit`|`Pod`が失敗する回数の上限|
|`.spec.activeDeadlineSeconds`|`Job`の制限時間|
|`.spec.template`|実行する`Pod`の定義|
|`.spec.template.restartPolicy`|`Pod`失敗時の挙動<br>`Never` : 新たに`Pod`を作成<br>`OnFailure` : 同じ`Pod`を再実行|

`Pod`が正常終了する(終了ステータス0を返す)前提で使用する.  
指定した数の`Pod`が正常終了した場合に`Job`が完了扱いとなる.  
失敗回数の上限や制限時間を超過した場合は`Job`が失敗となる.  
```bash
# 60秒sleepするJobを作成
$ kubectl apply -f sample-job.yaml
job.batch/sample-job created
# Podが起動し60秒後にCompletedとなる
$ kubectl get pods --watch
NAME               READY   STATUS              RESTARTS   AGE
sample-job-7k6bs   0/1     Pending             0          0s
sample-job-7k6bs   0/1     Pending             0          0s
sample-job-7k6bs   0/1     ContainerCreating   0          0s
sample-job-7k6bs   1/1     Running             0          13s
sample-job-7k6bs   0/1     Completed           0          73s

# 30秒sleepするJobを作成
# 合計10回成功するまでPodを2個並列で実行する設定
$ kubectl apply -f sample-paralleljob.yaml
job.batch/sample-paralleljob created
$ kubectl get pods --watch
NAME                       READY   STATUS              RESTARTS   AGE
sample-paralleljob-npscc   0/1     Pending             0          0s
sample-paralleljob-2vlfk   0/1     Pending             0          0s
sample-paralleljob-npscc   0/1     ContainerCreating   0          0s
sample-paralleljob-2vlfk   0/1     ContainerCreating   0          0s
sample-paralleljob-npscc   1/1     Running             0          2s
sample-paralleljob-2vlfk   1/1     Running             0          2s
sample-paralleljob-2vlfk   0/1     Completed           0          33s
sample-paralleljob-262xj   0/1     Pending             0          0s
sample-paralleljob-262xj   0/1     ContainerCreating   0          0s
sample-paralleljob-npscc   0/1     Completed           0          33s
sample-paralleljob-p755p   0/1     Pending             0          0s
sample-paralleljob-p755p   0/1     ContainerCreating   0          0s
sample-paralleljob-p755p   1/1     Running             0          2s
sample-paralleljob-262xj   1/1     Running             0          2s
sample-paralleljob-p755p   0/1     Completed           0          32s
sample-paralleljob-dk5fg   0/1     Pending             0          0s
sample-paralleljob-dk5fg   0/1     ContainerCreating   0          0s
sample-paralleljob-262xj   0/1     Completed           0          32s
sample-paralleljob-jbmdm   0/1     Pending             0          0s
sample-paralleljob-jbmdm   0/1     ContainerCreating   0          0s
sample-paralleljob-jbmdm   1/1     Running             0          3s
sample-paralleljob-dk5fg   1/1     Running             0          3s
sample-paralleljob-jbmdm   0/1     Completed           0          33s
sample-paralleljob-gnmsk   0/1     Pending             0          0s
sample-paralleljob-dk5fg   0/1     Completed           0          33s
sample-paralleljob-6qr2q   0/1     Pending             0          0s
sample-paralleljob-gnmsk   0/1     ContainerCreating   0          0s
sample-paralleljob-6qr2q   0/1     ContainerCreating   0          0s
sample-paralleljob-6qr2q   1/1     Running             0          2s
sample-paralleljob-gnmsk   1/1     Running             0          2s
sample-paralleljob-6qr2q   0/1     Completed           0          33s
sample-paralleljob-6hdkq   0/1     Pending             0          1s
sample-paralleljob-gnmsk   0/1     Completed           0          33s
sample-paralleljob-pp8pz   0/1     Pending             0          0s
sample-paralleljob-6hdkq   0/1     ContainerCreating   0          1s
sample-paralleljob-pp8pz   0/1     ContainerCreating   0          0s
sample-paralleljob-pp8pz   1/1     Running             0          2s
sample-paralleljob-6hdkq   1/1     Running             0          3s
sample-paralleljob-6hdkq   0/1     Completed           0          33s
sample-paralleljob-pp8pz   0/1     Completed           0          32s
```
[sample-job.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-job.yaml)  
[sample-paralleljob.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter05/sample-paralleljob.yaml)  

### CronJob
設定されたスケジュールに基づいて`Job`を実行するリソース.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.schedule`|`Job`を実行するスケジュール|
|`.spec.concurrencyPolicy`|実行中の`Job`が終わる前に<br>次の実行タイミングになったときの挙動<br>`Allow` : 次の`Job`を実行<br>`Forbid` : 次の`Job`を実行しない<br>`Replace` : 実行中の`Job`を中止して次の`Job`を実行|
|`.spec.startingDeadlineSeconds`|開始時刻が`.spec.schedule`より遅れる場合に許容できる秒数|
|`.spec.successfulJobsHistoryLimit`|成功した`Job`を保存する数の上限|
|`.spec.failedJobsHistoryLimit`|失敗した`Job`を保存する数の上限|
|`.spec.suspend`|`CronJob`を停止するかどうかの設定<br>(`true`/`false`)|
|`.spec.jobTemplate`|実行する`Job`の定義|

<br>
```bash
# CronJobを起動
$ kubectl apply -f sample-cronjob.yaml

# .spec.scheduleに設定したタイミングでJobが実行される
$ kubectl get jobs --watch
NAME                        COMPLETIONS   DURATION   AGE
sample-cronjob-1582722300   0/1                      1s
sample-cronjob-1582722300   0/1           1s         1s
sample-cronjob-1582722300   0/1           43s        43s
sample-cronjob-1582722360   0/1                      0s
sample-cronjob-1582722360   0/1           0s         0s
sample-cronjob-1582722300   1/1           85s        85s
sample-cronjob-1582722360   1/1           41s        41s
sample-cronjob-1582722420   0/1                      0s
sample-cronjob-1582722420   0/1           0s         0s
sample-cronjob-1582722420   0/1           42s        42s
sample-cronjob-1582722480   0/1                      1s
sample-cronjob-1582722480   0/1           0s         1s
sample-cronjob-1582722420   0/1           94s        94s
sample-cronjob-1582722480   1/1           42s        43s
sample-cronjob-1582722540   0/1                      0s
sample-cronjob-1582722540   0/1           0s         0s
sample-cronjob-1582722540   1/1           41s        42s
```

## おまけ
しもべのお腹の上には乗ってくれるけどなぜかお尻を向けてくるにゃーにゃ  
![そとちゃんのおしり](/images/2020-02-26-sotochan.jpg)  
