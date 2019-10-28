---
title: "Hello Minikubeする"
date: 2019-10-28T21:56:44+09:00
draft: true
tags: ["Kubernetes", "作業ログ"]
---

## こんにちはMinikube

Minikubeをインストールしたので, 公式のチュートリアルをやってみた.  

<!--more-->
---

## 動作環境

- masOS Mojave 10.14
- Minikube v1.5.0
- VirtualBox 6.0.12

## やったこと

公式の[Hello Minikube](https://kubernetes.io/docs/tutorials/hello-minikube/)の内容.  
[日本語版](https://kubernetes.io/ja/docs/tutorials/hello-minikube/)もあるので今回はいちいち訳さずに気楽にやってみる.  

- [Minikubeクラスタの作成](#minikubeクラスタの作成)
- [Deploymentの作成](#deploymentの作成)
- [Serviceの作成](#serviceの作成)
- [アドオンの有効化](#アドオンの有効化)
- [クリーンアップ](#クリーンアップ)

## Minikubeクラスタの作成
https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-minikube-cluster  

`Minikube`を起動して, dashboardとやらを開いてみる.  

```bash
# Minikubeを起動する
$ minikube start --vm-driver=virtualbox
😄  minikube v1.5.0 on Darwin 10.14
🔥  Creating virtualbox VM (CPUs=2, Memory=2000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.16.2 on Docker 18.09.9 ...
🚜  Pulling images ...
🚀  Launching Kubernetes ...
⌛  Waiting for: apiserver proxy etcd scheduler controller dns
🏄  Done! kubectl is now configured to use "minikube"
⚠️  /usr/local/bin/kubectl is version 1.14.6, and is incompatible with Kubernetes 1.16.2. You will need to update /usr/local/bin/kubectl or use 'minikube kubectl' to connect with this cluster

# dashboardを起動する
$ minikube dashboard
🔌  Enabling dashboard ...
🤔  Verifying dashboard health ...
🚀  Launching proxy ...
🤔  Verifying proxy health ...
🎉  Opening http://127.0.0.1:53295/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```

勝手にブラウザが動いてdashboardの画面が開く.  

![dashboard画面](/images/2019-10-28-sc01.png)  

なんかいろいろ見られそうで便利そう(小並感)  

バックグラウンド起動にできないのめんどい...  
とりあえず以下はターミナルの別タブでやっていく.  

## Deploymentの作成
https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-deployment  

`Deployment`を作る.  

[Docker Quickstart](https://uzimihsr.github.io/post/2019-10-13-docker-03/)でちょっと触ったけど`Deployment`ってなんだったっけ...?(すっとぼけ)  

>KubernetesのPod は、コンテナの管理やネットワーキングの目的でまとめられた、1つ以上のコンテナのグループです。このチュートリアルのPodがもつコンテナは1つのみです。Kubernetesの Deployment はPodの状態を確認し、Podのコンテナが停止した場合には再起動します。DeploymentはPodの作成やスケールを管理するために推奨される方法(手段)です。  
(https://kubernetes.io/ja/docs/tutorials/hello-minikube/#deployment%E3%81%AE%E4%BD%9C%E6%88%90 より引用)  

なるほど. なんかコンテナがいくつか集まったのが`Pod`で, 複数台の`Pod`をいい感じに管理するのが`Deployment`だったはず.  

今回は以下のファイルで作成された`Docker image`を使って`Deployment`を作成する.  

<details><summary>`server.js`</summary><div>

```javascript
var http = require('http');

var handleRequest = function(request, response) {
  console.log('Received request for URL: ' + request.url);
  response.writeHead(200);
  response.end('Hello World!');
};
var www = http.createServer(handleRequest);
www.listen(8080);
```
https://github.com/kubernetes/website/blob/master/content/en/examples/minikube/server.js

</div></details>

<details><summary>`Dockerfile`</summary><div>

```docker
FROM node:6.14.2
EXPOSE 8080
COPY server.js .
CMD node server.js
```
https://github.com/kubernetes/website/blob/master/content/en/examples/minikube/Dockerfile

</div></details>

```bash
# Deployment(hello-node)を作成する
$ kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node
deployment.apps/hello-node created

# Deploymentの一覧を表示する
$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           40s

# Podの一覧を表示する
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7676b5fb8d-x7l5t   1/1     Running   0          88s

# cluster eventを表示する
$ kubectl get events
LAST SEEN   TYPE     REASON              OBJECT                                MESSAGE
<unknown>   Normal   Scheduled           pod/hello-node-7676b5fb8d-x7l5t       Successfully assigned default/hello-node-7676b5fb8d-x7l5t to minikube
3m17s       Normal   Pulling             pod/hello-node-7676b5fb8d-x7l5t       Pulling image "gcr.io/hello-minikube-zero-install/hello-node"
2m38s       Normal   Pulled              pod/hello-node-7676b5fb8d-x7l5t       Successfully pulled image "gcr.io/hello-minikube-zero-install/hello-node"
2m38s       Normal   Created             pod/hello-node-7676b5fb8d-x7l5t       Created container hello-node
2m38s       Normal   Started             pod/hello-node-7676b5fb8d-x7l5t       Started container hello-node
3m17s       Normal   SuccessfulCreate    replicaset/hello-node-7676b5fb8d      Created pod: hello-node-7676b5fb8d-x7l5t
3m17s       Normal   ScalingReplicaSet   deployment/hello-node                 Scaled up replica set hello-node-7676b5fb8d to 1

# kubectlの設定を表示する
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
- cluster:
    certificate-authority: /Users/uzimihsr/.minikube/ca.crt
    server: https://192.168.99.104:8443
  name: minikube
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-for-desktop
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
- name: minikube
  user:
    client-certificate: /Users/uzimihsr/.minikube/client.crt
    client-key: /Users/uzimihsr/.minikube/client.key
```

`kubectl get events`は初めて打った. 内部で行われている操作がわかって便利そう.  

`kubectl`の設定はDocker for Desktopのぶんの設定も表示しているっぽい.  

以上で`Deployment`の作成ができた.  

## Serviceの作成
https://kubernetes.io/docs/tutorials/hello-minikube/#create-a-service  

`Service`を作る.  

ってか`Service`ってなんだっけ?(すっとぼけ)(2回目)  

>通常、PodはKubernetesクラスタ内部のIPアドレスからのみアクセスすることができます。hello-nodeコンテナをKubernetesの仮想ネットワークの外部からアクセスするためには、KubernetesのServiceとしてポッドを公開する必要があります。  
(https://kubernetes.io/ja/docs/tutorials/hello-minikube/#service%E3%81%AE%E4%BD%9C%E6%88%90 より引用)  

ほーん.  

`Pod`のネットワークまわりの設定をするのが`Service`.  

```bash
# Podをインターネット(外部)からアクセス可能にするためのServiceを作成する
$ kubectl expose deployment hello-node --type=LoadBalancer --port=8080
service/hello-node exposed

# Serviceの一覧を表示する
$ kubectl get services
NAME         TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
hello-node   LoadBalancer   10.105.200.193   <pending>     8080:32547/TCP   71s
kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          101m

# Service(hello-node)にアクセスする
$ minikube service hello-node
|-----------|------------|-------------|-----------------------------|
| NAMESPACE |    NAME    | TARGET PORT |             URL             |
|-----------|------------|-------------|-----------------------------|
| default   | hello-node |             | http://192.168.99.104:32547 |
|-----------|------------|-------------|-----------------------------|
🎉  Opening kubernetes service  default/hello-node in default browser...
```

勝手にブラウザが開き, **server.js** で定義されたHello World!の画面が表示される.  

![Hello World!](/images/2019-10-28-sc02.png)  

以上で`Service`の作成と`Pod`内のコンテナへのアクセスができた.  

ちなみに最初に開いたdashboardを見ると作成したリソースの情報が見られるようになっている. 便利.  

![2019-10-28-sc03.png](/images/2019-10-28-sc03.png)  

## アドオンの有効化
https://kubernetes.io/docs/tutorials/hello-minikube/#enable-addons  

`Minikube`に元々入ってるアドオンが使えるらしいので触ってみる.  

```bash
# 利用可能なアドオンの一覧を表示する
$ minikube addons list
- addon-manager: enabled
- dashboard: enabled
- default-storageclass: enabled
- efk: disabled
- freshpod: disabled
- gvisor: disabled
- heapster: disabled
- helm-tiller: disabled
- ingress: disabled
- ingress-dns: disabled
- logviewer: disabled
- metrics-server: disabled
- nvidia-driver-installer: disabled
- nvidia-gpu-device-plugin: disabled
- registry: disabled
- registry-creds: disabled
- storage-provisioner: enabled
- storage-provisioner-gluster: disabled

# 試しにheapster(監視ツール)を使ってみる
# が, その前にkube-system(namespace)のPodとServiceの一覧を表示する
$ kubectl get pod,svc -n kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
pod/coredns-5644d7b6d9-d2vxs           1/1     Running   0          115m
pod/coredns-5644d7b6d9-rhr9r           1/1     Running   0          115m
pod/etcd-minikube                      1/1     Running   0          114m
pod/kube-addon-manager-minikube        1/1     Running   0          116m
pod/kube-apiserver-minikube            1/1     Running   0          114m
pod/kube-controller-manager-minikube   1/1     Running   0          114m
pod/kube-proxy-f45zs                   1/1     Running   0          115m
pod/kube-scheduler-minikube            1/1     Running   0          114m
pod/storage-provisioner                1/1     Running   0          115m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   116m

# heapsterを有効化する
$ minikube addons enable heapster
✅  heapster was successfully enabled

# 比較のためにPodとServiceを再度確認する
$ kubectl get pod,svc -n kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
pod/coredns-5644d7b6d9-d2vxs           1/1     Running   0          118m
pod/coredns-5644d7b6d9-rhr9r           1/1     Running   0          118m
pod/etcd-minikube                      1/1     Running   0          117m
pod/heapster-h2d7l                     1/1     Running   0          57s
pod/influxdb-grafana-m6c78             2/2     Running   0          57s
pod/kube-addon-manager-minikube        1/1     Running   0          118m
pod/kube-apiserver-minikube            1/1     Running   0          116m
pod/kube-controller-manager-minikube   1/1     Running   0          117m
pod/kube-proxy-f45zs                   1/1     Running   0          118m
pod/kube-scheduler-minikube            1/1     Running   0          117m
pod/storage-provisioner                1/1     Running   0          118m

NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                  AGE
service/heapster              ClusterIP   10.111.191.189   <none>        80/TCP                   57s
service/kube-dns              ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP   118m
service/monitoring-grafana    NodePort    10.102.71.198    <none>        80:30002/TCP             58s
service/monitoring-influxdb   ClusterIP   10.107.121.196   <none>        8083/TCP,8086/TCP        57s

# heapsterを無効化する
$ minikube addons disable heapster
✅  "heapster" was successfully disabled
```

アドオンの有効化/無効化ができた.  

なお[Heapster](https://github.com/kubernetes-retired/heapster)はDeprecatedされている模様...  

## クリーンアップ
https://kubernetes.io/docs/tutorials/hello-minikube/#clean-up

おかたづけする.  

```bash
# リソースの一覧を確認する
$ kubectl get all
NAME                              READY   STATUS    RESTARTS   AGE
pod/hello-node-7676b5fb8d-x7l5t   1/1     Running   0          40m

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/hello-node   LoadBalancer   10.105.200.193   <pending>     8080:32547/TCP   22m
service/kubernetes   ClusterIP      10.96.0.1        <none>        443/TCP          122m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-node   1/1     1            1           40m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-node-7676b5fb8d   1         1         1       40m

# Service(hello-node)を削除する
$ kubectl delete service hello-node
service "hello-node" deleted

# Deployment(hello-node)を削除する
$ kubectl delete deployment hello-node
deployment.apps "hello-node" deleted

# リソースを再確認する
$ kubectl get all
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   125m

# MinikubeのVMを停止する
$ minikube stop
✋  Stopping "minikube" in virtualbox ...
🛑  "minikube" stopped.

# MinikubeのVMを削除する
$ minikube delete
🔥  Deleting "minikube" in virtualbox ...
💔  The "minikube" cluster has been deleted.
🔥  Successfully deleted profile "minikube"
```

以上, Hello Minikubeした.  

## おまけ
お部屋から顔だけ出すそとちゃん  
![にゃーん](/images/2019-10-28-sotochan.jpg)  
