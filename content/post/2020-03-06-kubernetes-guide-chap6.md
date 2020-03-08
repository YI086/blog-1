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
クラスタ内のみで有効な仮想IPを作成する.  
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
