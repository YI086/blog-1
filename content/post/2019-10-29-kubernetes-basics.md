---
title: "Kubernetesの基本を学ぶ"
date: 2019-11-14T22:57:42+09:00
draft: false
tags: ["Kubernetes", "メモ"]
---

## 基本が大事

Hello Minikubeしたので, チュートリアルの続きを読んでみた.  

<!--more-->
---

## 読んだもの

[Kubernetesの基本を学ぶ](https://kubernetes.io/ja/docs/tutorials/kubernetes-basics/)  
元記事 : [Learn Kubernetes Basics](https://kubernetes.io/docs/tutorials/kubernetes-basics/)

日本語訳がしっかり用意されているので, 読んだ内容を自分の言葉でまとめる.  

## 動作環境

途中のチュートリアルは以下の環境を使用して行った.  

- masOS Mojave 10.14
- Minikube v1.5.0
- VirtualBox 6.0.12

## もくじ

- [Kubernetesの基本を学ぶ](#kubernetesの基本を学ぶ)
- [クラスタの作成](#クラスタの作成)
- [アプリケーションのデプロイ](#アプリケーションのデプロイ)
- [アプリケーションの探索](#アプリケーションの探索)
- [アプリケーションの公開](#アプリケーションの公開)
- [アプリケーションのスケーリング](#アプリケーションのスケーリング)
- [アプリケーションのアップデート](#アプリケーションのアップデート)

## Kubernetesの基本を学ぶ

**`Kubernetes`はどんなことができるの?**  

- コンテナアプリを簡単に実行できるようにしてくれる.  
- そのために必要なリソース, ツールを提供してくれる.  

## クラスタの作成

**`Kubernetes`クラスタとは**  

- クラスタを管理する`Master`とコンテナアプリを動かす`Node`で構成される.
- `Master`はクラスタ内の全ての操作を行う.
    - コンテナが`Node`で実行されるためのスケジューリングを行う.
- `Node`はVMまたは物理マシンで構成され, クラスタのワーカーとして動作する.
    - `Node`を管理し, `Master`と通信するための`Kubelet`を持つ.  
        - 通信は`Kubernetes API`を介して行う.  
    - コンテナを動かすための`Docker`を持つ.  

![Kubernetesクラスタ](/images/2019-11-14-fig01.png)  

<details><summary>**チュートリアル**</summary><div>

```bash
# Minikubeのバージョン確認
$ minikube version
minikube version: v1.5.2
commit: 792dbf92a1de583fcee76f8791cff12e0c9440ad

# Kubernetesクラスタの作成(VirtualBoxを使用)
$ minikube start --vm-driver=virtualbox
😄  minikube v1.5.2 on Darwin 10.14
🔥  Creating virtualbox VM (CPUs=2, Memory=2000MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.16.2 on Docker '18.09.9' ...
🚜  Pulling images ...
🚀  Launching Kubernetes ...
⌛  Waiting for: apiserver
🏄  Done! kubectl is now configured to use "minikube"

# kubectlのバージョン確認
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T23:41:55Z", GoVersion:"go1.12.10", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.2", GitCommit:"c97fe5036ef3df2967d086711e6c0c405941e14b", GitTreeState:"clean", BuildDate:"2019-10-15T19:09:08Z", GoVersion:"go1.12.10", Compiler:"gc", Platform:"linux/amd64"}

# クラスタ情報の確認
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.108:8443
KubeDNS is running at https://192.168.99.108:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

# Nodeの一覧を確認
$ kubectl get nodes
NAME       STATUS   ROLES    AGE    VERSION
minikube   Ready    master   107s   v1.16.2
```

</div></details>

## アプリケーションのデプロイ

**`Deployment`とは**

- アプリケーションインスタンスの作成, 更新方法を`Kubernetes`に指示するもの
    - アプリケーションインスタンスは`Node`に作成される
    - `Master`にある`Deployment Controller`が各インスタンスを監視する
        - 問題があった場合は`セルフヒーリング`を行う(別`Node`のインスタンスと置き換える)

![Deployment](/images/2019-11-14-fig02.png)  

<details><summary>**チュートリアル**</summary><div>

```bash
# Deploymentの作成
# kubernetes-bootcampという名前のDeploymentを作成
# --imageで使用するDocker imageを指定
$ kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
deployment.apps/kubernetes-bootcamp created

# Deploymentの一覧を確認
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           59s

# 別のターミナルを開き, ローカルマシンからKubernetes APIに接続するためのproxyを起動
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
# 使い終わったらCtrl + Cで終了できる

# Kubernetes APIにアクセスしてバージョンを確認
$ curl http://127.0.0.1:8001/version
{
  "major": "1",
  "minor": "16",
  "gitVersion": "v1.16.2",
  "gitCommit": "c97fe5036ef3df2967d086711e6c0c405941e14b",
  "gitTreeState": "clean",
  "buildDate": "2019-10-15T19:09:08Z",
  "goVersion": "go1.12.10",
  "compiler": "gc",
  "platform": "linux/amd64"
}

# 起動しているPodの名前を取得して環境変数POD_NAMEに代入
$ export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
$ echo Name of the Pod: $POD_NAME
Name of the Pod: kubernetes-bootcamp-69fbc6f4cf-6kppt
```

</div></details>

## アプリケーションの探索

**`Pod`とは**

- 1つ以上のコンテナとコンテナの共有リソースを持つ, `Kubernetes`の最小単位
    - 共有ストレージ, IPアドレス, コンテナの動作に関わる情報を持つ
    - `Pod`内のコンテナはIPアドレスとポートを共有する
- `Deployment`によって作成/管理される
- 割り当てられた(スケジュールされた)`Node`上で動作する

**`Node`とは**

- `Kubernetes`のワーカー
    - 物理マシン, 仮想マシン問わずワーカーとして機能できる
- 複数の`Pod`を持つことができる
- `Master`によって管理される
    - 同一クラスタの`Node`間での`Pod`のスケジューリングが自動で管理される
    - `Kubelet`が`Master`との通信を担当し, `Pod`を管理する
    - `Docker`を使用してレジストリからイメージを取得, コンテナの実行を行う

![PodとNode](/images/2019-11-14-fig03.png)  

<details><summary>**チュートリアル**</summary><div>


```bash
# Podの一覧を確認
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          58m

# Podの詳細を確認
$ kubectl describe pods
Name:         kubernetes-bootcamp-69fbc6f4cf-6kppt
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Wed, 06 Nov 2019 23:24:56 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.6
IPs:
  IP:           172.17.0.6
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://64bbc713c49fde8d7fa154fc35b53bcf2515cbf12426248da36b11690a11d2f4
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 06 Nov 2019 23:25:15 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-69fbc6f4cf-6kppt to minikube
  Normal  Pulling    59m        kubelet, minikube  Pulling image "gcr.io/google-samples/kubernetes-bootcamp:v1"
  Normal  Pulled     58m        kubelet, minikube  Successfully pulled image "gcr.io/google-samples/kubernetes-bootcamp:v1"
  Normal  Created    58m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    58m        kubelet, minikube  Started container kubernetes-bootcamp

# proxyは起動済み
# $ kubectl proxy

# Pod名も取得済み
# $ export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
$ echo Name of the Pod: $POD_NAME
Name of the Pod: kubernetes-bootcamp-69fbc6f4cf-6kppt

# Kubernetes APIを利用してPodにアクセスする
# が, この通りにやると失敗する
$ curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
Error trying to reach service: 'dial tcp 172.17.0.6:80: connect: connection refused'

# 上記コマンドでは失敗するので, 次のコマンドを実行する
# 詳細は下記`connection refusedになる問題について`を参照
$ curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-6kppt | v=1
```

</div></details>

<details><summary>**チュートリアルでconnection refusedになる問題について**</summary><div>

使用している`image`(**gcr.io/google-samples/kubernetes-bootcamp:v1**)が公開しているポートが80番でなく8080番であるにもかかわらず,  
`Kubernetes API`の[Get Connect Proxy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#get-connect-proxy-pod-v1-core)で`Pod`にアクセスする際にデフォルトのポート(80番)を叩こうとしていることが原因.  
接続しようとしている`Pod`には80番ポートが割り当てられているコンテナが無いため, 接続が拒否される.  

以下は確認手順.  
```
# 実際にimageをローカルに持ってくる
$ docker pull gcr.io/google-samples/kubernetes-bootcamp:v1
v1: Pulling from google-samples/kubernetes-bootcamp
5c90d4a2d1a8: Pull complete
ab30c63719b1: Pull complete
29d0bc1e8c52: Pull complete
d4fe0dc68927: Pull complete
dfa9e924f957: Pull complete
Digest: sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
Status: Downloaded newer image for gcr.io/google-samples/kubernetes-bootcamp:v1
gcr.io/google-samples/kubernetes-bootcamp:v1

# Dockerfileの内容を確認
$ docker image history gcr.io/google-samples/kubernetes-bootcamp:v1
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
8fafd8af70e9        3 years ago         /bin/sh -c #(nop)  CMD ["/bin/sh" "-c" "node…   0B
<missing>           3 years ago         /bin/sh -c #(nop) COPY file:de8ef36ebbfd5305…   742B
<missing>           3 years ago         /bin/sh -c #(nop)  EXPOSE 8080/tcp              0B
<missing>           3 years ago         /bin/sh -c #(nop) CMD ["node"]                  0B
<missing>           3 years ago         /bin/sh -c buildDeps="xz-utils"     && set -…   41.5MB
<missing>           3 years ago         /bin/sh -c #(nop) ENV NODE_VERSION=6.3.1        0B
<missing>           3 years ago         /bin/sh -c #(nop) ENV NPM_CONFIG_LOGLEVEL=in…   0B
<missing>           3 years ago         /bin/sh -c set -ex   && for key in     9554F…   80.8kB
<missing>           3 years ago         /bin/sh -c apt-get update && apt-get install…   44.7MB
<missing>           3 years ago         /bin/sh -c #(nop) CMD ["/bin/bash"]             0B
<missing>           3 years ago         /bin/sh -c #(nop) ADD file:76679eeb94129df23…   125MB

# 上記の通り, このimageは`EXPOSE 8080`で8080番ポートを公開している.  
# そのため, チュートリアル通りのコマンドではデフォルトの80番ポート(公開されていない)を見ようとするので失敗する.  
$ curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
Error trying to reach service: 'dial tcp 172.17.0.6:80: connect: connection refused'

# Pod名の後にポート番号を指定してあげると問題なくアクセスできる.  
$ curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME:8080/proxy/
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-6kppt | v=1
```

</div></details>


## アプリケーションの公開

**`Service`とは**

- `Pod`を外部に公開するためのもの
- IPを指定せずに`Label`を用いて各`Pod`を指定してアクセスできるようにする役割を持つ
- 複数`Pod`間の負荷分散を行う
    - 複数`Node`間に存在する`Pod`間でも負荷分散が可能

![Service](/images/2019-11-14-fig04.png)  

<details><summary>**チュートリアル**</summary><div>

```bash
# Podの一覧を確認
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          4d22h

# Serviceの一覧を確認
# kubernetesはMinikube起動時にデフォルトで生成されるService
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d22h

# 既に存在するDeployment(kubernetes-bootcamp)を公開するNodePort Serviceを作成
#
$ kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
service/kubernetes-bootcamp exposed

# 再度Serviceの一覧を確認
# 新たにService(kubernetes-bootcamp)が作成されていることを確認
$ kubectl get services
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.96.0.1       <none>        443/TCP          4d22h
kubernetes-bootcamp   NodePort    10.106.55.200   <none>        8080:31038/TCP   17s

# Service(kubernetes-bootcamp)の詳細を確認
# Nodeの31038番ががPodの8080番に割り当てられている
$ kubectl describe services/kubernetes-bootcamp
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   app=kubernetes-bootcamp
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP:                       10.106.55.200
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31038/TCP
Endpoints:                172.17.0.6:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# Serviceによって公開されているNodeのポート番号を環境変数NODE_PORTに代入
$ export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
$ echo NODE_PORT=$NODE_PORT
NODE_PORT=31038

# 公開されているポート番号へアクセス
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-6kppt | v=1

# 起動しているDeploymentの詳細を確認
# デフォルトでLabel(app=kubernetes-bootcamp)がつけられている
$ kubectl describe deployment
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Wed, 06 Nov 2019 23:24:56 +0900
Labels:                 app=kubernetes-bootcamp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=kubernetes-bootcamp
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=kubernetes-bootcamp
  Containers:
   kubernetes-bootcamp:
    Image:        gcr.io/google-samples/kubernetes-bootcamp:v1
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kubernetes-bootcamp-69fbc6f4cf (1/1 replicas created)
Events:          <none>

# Label(run=kubernetes-bootcamp)を指定してPodを表示
$ kubectl get pods -l run=kubernetes-bootcamp
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          4d23h

# 同様にServiceもLabelで絞り込んで表示
$ kubectl get services -l app=kubernetes-bootcamp
NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes-bootcamp   NodePort   10.106.55.200   <none>        8080:31038/TCP   16m

# 環境変数POD_NAMEにPod名を代入
$ export POD_NAME=$(kubectl get pods -o go-template --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
$ echo Name of the Pod: $POD_NAME
Name of the Pod: kubernetes-bootcamp-69fbc6f4cf-6kppt

# Pod(kubernetes-bootcamp-69fbc6f4cf-6kppt)にLabel(app=v1)を追加
# 既にkey(app)に対してvalue(kubernetes-bootcamp)がついていて, --overwriteを指定していないので失敗する
$ kubectl label pod $POD_NAME app=v1
error: 'app' already has a value (kubernetes-bootcamp), and --overwrite is false

# 手順とは異なるが別のkey:valueを指定してみる
$ kubectl label pod $POD_NAME app_hoge=v1
pod/kubernetes-bootcamp-69fbc6f4cf-6kppt labeled

# Pod(kubernetes-bootcamp-69fbc6f4cf-6kppt)の詳細を確認
# Label(app_hoge=v1)が追加されている
$ kubectl describe pods $POD_NAME
Name:         kubernetes-bootcamp-69fbc6f4cf-6kppt
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Wed, 06 Nov 2019 23:24:56 +0900
Labels:       app=kubernetes-bootcamp
              app_hoge=v1
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.6
IPs:
  IP:           172.17.0.6
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://64bbc713c49fde8d7fa154fc35b53bcf2515cbf12426248da36b11690a11d2f4
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 06 Nov 2019 23:25:15 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>

# Label(app_hoge=v1)を持つPodを表示
$ kubectl get pods -l app_hoge=v1
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          4d23h

# Label(app=kubernetes-bootcamp)を指定してServiceを削除
$ kubectl delete service -l app=kubernetes-bootcamp
service "kubernetes-bootcamp" deleted

# 再度Serviceの一覧を確認
# kubernetes-bootcampが削除されている
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   4d23h

# 再度ポートにアクセスする
# Serviceが削除され, 31038番が公開されていないので失敗する
$ curl $(minikube ip):$NODE_PORT
curl: (7) Failed to connect to 192.168.99.109 port 31038: Connection refused

# Podの中からアクセスする
# Serviceがなくてもアプリケーションは生きていることが確認できる
$ kubectl exec -ti $POD_NAME curl localhost:8080
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-6kppt | v=1
```

</div></details>

## アプリケーションのスケーリング

**Deploymentによるスケーリング**

- スケーリングは`Deployment`の`Replicas`を変更することで行う
- `Replicas`が増えた場合は利用可能な`Node`に新しい`Pod`が作成される
- 新たに生成された`Pod`へのトラフィック設定は`Service`で管理する

![スケールアウト](/images/2019-11-14-fig05.png)  

<details><summary>**チュートリアル**</summary><div>

```bash
# Deploymentの一覧を表示
# READYは(起動しているPod数)/(設定されているPod数)
# UP-TO-DATEは指定の状態になっているPod数
# AVAILABLEはユーザーが利用可能なPod数
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   1/1     1            1           5d

# Deployment(kubernetes-bootcamp)のReplicasを4に増やす
$ kubectl scale deployments/kubernetes-bootcamp --replicas=4
deployment.apps/kubernetes-bootcamp scaled

# 再度Deploymentの一覧を表示
# Pod数が指定した値(4)に変わっている
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4/4     4            4           5d

# Podの一覧を表示
# 確かにPodが4つそれぞれ別IPで稼働している
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          5d    172.17.0.6   minikube   <none>           <none>
kubernetes-bootcamp-69fbc6f4cf-9htwb   1/1     Running   0          11m   172.17.0.9   minikube   <none>           <none>
kubernetes-bootcamp-69fbc6f4cf-cm8pb   1/1     Running   0          11m   172.17.0.7   minikube   <none>           <none>
kubernetes-bootcamp-69fbc6f4cf-j5kq2   1/1     Running   0          11m   172.17.0.8   minikube   <none>           <none>

# Deployment(kubernetes-bootcamp)の詳細表示
# Replicasが4に変わっている
# 変更履歴はEventに記録される
$ kubectl describe deployments/kubernetes-bootcamp
Name:                   kubernetes-bootcamp
Namespace:              default
CreationTimestamp:      Wed, 06 Nov 2019 23:24:56 +0900
Labels:                 app=kubernetes-bootcamp
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=kubernetes-bootcamp
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=kubernetes-bootcamp
  Containers:
   kubernetes-bootcamp:
    Image:        gcr.io/google-samples/kubernetes-bootcamp:v1
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   kubernetes-bootcamp-69fbc6f4cf (4/4 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  6m48s  deployment-controller  Scaled up replica set kubernetes-bootcamp-69fbc6f4cf to 4

# 手順にはないが前のチュートリアルでServiceを削除してしまっているので再度作成
$ kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
service/kubernetes-bootcamp exposed

# Service(kubernetes-bootcamp)の詳細を確認
# Deployment(kubernetes-bootcamp)が管理する4つのPodが接続されている
$ kubectl describe services/kubernetes-bootcamp
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   app=kubernetes-bootcamp
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP:                       10.111.26.43
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31189/TCP
Endpoints:                172.17.0.6:8080,172.17.0.7:8080,172.17.0.8:8080 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# 環境変数NODE_PORTに公開されているNodeのポート番号を代入
$ export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
$ echo NODE_PORT=$NODE_PORT
NODE_PORT=31189

# アクセスするたびに違うPodからレスポンスが返ってくる
# 負荷分散ができている
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-6kppt | v=1
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-cm8pb | v=1
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-j5kq2 | v=1
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-j5kq2 | v=1
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-69fbc6f4cf-9htwb | v=1

# Deployment(kubernetes-bootcamp)のReplicasを2に減らす
$ kubectl scale deployments/kubernetes-bootcamp --replicas=2
deployment.apps/kubernetes-bootcamp scaled

# Deploymentの一覧を確認
# Pod数が2に減っている
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2/2     2            2           5d

# Podの一覧を表示
# 4つあったPodのうち2つが消えている
$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          5d    172.17.0.6   minikube   <none>           <none>
kubernetes-bootcamp-69fbc6f4cf-9htwb   1/1     Running   0          19m   172.17.0.9   minikube   <none>           <none>
```

</div></details>

## アプリケーションのアップデート

**ローリングアップデートとは**

- `Pod`を段階的にアップデートすること
    - ダウンタイムが発生しない

![ローリングアップデート](/images/2019-11-14-fig06.png)  

<details><summary>**チュートリアル**</summary><div>

```bash
# 手順にはないが後半の手順とPod数を合わせるためスケールアウトする
$ kubectl scale deployments/kubernetes-bootcamp --replicas=4
deployment.apps/kubernetes-bootcamp scaled

# Deploymentの一覧を確認
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   4/4     4            4           5d23h

# Podの一覧を確認
$ kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6hx6m   1/1     Running   0          57s
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running   0          5d23h
kubernetes-bootcamp-69fbc6f4cf-6rvk4   1/1     Running   0          57s
kubernetes-bootcamp-69fbc6f4cf-9htwb   1/1     Running   0          23h

# Podの詳細を確認
$ kubectl describe pods
Name:         kubernetes-bootcamp-69fbc6f4cf-6hx6m
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:05:57 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.7
IPs:
  IP:           172.17.0.7
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://4a1577f4cc007bbdc62d54afb048b3c67c622e9d4fa4fcbc7bb4e7a7014a8e70
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:05:58 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-69fbc6f4cf-6hx6m to minikube
  Normal  Pulled     91s        kubelet, minikube  Container image "gcr.io/google-samples/kubernetes-bootcamp:v1" already present on machine
  Normal  Created    91s        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    91s        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-69fbc6f4cf-6kppt
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Wed, 06 Nov 2019 23:24:56 +0900
Labels:       app=kubernetes-bootcamp
              app_hoge=v1
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.6
IPs:
  IP:           172.17.0.6
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://64bbc713c49fde8d7fa154fc35b53bcf2515cbf12426248da36b11690a11d2f4
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Wed, 06 Nov 2019 23:25:15 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>


Name:         kubernetes-bootcamp-69fbc6f4cf-6rvk4
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:05:57 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.8
IPs:
  IP:           172.17.0.8
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://124b92bb9f17895f808deee23916dccc6efb95af2fd7e6d8214af7f3ad0bdf97
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:05:58 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-69fbc6f4cf-6rvk4 to minikube
  Normal  Pulled     91s        kubelet, minikube  Container image "gcr.io/google-samples/kubernetes-bootcamp:v1" already present on machine
  Normal  Created    91s        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    91s        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-69fbc6f4cf-9htwb
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Mon, 11 Nov 2019 23:35:24 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=69fbc6f4cf
Annotations:  <none>
Status:       Running
IP:           172.17.0.9
IPs:
  IP:           172.17.0.9
Controlled By:  ReplicaSet/kubernetes-bootcamp-69fbc6f4cf
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://ce75a00dbebf8ede509c4bf2776029666518022ee1b164aaced4c7edc90def8d
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v1
    Image ID:       docker-pullable://gcr.io/google-samples/kubernetes-bootcamp@sha256:0d6b8ee63bb57c5f5b6156f446b3bc3b3c143d233037f3a2f00e279c8fcc64af
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Mon, 11 Nov 2019 23:35:25 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-69fbc6f4cf-9htwb to minikube
  Normal  Pulled     23h        kubelet, minikube  Container image "gcr.io/google-samples/kubernetes-bootcamp:v1" already present on machine
  Normal  Created    23h        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    23h        kubelet, minikube  Started container kubernetes-bootcamp

# Deployment(kubernetes-bootcamp)で使用しているimage(kubernetes-bootcamp)のバージョンを変更
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
deployment.apps/kubernetes-bootcamp image updated

# 何度かPodの一覧を確認すると順番に古いPodが削除/新しいPodが生成されていることがわかる
$ kubectl get pods
NAME                                   READY   STATUS              RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6hx6m   1/1     Terminating         0          2m53s
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Running             0          5d23h
kubernetes-bootcamp-69fbc6f4cf-6rvk4   1/1     Terminating         0          2m53s
kubernetes-bootcamp-69fbc6f4cf-9htwb   1/1     Running             0          23h
kubernetes-bootcamp-b4d9f565-ftfn6     0/1     ContainerCreating   0          6s
kubernetes-bootcamp-b4d9f565-vd6fb     1/1     Running             0          6s
kubernetes-bootcamp-b4d9f565-wwxnf     0/1     ContainerCreating   0          1s
$ kubectl get pods
NAME                                   READY   STATUS        RESTARTS   AGE
kubernetes-bootcamp-69fbc6f4cf-6hx6m   1/1     Terminating   0          2m56s
kubernetes-bootcamp-69fbc6f4cf-6kppt   1/1     Terminating   0          5d23h
kubernetes-bootcamp-69fbc6f4cf-6rvk4   1/1     Terminating   0          2m56s
kubernetes-bootcamp-69fbc6f4cf-9htwb   1/1     Terminating   0          23h
kubernetes-bootcamp-b4d9f565-ftfn6     1/1     Running       0          9s
kubernetes-bootcamp-b4d9f565-qxjhr     1/1     Running       0          2s
kubernetes-bootcamp-b4d9f565-vd6fb     1/1     Running       0          9s
kubernetes-bootcamp-b4d9f565-wwxnf     1/1     Running       0          4s
$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-b4d9f565-ftfn6   1/1     Running   0          48s
kubernetes-bootcamp-b4d9f565-qxjhr   1/1     Running   0          41s
kubernetes-bootcamp-b4d9f565-vd6fb   1/1     Running   0          48s
kubernetes-bootcamp-b4d9f565-wwxnf   1/1     Running   0          43s

# Service(kubernetes-bootcamp)の詳細を確認
$ kubectl describe services/kubernetes-bootcamp
Name:                     kubernetes-bootcamp
Namespace:                default
Labels:                   app=kubernetes-bootcamp
Annotations:              <none>
Selector:                 app=kubernetes-bootcamp
Type:                     NodePort
IP:                       10.111.26.43
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31189/TCP
Endpoints:                172.17.0.10:8080,172.17.0.11:8080,172.17.0.12:8080 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>

# 環境変数NODE_PORTにService(kubernetes-bootcamp)で公開されているポート番号を代入
$ export NODE_PORT=$(kubectl get services/kubernetes-bootcamp -o go-template='{{(index .spec.ports 0).nodePort}}')
$ echo NODE_PORT=$NODE_PORT
NODE_PORT=31189

# 公開されているポートにアクセス
$ curl $(minikube ip):$NODE_PORT
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-b4d9f565-mds8p | v=2

# ローリングアップデートが成功したか確認する
$ kubectl rollout status deployments/kubernetes-bootcamp
deployment "kubernetes-bootcamp" successfully rolled out

# Podの詳細を確認
# Imageがすべてv2に変わっている
$ kubectl describe pods
Name:         kubernetes-bootcamp-b4d9f565-4wkhr
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.11
IPs:
  IP:           172.17.0.11
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://c8bebac0609ee166972beae7437c71733955bfde4107708d28fe3606d68e44d6
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-4wkhr to minikube
  Normal  Pulled     7m23s      kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    7m23s      kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    7m22s      kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-fnkvz
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.10
IPs:
  IP:           172.17.0.10
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://896cc268b334f8241463d9c66af5c7b1accc909c86ea304e95886709b28dedf0
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-fnkvz to minikube
  Normal  Pulled     7m23s      kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    7m23s      kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    7m22s      kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-mds8p
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:14:00 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.12
IPs:
  IP:           172.17.0.12
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://67e064117055eb3050d2fcc7aaeca90fbfa206ce17943c4746103d783e18b4cf
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:01 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-mds8p to minikube
  Normal  Pulled     7m21s      kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    7m21s      kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    7m21s      kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-z7t47
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:14:01 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.13
IPs:
  IP:           172.17.0.13
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://e41971ea038199381f5a2ac385e984b1dfc6c7564101b88d104df9d008cb45cb
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:02 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-z7t47 to minikube
  Normal  Pulled     7m20s      kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    7m20s      kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    7m20s      kubelet, minikube  Started container kubernetes-bootcamp

# 今度はv10を指定してアップデートしてみる
$ kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
deployment.apps/kubernetes-bootcamp image updated

# Deploymentの一覧を確認
# Pod数がおかしいことになっている
$ kubectl get deployments
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   3/4     2            3           5d23h

# Podの一覧を確認
# Statusがおかしいものがある
$ kubectl get pods
NAME                                   READY   STATUS             RESTARTS   AGE
kubernetes-bootcamp-6b4c55d8fc-2rb26   0/1     ImagePullBackOff   0          81s
kubernetes-bootcamp-6b4c55d8fc-z9gn7   0/1     ImagePullBackOff   0          81s
kubernetes-bootcamp-b4d9f565-4wkhr     1/1     Running            0          11m
kubernetes-bootcamp-b4d9f565-fnkvz     1/1     Running            0          11m
kubernetes-bootcamp-b4d9f565-mds8p     1/1     Running            0          11m

# Podの詳細を確認
# Eventsを見るとv10が見つからずエラーとなっている
$ kubectl describe pods
Name:         kubernetes-bootcamp-6b4c55d8fc-2rb26
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:23:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=6b4c55d8fc
Annotations:  <none>
Status:       Pending
IP:           172.17.0.7
IPs:
  IP:           172.17.0.7
Controlled By:  ReplicaSet/kubernetes-bootcamp-6b4c55d8fc
Containers:
  kubernetes-bootcamp:
    Container ID:
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v10
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  <unknown>            default-scheduler  Successfully assigned default/kubernetes-bootcamp-6b4c55d8fc-2rb26 to minikube
  Normal   Pulling    41s (x4 over 2m18s)  kubelet, minikube  Pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"
  Warning  Failed     40s (x4 over 2m16s)  kubelet, minikube  Failed to pull image "gcr.io/google-samples/kubernetes-bootcamp:v10": rpc error: code = Unknown desc = Error response from daemon: manifest for gcr.io/google-samples/kubernetes-bootcamp:v10 not found
  Warning  Failed     40s (x4 over 2m16s)  kubelet, minikube  Error: ErrImagePull
  Warning  Failed     29s (x6 over 2m16s)  kubelet, minikube  Error: ImagePullBackOff
  Normal   BackOff    17s (x7 over 2m16s)  kubelet, minikube  Back-off pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"


Name:         kubernetes-bootcamp-6b4c55d8fc-z9gn7
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:23:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=6b4c55d8fc
Annotations:  <none>
Status:       Pending
IP:           172.17.0.6
IPs:
  IP:           172.17.0.6
Controlled By:  ReplicaSet/kubernetes-bootcamp-6b4c55d8fc
Containers:
  kubernetes-bootcamp:
    Container ID:
    Image:          gcr.io/google-samples/kubernetes-bootcamp:v10
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  <unknown>            default-scheduler  Successfully assigned default/kubernetes-bootcamp-6b4c55d8fc-z9gn7 to minikube
  Normal   BackOff    61s (x6 over 2m17s)  kubelet, minikube  Back-off pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"
  Normal   Pulling    46s (x4 over 2m18s)  kubelet, minikube  Pulling image "gcr.io/google-samples/kubernetes-bootcamp:v10"
  Warning  Failed     45s (x4 over 2m17s)  kubelet, minikube  Failed to pull image "gcr.io/google-samples/kubernetes-bootcamp:v10": rpc error: code = Unknown desc = Error response from daemon: manifest for gcr.io/google-samples/kubernetes-bootcamp:v10 not found
  Warning  Failed     45s (x4 over 2m17s)  kubelet, minikube  Error: ErrImagePull
  Warning  Failed     31s (x7 over 2m17s)  kubelet, minikube  Error: ImagePullBackOff


Name:         kubernetes-bootcamp-b4d9f565-4wkhr
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.11
IPs:
  IP:           172.17.0.11
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://c8bebac0609ee166972beae7437c71733955bfde4107708d28fe3606d68e44d6
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-4wkhr to minikube
  Normal  Pulled     12m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    12m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    12m        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-fnkvz
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.10
IPs:
  IP:           172.17.0.10
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://896cc268b334f8241463d9c66af5c7b1accc909c86ea304e95886709b28dedf0
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-fnkvz to minikube
  Normal  Pulled     12m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    12m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    12m        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-mds8p
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:14:00 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.12
IPs:
  IP:           172.17.0.12
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://67e064117055eb3050d2fcc7aaeca90fbfa206ce17943c4746103d783e18b4cf
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:01 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-mds8p to minikube
  Normal  Pulled     12m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    12m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    12m        kubelet, minikube  Started container kubernetes-bootcamp

# 直前のローリングアップデートを巻き戻す
$ kubectl rollout undo deployments/kubernetes-bootcamp
deployment.apps/kubernetes-bootcamp rolled back

# Podの一覧を確認
# Statusが正常に戻っている
$ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
kubernetes-bootcamp-b4d9f565-4wkhr   1/1     Running   0          13m
kubernetes-bootcamp-b4d9f565-b9t5c   1/1     Running   0          4s
kubernetes-bootcamp-b4d9f565-fnkvz   1/1     Running   0          13m
kubernetes-bootcamp-b4d9f565-mds8p   1/1     Running   0          13m

# Podの詳細を確認
$ kubectl describe pods
Name:         kubernetes-bootcamp-b4d9f565-4wkhr
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.11
IPs:
  IP:           172.17.0.11
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://c8bebac0609ee166972beae7437c71733955bfde4107708d28fe3606d68e44d6
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-4wkhr to minikube
  Normal  Pulled     16m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    16m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    16m        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-b9t5c
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:27:35 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.8
IPs:
  IP:           172.17.0.8
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://644cf4f577f0a12f4e43d2c8ab98ab587cb231f5f9dbd41a48885c4255e146cb
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:27:36 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-b9t5c to minikube
  Normal  Pulled     2m46s      kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    2m46s      kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    2m46s      kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-fnkvz
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:13:59 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.10
IPs:
  IP:           172.17.0.10
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://896cc268b334f8241463d9c66af5c7b1accc909c86ea304e95886709b28dedf0
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:00 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-fnkvz to minikube
  Normal  Pulled     16m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    16m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    16m        kubelet, minikube  Started container kubernetes-bootcamp


Name:         kubernetes-bootcamp-b4d9f565-mds8p
Namespace:    default
Priority:     0
Node:         minikube/192.168.99.109
Start Time:   Tue, 12 Nov 2019 23:14:00 +0900
Labels:       app=kubernetes-bootcamp
              pod-template-hash=b4d9f565
Annotations:  <none>
Status:       Running
IP:           172.17.0.12
IPs:
  IP:           172.17.0.12
Controlled By:  ReplicaSet/kubernetes-bootcamp-b4d9f565
Containers:
  kubernetes-bootcamp:
    Container ID:   docker://67e064117055eb3050d2fcc7aaeca90fbfa206ce17943c4746103d783e18b4cf
    Image:          jocatalin/kubernetes-bootcamp:v2
    Image ID:       docker-pullable://jocatalin/kubernetes-bootcamp@sha256:fb1a3ced00cecfc1f83f18ab5cd14199e30adc1b49aa4244f5d65ad3f5feb2a5
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 12 Nov 2019 23:14:01 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-jj2nf (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-jj2nf:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-jj2nf
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned default/kubernetes-bootcamp-b4d9f565-mds8p to minikube
  Normal  Pulled     16m        kubelet, minikube  Container image "jocatalin/kubernetes-bootcamp:v2" already present on machine
  Normal  Created    16m        kubelet, minikube  Created container kubernetes-bootcamp
  Normal  Started    16m        kubelet, minikube  Started container kubernetes-bootcamp
```

</div></details>

## おわり
おわり.  
自分で絵を書くのがめんどくさかったけどなんとなく理解できた. 気がする.  
