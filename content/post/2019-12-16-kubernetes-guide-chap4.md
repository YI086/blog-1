---
title: "Kubernetes完全に理解したい 4章"
date: 2019-12-16T21:26:15+09:00
draft: false
tags: ["Kubernetes", "メモ"]
---

## 完全に理解したい

[Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055)をまじめに読み始めたのでちょっとずつまとめていきたい.  
1~3章はDockerのおさらいだったりKubernetesとは?みたいな話だったので割愛.  

<!--more-->
---

## 読んだもの

- [Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055) 4章 APIリソースとkubectl

重要そうなところとかよく使いそうなところだけまとめる.  

## 読んだことのまとめ

- `Kubernetes`には役割の異なるたくさんのリソースがある.
- `Namespace`によりクラスタを仮想的に分離できる.
- リソースに関する操作は`Kubernetes API`を通じて行う. CLIなら`kubectl`でいろいろ操作できる.  

### Kubernetesリソースの種類
`Kubernetes`のリソースはだいたい5種類に分けられるらしい.  

#### Workloads  
コンテナを実行する部分に関わるリソース. 基本的に`Pod`と各種`Controller`で構成される.  
重要そうなものだけ以下にまとめる.  

|リソース|概要|
|---|---|
|`Pod`|`Kubernetes`の最小単位|
|`ReplicaSet`|`Pod`のレプリカを複数維持する|
|`Deployment`|`ReplicaSet`を複数管理する|
|`DaemonSet`|各`Node`に`Pod`を1台ずつ維持する|
|`StatefulSet`|ステートフルな`Deployment`と`Pod`を管理する|
|`Job`|回数制限付きの`Pod`の処理を行う|
|`CronJob`|`Job`のスケジューリングを行う|

#### Discovery & LB
コンテナの通信に関わるリソース.  

|リソース|概要|
|---|---|
|`Service`|複数の`Pod`で構成されるアプリケーションを外部に公開するL4レベルのLB|
|`Ingress`|外部から`Service`へのアクセスを可能にするL7レベルのLB|

#### Config & Storage
コンテナの設定ファイルや機密情報, 永続化ボリュームに関するリソース.  

|リソース|概要|
|---|---|
|`Secret`|`Pod`で扱うパスワードや鍵などの機密情報を管理する|
|`ConfigMap`|`Pod`で扱う設定情報を管理する|
|`PersistentVolumeClaim`|`PersistentVolume`(後述)を管理する|

#### Cluster
クラスタ自体の挙動に関するリソース.  

|リソース|概要|
|---|---|
|`Node`|`Master`によって管理されるワーカーマシン|
|`Namespace`|クラスタを仮想的に分離させる|
|`PersistentVolume`|`Pod`から利用できる永続化ボリューム|
|`ResourceQuota`|`Namespace`ごとに使用できるリソースの数に制限をかける|
|`ServiceAccount`|`Namespace`に紐付いたユーザ情報|
|`Role`|`Namespace`レベルで許可される操作の情報|
|`ClusterRole`|クラスタレベルで許可される操作の情報|
|`RoleBinding`|`Namespace`レベルでユーザ情報と`Role`を紐付けて権限管理を行う|
|`ClusterRoleBinding`|クラスタレベルでユーザ情報と`ClusterRole`を紐付けて権限管理を行う|
|`NetworkPolicy`|クラスタ内の`Pod`同士の通信を制限する|

#### Metadata
クラスタ内の他のリソースを操作するためのリソース.  

|リソース|概要|
|---|---|
|`LimitRange`|`Namespace`レベルで`Pod`が使用できるCPUやメモリの制限を行う|
|`HorizontalPodAutoscaler`|`Deployment`または`ReplicaSet`のオートスケーリングを行う|
|`PodDisruptionBudget`|`Node`更新処理の際に維持する`Pod`の数を設定する|
|`CustomRessourceDefinition`|独自のリソースを作って`Kubernetes`を拡張する|

### Namespaceによる仮想クラスタ分離
クラスタ作成時にデフォルトで作成される`Namespace`は3つ.  

- **kube-system**
    - クラスタの各種コンポーネントやアドオンといったシステムがデプロイされる
- **kube-public**
    - 全ユーザが使用可能な`ConfigMap`などの情報が配置される
- **default**
    - ユーザーがデフォルトで使用し, 任意のリソースが作成される

