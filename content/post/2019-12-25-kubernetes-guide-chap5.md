---
title: "Kubernetes完全に理解したい 5章"
date: 2019-12-25T22:10:56+09:00
draft: true
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

- aaa

### Pod
`Kubernetes`の最小単位.  
コンテナの起動を担当する.  

1つの`Pod`には複数のコンテナを内包することができ, それらは同じIPを共有する.  
それぞれのコンテナにはポート番号を変えてアクセスできる.  

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
```

[sample-2pod.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/be0185d3970f97b3b8d7aaa2a51c07144ab54171/samples/chapter05/sample-2pod.yaml)  

`Pod`内のコンテナに入って直接操作することもできる.  

```bash
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

# Pod内のredisコンテナに入って環境変数を表示
$ kubectl exec -it sample-2pod -c redis-container -- /bin/sh -c "printenv"
KUBERNETES_PORT=tcp://10.7.240.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=sample-2pod
REDIS_DOWNLOAD_SHA=98c4254ae1be4e452aa7884245471501c9aa657993e0318d88f048093e7f88fd
HOME=/root
TERM=xterm
KUBERNETES_PORT_443_TCP_ADDR=10.7.240.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
REDIS_DOWNLOAD_URL=http://download.redis.io/releases/redis-3.2.12.tar.gz
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT_443_TCP=tcp://10.7.240.1:443
REDIS_VERSION=3.2.12
GOSU_VERSION=1.10
KUBERNETES_SERVICE_HOST=10.7.240.1
PWD=/data
```

コンテナで使用する`Docker image`の`ENTRYPOINT`と`CMD`はそれぞれ`Pod`定義の`spec.container[].command`と`spec.container[].args`で上書きできる.  

```bash
# sample-entrypointはnginx-containerのENTRYPOINTとCMDを上書きしたコンテナを持つPod
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

### ReplicaSet

### Deployment

### DaemonSet

### StatefulSet

### Job

### CronJob
