---
title: "Kubernetes完全に理解したい 6章"
date: 2020-03-04T22:18:07+09:00
draft: true
tags: ["Kubernetes", "メモ"]
---

## ServiceとかIngressとか

[Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055)の続き.  
次はServiceとかの話.  

<!--more-->
---

## 読んだもの

- [Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055) 6章(Discovery & LBリソース)

重要そうなところとかよく使いそうなところだけまとめる.  

## 読んだことのまとめ

- [Service](#service)
  - [ClusterIP Service](#clusterip-service)
  - [ExternalIP Service](#externalip-service)
  - [NodePort Service](#nodeport-service)

### Service
`Pod`へのトラフィックの負荷分散とサービスディスカバリを行うリソース.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.type`|`Service`の種類を指定する|
|`.spec.ports[]`|トラフィックを受け付けるポートと<br>転送先コンテナのポートに関する設定|
|`.spec.selector`|`Service`の対象となる`Pod`のラベル|

`Service`は指定したラベルを持つ`Pod`へのトラフィックを振り分けたり,  
`Service`名から対象となる`Pod`を探し出したりする.  
`Pod`は作り直すたびにIPが変わってしまうので,  
ラベルで管理する`Service`を使うととても便利.  

#### ClusterIP Service
一番簡単な`Service`. マニフェストの`.spec.type`は`ClusterIP`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.ports[].port`|トラフィックを受け付けるポート|
|`.spec.ports[].targetPort`|転送先`Pod`のポート(コンテナ)|

クラスタ内のみで有効な仮想IPを作成し, トラフィックをコンテナに振り分ける.  
基本的にはクラスタ内でのロードバランサ(負荷分散装置)として使用する.  
```bash
# Deploymentが存在し, Podのラベル(app=sample-app)を指定するClusterIP Serviceが作成済の状態
$ kubectl get deployments
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
sample-deployment   3/3     3            3           80s
$ kubectl get pods -l app=sample-app -o custom-columns="NAME:{metadata.name},IP:{status.podIP}"
NAME                                 IP
sample-deployment-6c5948bf66-c5j2l   10.4.2.40
sample-deployment-6c5948bf66-fwdzj   10.4.1.14
sample-deployment-6c5948bf66-xkn4v   10.4.2.41
$ kubectl get service sample-clusterip
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
sample-clusterip   ClusterIP   10.7.246.109   <none>        8080/TCP   13s

# Service(10.7.246.109)の8080番宛のトラフィックを各Podの80番ポートに転送する設定
$ kubectl describe service sample-clusterip
Name:              sample-clusterip
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     ...
Selector:          app=sample-app
Type:              ClusterIP
IP:                10.7.246.109
Port:              http-port  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.4.1.14:80,10.4.2.40:80,10.4.2.41:80
Session Affinity:  None
Events:            <none>

# ロードバランシングの確認のため, 各Podのnginxで表示するページにPod名を記述
$ for PODNAME in $(kubectl get pods -l app=sample-app -o jsonpath='{.items[*].metadata.name}'); do
    kubectl exec -it ${PODNAME} -- cp /etc/hostname /usr/share/nginx/html/index.html;
  done

# 使い捨てのPodからCurlして確認
# 何回か繰り返すと各Pod名が表示され, 負荷分散されていることがわかる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -s http://10.7.246.109:8080
sample-deployment-6c5948bf66-c5j2l
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -s http://10.7.246.109:8080
sample-deployment-6c5948bf66-fwdzj
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -s http://10.7.246.109:8080
sample-deployment-6c5948bf66-xkn4v
pod "testpod" deleted

# Serviceが存在する状態でPodを再作成してIPが変わってもラベルが同じなら負荷分散の設定が効く
$ kubectl delete deployment sample-deployment
deployment.extensions "sample-deployment" deleted
$ kubectl apply -f sample-deployment.yaml
deployment.apps/sample-deployment created
$ kubectl get pods -l app=sample-app -o custom-columns="NAME:{metadata.name},IP:{status.podIP}"
NAME                                 IP
sample-deployment-6c5948bf66-g4dpz   10.4.2.58
sample-deployment-6c5948bf66-h9qhx   10.4.2.59
sample-deployment-6c5948bf66-vvtl8   10.4.1.15
$ kubectl describe service sample-clusterip | grep Endpoints
Endpoints:         10.4.1.15:80,10.4.2.58:80,10.4.2.59:80

# Podの環境変数からServiceの情報を取得できる(サービスディスカバリ)
$ kubectl exec -it sample-deployment-6c5948bf66-g4dpz printenv | grep SAMPLE_CLUSTERIP
SAMPLE_CLUSTERIP_PORT=tcp://10.7.246.109:8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_PORT=8080
SAMPLE_CLUSTERIP_SERVICE_HOST=10.7.246.109
SAMPLE_CLUSTERIP_SERVICE_PORT=8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_ADDR=10.7.246.109
SAMPLE_CLUSTERIP_SERVICE_PORT_HTTP_PORT=8080
SAMPLE_CLUSTERIP_PORT_8080_TCP=tcp://10.7.246.109:8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_PROTO=tcp

# Service名からIPを正引きできる
# このときのFQDNは<Service名>.<namespace>.svc.cluster.localを指定する
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-clusterip.default.svc.cluster.local
...
;; QUESTION SECTION:
;sample-clusterip.default.svc.cluster.local. IN A

;; ANSWER SECTION:
sample-clusterip.default.svc.cluster.local. 30 IN A 10.7.246.109
...
pod "testpod" deleted

# 逆引きも可能
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig -x 10.7.246.109
...
;; QUESTION SECTION:
;109.246.7.10.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
109.246.7.10.in-addr.arpa. 30	IN	PTR	sample-clusterip.default.svc.cluster.local.
...
pod "testpod" deleted
```
[sample-deployment.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-deployment.yaml)  
[sample-clusterip.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-clusterip.yaml)  

#### ExternalIP Service
`ClusterIP Service`の機能に加えてクラスタ外からのトラフィックもコンテナに振り分ける`Service`.  
マニフェストの`.spec.type`は`ClusterIP`であることに注意.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.externalIPs[]`|トラフィックを受け付ける`Node`のIP|

`ExternalIP Service`ではクラスタの`Node`のIPとポートを指定し,  
そこへ来たトラフィックを`Pod`の任意のポート(コンテナ)に振り分ける.  
`Node`のIPを指定するため,  
`Node`へ疎通できるネットワークからであればクラスタ外からでもクラスタ内のコンテナに疎通できる.  
クラスタ内からのトラフィックについては`ClusterIP Service`と同様に仮想IPで受けて`Pod`に振り分ける.  

```bash
# Podの状態
$ kubectl get pods -o custom-columns="NAME:{.metadata.name},PodIP:{.status.podIP}"
NAME                                 PodIP
sample-deployment-6c5948bf66-9j5jh   10.4.2.3
sample-deployment-6c5948bf66-9md5t   10.4.2.5
sample-deployment-6c5948bf66-ghbrx   10.4.2.4

# Nodeの状態
$ kubectl get nodes -o custom-columns="NAME:{metadata.name},IP:{status.addresses[].address}"
NAME                                                  IP
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   10.138.0.9
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   10.138.0.10
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   10.138.0.8

# ExternalIP Service(sample-externalip)が存在する状態
$ kubectl get services
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP              PORT(S)    AGE
kubernetes          ClusterIP   10.7.240.1     <none>                   443/TCP    93d
sample-externalip   ClusterIP   10.7.241.169   10.138.0.9,10.138.0.10   8080/TCP   50s

# Nodeのうち2つ(10.138.0.9,10.138.0.10)の8080番へのトラフィックを各Podの80番に転送する設定
# クラスタ内からのトラフィックは仮想IP(10.7.241.169:8080)で受ける
$ kubectl describe service sample-externalip
Name:              sample-externalip
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     ...
Selector:          app=sample-app
Type:              ClusterIP
IP:                10.7.241.169
External IPs:      10.138.0.9,10.138.0.10
Port:              http-port  8080/TCP
TargetPort:        80/TCP
Endpoints:         10.4.2.3:80,10.4.2.4:80,10.4.2.5:80
Session Affinity:  None
Events:            <none>

# 今回はクラスタがGKEで提供されているため, 以下のcurlは同じネットワークのインスタンスから実行
# ExternalIPで指定されているNodeのIPとポートを叩くとコンテナへ転送される
$ curl http://10.138.0.9:8080
sample-deployment-6c5948bf66-9md5t
$ curl http://10.138.0.9:8080
sample-deployment-6c5948bf66-9j5jh
$ curl http://10.138.0.9:8080
sample-deployment-6c5948bf66-ghbrx
$ curl http://10.138.0.10:8080
sample-deployment-6c5948bf66-9md5t
$ curl http://10.138.0.10:8080
sample-deployment-6c5948bf66-9j5jh
$ curl http://10.138.0.10:8080
sample-deployment-6c5948bf66-ghbrx
# ExternalIPで指定していないNodeを叩いてもコンテナへ疎通できない
$ curl http://10.138.0.8:8080
curl: (7) Failed to connect to 10.138.0.8 port 8080: Connection refused
```

[sample-externalip.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-externalip.yaml)(`.spec.externalIPs[]`は`Node`に合わせて書き換える)  

### NodePort Service
`ExternalIP Service`と同様にクラスタ外からのトラフィックを受ける`Service`.  
マニフェストの`.spec.type`は`NodePort`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.ports[].nodePort`|クラスタ外からのトラフィックを受けるポートを指定<br>(指定しない場合は自動で決定)<br>すべての`Node`で使用可能なポートを指定する必要がある|

`ExternalIP Service`とは異なり,  
クラスタのすべての`Node`の指定したポートへのトラフィックをクラスタ内の`Pod`に転送する.  

```bash
# Podの状態
$ kubectl get pods -o custom-columns="NAME:{.metadata.name},PodIP:{.status.podIP}"
NAME                                PodIP
sample-deployment-6cd85bd5f-57nsj   10.0.0.6
sample-deployment-6cd85bd5f-5rgtk   10.0.2.3
sample-deployment-6cd85bd5f-bwq9t   10.0.1.3

# NodePort Service(sample-nodeport)が存在する状態
$ kubectl get services
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes        ClusterIP   10.4.0.1      <none>        443/TCP          12d
sample-nodeport   NodePort    10.4.13.251   <none>        8080:30080/TCP   7s

# 各Nodeの30080番へのトラフィックを各Podの80番へ転送する設定
# クラスタ内からのトラフィックは仮想IP(10.4.13.251)の8080番で受ける
$ kubectl describe service sample-nodeport
Name:                     sample-nodeport
Namespace:                default
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            ...
Selector:                 app=sample-app
Type:                     NodePort
IP:                       10.4.13.251
Port:                     http-port  8080/TCP
TargetPort:               80/TCP
NodePort:                 http-port  30080/TCP
Endpoints:                10.0.0.6:80,10.0.1.3:80,10.0.2.3:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# Nodeの外部IPを確認
$ kubectl get nodes -o wide
NAME                              STATUS   ROLES    AGE   VERSION           INTERNAL-IP     EXTERNAL-IP       OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-k8s-01-pool-1-2c84f666-8jwm   Ready    <none>   13h   v1.14.10-gke.17   10.128.15.192   www.www.www.www   Container-Optimized OS from Google   4.14.138+        docker://18.9.7
gke-k8s-01-pool-2-641104a4-7r06   Ready    <none>   14h   v1.14.10-gke.17   10.128.0.60     xxx.xxx.xxx.xxx   Container-Optimized OS from Google   4.14.138+        docker://18.9.7
gke-k8s-01-pool-2-641104a4-gbrv   Ready    <none>   20h   v1.14.10-gke.17   10.128.0.56     yyy.yyy.yyy.yyy   Container-Optimized OS from Google   4.14.138+        docker://18.9.7
gke-k8s-01-pool-2-641104a4-zgjm   Ready    <none>   23h   v1.14.10-gke.17   10.128.0.55     zzz.zzz.zzz.zzz   Container-Optimized OS from Google   4.14.138+        docker://18.9.7
# GKEなのでGCPのインスタンス一覧からでも確認できる
$ gcloud compute instances list
NAME                             ZONE           MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP    EXTERNAL_IP      STATUS
gke-k8s-01-pool-1-2c84f666-8jwm  us-central1-a  f1-micro                    10.128.15.192  www.www.www.www  RUNNING
gke-k8s-01-pool-2-641104a4-7r06  us-central1-a  n1-standard-1  true         10.128.0.60    xxx.xxx.xxx.xxx  RUNNING
gke-k8s-01-pool-2-641104a4-gbrv  us-central1-a  n1-standard-1  true         10.128.0.56    yyy.yyy.yyy.yyy  RUNNING
gke-k8s-01-pool-2-641104a4-zgjm  us-central1-a  n1-standard-1  true         10.128.0.55    zzz.zzz.zzz.zzz  RUNNING

# Nodeの外部IPの指定されたポートを叩くとちゃんと転送される
$ curl http://www.www.www.www:30080
sample-deployment-6cd85bd5f-bwq9t
$ curl http://xxx.xxx.xxx.xxx:30080
sample-deployment-6cd85bd5f-5rgtk
$ curl http://yyy.yyy.yyy.yyy:30080
sample-deployment-6cd85bd5f-57nsj
$ curl http://zzz.zzz.zzz.zzz:30080
sample-deployment-6cd85bd5f-57nsj
```
[sample-nodeport.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-nodeport.yaml)  

### LoadBalancer Service
クラスタ外部のロードバランサーを利用してクラスタ外からのトラフィックを受ける`Service`.  
マニフェストの`.spec.type`は`LoadBalancer`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.ports[].port`|外部ロードバランサーでトラフィックを受けるポートと<br>クラスタ内からのトラフィックを受けるポートを指定|
|`.spec.ports[].nodePort`|`NodePort Service`と同じ|
|`.spec.loadBalancerIP`|利用可能な場合のみ<br>外部ロードバランサーの静的IPを指定|
|`.spec.loadBalancerSourceRanges`|外部ロードバランサーのファイアウォールで<br>許可する通信元の設定<br>(未指定の場合は0.0.0.0/0になる)|

`LoadBalancer Service`クラスタ外にある`LoadBalancer`宛のトラフィックをクラスタの`Node`に対して振り分け,  
`Node`に届いたトラフィックは`NodePort Service`の仕組みで`Pod`に転送される.  

```bash
# Podの状態
$ kubectl get pods -o custom-columns="NAME:{.metadata.name},PodIP:{.status.podIP}"
NAME                                PodIP
sample-deployment-6cd85bd5f-57nsj   10.0.0.6
sample-deployment-6cd85bd5f-5rgtk   10.0.2.3
sample-deployment-6cd85bd5f-bwq9t   10.0.1.3

# LoadBalancer Service(sample-lb)が存在する状態
$ kubectl get services
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
kubernetes   ClusterIP      10.4.0.1      <none>         443/TCP          12d
sample-lb    LoadBalancer   10.4.13.107   34.71.248.88   8080:30082/TCP   5m43s

# 外部ロードバランサーの8080番に届いたトラフィックを各Nodeの30082番に振り分ける設定
# NodePort Serviceを内包しているのでNodeの30082番に届いたトラフィックは各Podの80番に振り分けられる
# クラスタ内からのトラフィックは仮想IP(10.4.13.107)の8080番で受ける
$ kubectl describe service sample-lb
Name:                     sample-lb
Namespace:                default
Labels:                   <none>
Annotations:              kubectl.kubernetes.io/last-applied-configuration:
                            ...
Selector:                 app=sample-app
Type:                     LoadBalancer
IP:                       10.4.13.107
LoadBalancer Ingress:     34.71.248.88
Port:                     http-port  8080/TCP
TargetPort:               80/TCP
NodePort:                 http-port  30082/TCP
Endpoints:                10.0.0.6:80,10.0.1.3:80,10.0.2.3:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age    From                Message
  ----    ------                ----   ----                -------
  Normal  EnsuringLoadBalancer  6m46s  service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   5m47s  service-controller  Ensured load balancer

# GKEの場合はLoadBalancer Serviceの作成時に自動でロードバランサーが払い出される
$ gcloud compute forwarding-rules list
NAME                              REGION       IP_ADDRESS    IP_PROTOCOL  TARGET
a0c33af4a736211ea911a42010a80001  us-central1  34.71.248.88  TCP          us-central1/targetPools/a0c33af4a736211ea911a42010a80001

# 外部ロードバランサーの8080番を叩くとPodに転送される
$ curl http://www.www.www.www:8080
sample-deployment-6cd85bd5f-bwq9t
$ curl http://www.www.www.www:8080
sample-deployment-6cd85bd5f-5rgtk
$ curl http://www.www.www.www:8080
sample-deployment-6cd85bd5f-57nsj
```
[sample-lb.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-lb.yaml)  

### Headless Service
仮想IPではなくDNS Round Robinでエンドポイントを提供する`Service`.  
マニフェストの`.spec.type`は`ClusterIP`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.clusterIP`|`None`のみ指定可能|

`Headless Service`の名前解決時には`<serviceName>.<namespace>.svc.cluster.local`で正引きできる.  
`Pod`が`StatefulSet`の場合のみ, `<podName>.<serviceName>.<namespace>.svc.cluster.local`で`Pod`単位で正引きできる.  

```bash
# Podが存在する状態
$ kubectl get pods -o custom-columns="NAME:{.metadata.name},PodIP:{.status.podIP}"
NAME                            PodIP
sample-statefulset-headless-0   10.0.1.2
sample-statefulset-headless-1   10.0.1.3
sample-statefulset-headless-2   10.0.1.4

# 動作確認用にnginxで表示するHTMLにPod名を記述
$ for PODNAME in $(kubectl get pods -l app=sample-app -o jsonpath='{.items[*].metadata.name}'); do
    kubectl exec -it ${PODNAME} -- cp /etc/hostname /usr/share/nginx/html/index.html;
  done

# Headless Service(sample-headless)が存在する状態
$ kubectl get services
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes        ClusterIP   10.4.0.1     <none>        443/TCP   12d
sample-headless   ClusterIP   None         <none>        80/TCP    19s

# sample-headlessの80番へのトラフィックを各Podの80番に転送する設定
$ kubectl describe service sample-headless
Name:              sample-headless
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     ...
Selector:          app=sample-app
Type:              ClusterIP
IP:                None
Port:              http-port  80/TCP
TargetPort:        80/TCP
Endpoints:         10.0.1.2:80,10.0.1.3:80,10.0.1.4:80
Session Affinity:  None
Events:            <none>

# 使い捨てのPodから名前解決を行うとPodのIPが返ってくる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-headless.default.svc.cluster.local +short
10.0.1.4
10.0.1.3
10.0.1.2
pod "testpod" deleted

# 使い捨てのPodからsample-headlessを叩くとRound Robin方式で各Podへ振り分けられる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -sSL sample-headless.default.svc.cluster.local
sample-statefulset-headless-0
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -sSL sample-headless.default.svc.cluster.local
sample-statefulset-headless-1
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -sSL sample-headless.default.svc.cluster.local
sample-statefulset-headless-2
pod "testpod" deleted

# PodがStatefulSetで管理されている場合のみPod名でも名前解決できる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-statefulset-headless-0.sample-headless.default.svc.cluster.local +short
10.0.1.2
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-statefulset-headless-1.sample-headless.default.svc.cluster.local +short
10.0.1.3
pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-statefulset-headless-2.sample-headless.default.svc.cluster.local +short
10.0.1.4
pod "testpod" deleted
```
[sample-statefulset-headless.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-statefulset-headless.yaml)  
[sample-headless.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-headless.yaml)  

### ExternalName Service
名前解決に対して外部ドメインのCNAMEを返す`Service`.  
マニフェストの`.spec.type`は`ExternalName`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.spec.externalName`|外部ドメインのCNAMEを指定できる|

クラスタ内から外部のサービスに対してアクセスする際は`ExternalName Service`に対して名前解決を行い得られたCNAMEを利用すればいいので,  
`Pod`内に外部サービスのアクセス先情報を持たせる必要がなくなり,  
アクセス先を変える場合は`ExternalName Service`の設定変更のみで行うことができる(疎結合).  
```bash
# ExternalName Service(sample-externalname)が存在する状態
$ kubectl get services
NAME                  TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes            ClusterIP      10.4.0.1     <none>        443/TCP   13d
sample-externalname   ExternalName   <none>       example.com   <none>    5s

# example.comをCNAMEとして返す設定
$ kubectl describe service sample-externalname
Name:              sample-externalname
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     ...
Selector:          <none>
Type:              ExternalName
IP:
External Name:     example.com
Session Affinity:  None
Events:            <none>

# 使い捨てのPodから名前解決するとCNAMEが返ってくる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- dig sample-externalname.default.svc.cluster.local CNAME +short
example.com.
pod "testpod" deleted

# クラスタ内からそのままServiceを叩いてもエラーになる
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -I -sSL sample-externalname.default.svc.cluster.local
HTTP/1.1 404 Not Found
Content-Type: text/html
Date: Wed, 01 Apr 2020 14:54:34 GMT
Server: ECS (ord/4CF4)
Content-Length: 345

pod "testpod" deleted

# ホストヘッダを付与してServiceを叩くとexample.comに接続できる(本来の用途ではないはず...)
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -I -sSL sample-externalname.default.svc.cluster.local -H 'Host:example.com'
HTTP/1.1 200 OK
Accept-Ranges: bytes
Age: 522515
Cache-Control: max-age=604800
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Apr 2020 14:54:39 GMT
Etag: "3147526947+ident"
Expires: Wed, 08 Apr 2020 14:54:39 GMT
Last-Modified: Thu, 17 Oct 2019 07:18:26 GMT
Server: ECS (ord/572A)
X-Cache: HIT
Content-Length: 1256

pod "testpod" deleted
```
[sample-externalname.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-externalname.yaml)(`.spec.externalName`は好きな外部のドメインに書き換える)    

### None-Selector Service
`ExternalName Service`と異なり, 名前解決に対してCNAMEを返すのではなく指定したIPに対してロードバランシングする`Service`.  
マニフェストの`.spec.type`は`ClusterIP`.  

|よく使いそうな設定項目|説明|
|---|---|
|`.metadata.name`(`Endpoint`)|`Service`と同じ名前を指定|
|`.subnets[].addresses[].ip`(`Endpoint`)|ロードバランシングしたいIPを指定|

`.spec.selector`がなく`.spec.type`が`ClusterIP`の`Service`なので,  
仮想IP(ClusterIP)を叩かれたときの転送先の設定を同じ名前の`Endpoints`に定義する必要がある.  
(実は他の`Service`は生成時に自動で対応する`Endpoints`を作っている.)  
ちょっとむずかしいけど, 要はクラスタ内からの通信に対して任意のロードバランサーが作れる仕組み.  

```bash
# None-Selector Service(sample-none-selector)が存在する状態
$ kubectl get services sample-none-selector
NAME                   TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
sample-none-selector   ClusterIP   10.4.2.165   <none>        8080/TCP   12s

# クラスタ内から仮想IP(10.4.2.165)に対して名前解決した場合17.178.96.59(apple.com)か172.217.175.110(google.com)に分ける設定
# あくまでわかりやすくするための例. 本当はちゃんと同じ外部サービスのIPを複数指定するべき.  
$ kubectl describe service sample-none-selector
Name:              sample-none-selector
Namespace:         default
Labels:            <none>
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     ...
Selector:          <none>
Type:              ClusterIP
IP:                10.4.2.165
Port:              <unset>  8080/TCP
TargetPort:        80/TCP
Endpoints:         17.178.96.59:80,172.217.175.110:80
Session Affinity:  None
Events:            <none>

# 手動で作成したEndpoints
$ kubectl get endpoints sample-none-selector
NAME                   ENDPOINTS                            AGE
sample-none-selector   17.178.96.59:80,172.217.175.110:80   78s

# None-Selector Serviceは同じ名前のEndpointsに設定されたIPを参照している
$ kubectl describe endpoints sample-none-selector
Name:         sample-none-selector
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                ...
Subsets:
  Addresses:          17.178.96.59,172.217.175.110
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  80    TCP

Events:  <none>

# 使い捨てのPodから仮想IPを叩くと指定した外部IPに転送される
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -I -s 10.4.2.165:8080
HTTP/1.1 301 Moved Permanently
Location: http://www.google.com:8080/
Content-Type: text/html; charset=UTF-8
Date: Thu, 02 Apr 2020 14:38:32 GMT
Expires: Sat, 02 May 2020 14:38:32 GMT
Cache-Control: public, max-age=2592000
Server: gws
Content-Length: 224
X-XSS-Protection: 0
X-Frame-Options: SAMEORIGIN

pod "testpod" deleted
$ kubectl run --image=centos:6 --restart=Never --rm -i testpod -- curl -I -s 10.4.2.165:8080
HTTP/1.1 301 MOVED PERMANENTLY
Server: Apache
Date: Thu, 02 Apr 2020 14:38:37 GMT
Location: https://www.apple.com/
Content-type: text/html
Connection: close

pod "testpod" deleted
```
[sample-none-selector.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter06/sample-none-selector.yaml)(`Endpoints`の`.subnets[].addresses[].ip`は有効な外部のドメインに書き換える)  

### Ingress