<br>
```bash
# Namespaceの一覧を表示
$ kubectl get namespaces
NAME          STATUS   AGE
default       Active   90m
kube-public   Active   90m
kube-system   Active   90m
```

### kubectl
`kubectl`は`Master`の`kube-apiserver`と通信を行うCLIツール.  
`Kubernetes API`自体はRESTfulであるから代わりに各言語のクライアントライブラリを使うこともできる.  

```bash
# ローカルからAPIにアクセスするためにproxyを別ターミナルで起動
$ kubectl proxy
Starting to serve on 127.0.0.1:8001

# APIを直接叩いてNamespaceの一覧を表示
# $(kubectl get namespaces)と同じ情報を取得できる
$ curl 127.0.0.1:8001/api/v1/namespaces
{
  "kind": "NamespaceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces",
    "resourceVersion": "23058"
  },
  "items": [
    {
      "metadata": {
        "name": "default",
        "selfLink": "/api/v1/namespaces/default",
        "uid": "...",
        "resourceVersion": "25",
        "creationTimestamp": "2019-12-16T12:29:50Z"
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }
    },
    {
      "metadata": {
        "name": "kube-public",
        "selfLink": "/api/v1/namespaces/kube-public",
        "uid": "...",
        "resourceVersion": "35",
        "creationTimestamp": "2019-12-16T12:29:50Z"
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }
    },
    {
      "metadata": {
        "name": "kube-system",
        "selfLink": "/api/v1/namespaces/kube-system",
        "uid": "...",
        "resourceVersion": "166",
        "creationTimestamp": "2019-12-16T12:29:50Z",
        "annotations": {
          "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"v1\",\"kind\":\"Namespace\",\"metadata\":{\"annotations\":{},\"name\":\"kube-system\",\"namespace\":\"\"}}\n"
        }
      },
      "spec": {
        "finalizers": [
          "kubernetes"
        ]
      },
      "status": {
        "phase": "Active"
      }
    }
  ]
}
```

上の例では`curl`で認証情報を渡すのが面倒なのでproxyを使用したが,  
本来`kubectl`が`kube-apiserver`と通信するために必要な認証情報は`kubeconfig(~/.kube/config)`に記述する.  
各種サービスを使用してクラスタを構築した場合はデフォルトで記述されていることが多い.  

```bash
# kubeconfigを確認
$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ...
    server: https://34.83.155.176
  name: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
- cluster:
    certificate-authority: /Users/uzimihsr/.minikube/ca.crt
    server: https://192.168.99.110:8443
  name: minikube
contexts:
- context:
    cluster: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
    user: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
  name: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
kind: Config
preferences: {}
users:
- name: gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
  user:
    auth-provider:
      config:
        access-token: ...
        cmd-args: config config-helper --format=json
        cmd-path: /Users/uzimihsr/google-cloud-sdk/bin/gcloud
        expiry: 2019-12-16T14:59:54Z
        expiry-key: '{.credential.token_expiry}'
        token-key: '{.credential.access_token}'
      name: gcp
- name: minikube
  user:
    client-certificate: /Users/uzimihsr/.minikube/client.crt
    client-key: /Users/uzimihsr/.minikube/client.key
```

`kubectl`では`kubeconfig`の`Context`を切り替えることで異なるクラスタへ接続することができる.  

```bash
# Contextの一覧を表示
# GKE用のContextを使用している
$ kubectl config get-contexts
CURRENT   NAME                                                    CLUSTER                                                 AUTHINFO                                                NAMESPACE
*         gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide   gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide   gke_uzimihsr-01_us-west1-a_uzimihsr-k8s-perfect-guide
          minikube                                                minikube                                                minikube

# 使用するContextを変更
$ kubectl config use-context minikube
Switched to context "minikube".

# 現在のContextを確認
$ kubectl config current-context
minikube
```

`kubectl`とマニフェストファイルを使ってリソースの操作をすることができる.  

```bash
# リソース(Pod)の作成
$ kubectl apply -f sample-pod.yaml
pod/sample-pod created

# Podの一覧表示
$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
sample-pod   1/1     Running   0          18s

# リソースの削除
$ kubectl delete -f sample-pod.yaml
pod "sample-pod" deleted
```
[sample-pod.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter04/sample-pod.yaml)  

リソースにKey-Value形式のラベルをつけることで, 管理がしやすくなる.  
`ReplicaSet`や`Service`ではこのラベルを使用して`Pod`の管理を行っている.  

```bash
# Pod(sample-label)はlabel1(Key)=val1(Value)とlabel2=val2の2つのラベルを持つ
$ kubectl describe pod sample-label
...
Labels:             label1=val1
                    label2=val2
...

# Podの一覧表示
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
sample-label   1/1     Running   0          59s
sample-pod     1/1     Running   0          29s

# ラベル(label1=val1)で絞り込む
$ kubectl get pods -l label1=val1
NAME           READY   STATUS    RESTARTS   AGE
sample-label   1/1     Running   0          5m55s

# ラベルのValueを表示させる
$ kubectl get pods -L label1 -L label2
NAME           READY   STATUS    RESTARTS   AGE     LABEL1   LABEL2
sample-label   1/1     Running   0          7m41s   val1     val2
sample-pod     1/1     Running   0          7m11s
```
[sample-label.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter04/sample-label.yaml)  

`kubectl get`はオプションを付けることで出力形式を指定することができる.  

```bash
# 詳細な情報を付与して一覧表示
$ kubectl get pods -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP         NODE                                                  NOMINATED NODE   READINESS GATES
sample-label   1/1     Running   0          11m   10.4.2.4   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>
sample-pod     1/1     Running   0          10m   10.4.2.5   gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   <none>           <none>

# YAML形式でPod(sample-pod)の情報を出力
$ kubectl get pod sample-pod -o yaml
apiVersion: v1
kind: Pod
...
status:
  ...
  hostIP: 10.138.0.5
  ...
...

# 表示項目を指定して一覧表示
# Podのmetadata.nameの値をNAME, status.hostIPの値をNodeIPとして表示
$ kubectl get pods -o custom-columns="NAME:{.metadata.name},NodeIP:{.status.hostIP}"
NAME           NodeIP
sample-label   10.138.0.5
sample-pod     10.138.0.5
```

`kubectl top`でCPUやメモリの使用量を確認できる.  

```bash
# NodeのCPU, メモリ使用量を確認
$ kubectl top nodes
NAME                                                  CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-0dgf   51m          5%     674Mi           25%
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-n2mf   117m         12%    786Mi           29%
gke-uzimihsr-k8s-perfect-default-pool-e0d8f087-wk64   37m          3%     651Mi           24%

# PodのCPU, メモリ使用量をコンテナ情報つきで確認
$ kubectl top pods --containers
POD            NAME              CPU(cores)   MEMORY(bytes)
sample-label   nginx-container   0m           1Mi
sample-pod     nginx-container   0m           1Mi
```

`kubectl exec`で`Pod`上でコマンドを実行することができる.  

```bash
# Podのコンテナでshを起動
$ kubectl exec -it sample-pod -c nginx-container /bin/sh
# Ctrl+Dまたはexitコマンドで終了

# コマンドに引数がある場合は--の後に指定する
$ kubectl exec -it sample-pod -c nginx-container -- /bin/sh -c "ls --all --classify | grep media"
media/
```

`kubectl logs`で`Pod`のログが確認できる.  

```bash
# コンテナを指定してPodのログを確認
$ kubectl logs sample-pod -c nginx-container
...

# 全コンテナのログを垂れ流す
$ kubectl logs -f sample-pod --all-containers
...

# 期間を指定して出力
# 1時間前から10件分のログをタイムスタンプ付きで表示
$ kubectl logs --since=1h --tail=10 --timestamps=true sample-pod
...
```

`kubectl cp`で`Pod`とローカルマシンでファイルのコピーができる.  

```bash
# Podの/etc/hostnameをローカルにコピー
$ kubectl cp sample-pod:/etc/hostname ./
$ cat ./hostname
sample-pod

# ローカルのファイルをPodの/tmp/newfileにコピー
$ kubectl cp ./hostname sample-pod:/tmp/newfile
$ kubectl exec -it sample-pod -- /bin/sh -c "cat /tmp/newfile"
sample-pod
```

`kubectl port-forward`でポート転送ができる.  

```bash
# localhostの8888番ポートをPodの80番ポートに転送
$ kubectl port-forward sample-pod 8888:80
Forwarding from 127.0.0.1:8888 -> 80
Forwarding from [::1]:8888 -> 80
# 確認できたらCtrl+Cで終了

# 別ターミナルで疎通確認
$ curl -I localhost:8888
HTTP/1.1 200 OK
Server: nginx/1.12.2
...
```

## おまけ
おててしまってお上品に座るそとちゃん  
![そとちゃん](/images/2019-12-16-sotochan.jpg)  
