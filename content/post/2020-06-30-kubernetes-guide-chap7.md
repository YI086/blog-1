---
title: "Kubernetes完全に理解したい 7章"
date: 2020-06-27T21:57:31+09:00
draft: true
tags: ["Kubernetes", "メモ"]
---

## ServiceとかIngressとか

[Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055)の続き.  
ConfigMapとかSecretとか, Podから利用できるリソースの話.  

<!--more-->
---

## 読んだもの

- [Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055) 7章(Config & Storageリソース)

重要そうなところとかよく使いそうなところだけまとめる.  

## 読んだことのまとめ

- [ConfigMap](#configmap)
- [Secret](#secret)

### ConfigMap

`Pod`から利用できる情報をKey-Value形式で保持するリソース.  
平文ファイル, 直接入力, マニフェストファイル(YAML)の3種類の方法で作成できる.  

{{< highlight bash >}}
# 平文ファイルからConfigMapを作成
$ cat config.txt
abc
def
ghi
1234
$ kubectl create configmap --save-config sample-configmap-01 --from-file=./config.txt
configmap/sample-configmap-01 created

# kubectlで値を直接入力してConfigMapを作成
$ kubectl create configmap --save-config web-config \
    --from-literal=connection.max=100 \
    --from-literal=connection.min=10
configmap/web-config created

# マニフェストファイルから作成する
$ kubectl apply -f sample-configmap.yaml
configmap/sample-configmap created

# 値がKey-Value形式で保存されている
$ kubectl get configmap sample-configmap-01 -o json | jq '.data'
{
  "config.txt": "abc\ndef\nghi\n1234\n"
}
$ kubectl get configmap web-config -o json | jq '.data'
{
  "connection.max": "100",
  "connection.min": "10"
}
$ kubectl get configmap sample-configmap -o json | jq '.data'
{
  "connection.max": "100",
  "connection.min": "10",
  "nginx.conf": "user nginx;\nworker_processes auto;\nerror_log /var/log/nginx/error.log;\npid /run/nginx.pid;\n",
  "sample.properties": "property.1=value-1\nproperty.2=value-2\nproperty.3=value-3\n",
  "thread": "16"
}
{{< /highlight >}}

[sample-configmap.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-configmap.yaml)  

作成した`ConfigMap`は`Pod`から環境変数またはマウントしたファイルとして扱うことができる.  

{{< highlight bash >}}
# ConfigMap(sample-configmap)のValue(Key=connection.max)が環境変数CONNECTION_MAXとして利用されている
$ kubectl get pod sample-configmap-single-env -o json | jq '.spec.containers[].env[]'
{
  "name": "CONNECTION_MAX",
  "valueFrom": {
    "configMapKeyRef": {
      "key": "connection.max",
      "name": "sample-configmap"
    }
  }
}
$ kubectl exec -it sample-configmap-single-env env | grep CONNECTION_MAX
CONNECTION_MAX=100

# ConfigMap(sample-configmap)のKey-ValueがVolume(config-volume)として/configにマウントされている
$ kubectl get pod sample-configmap-multi-volume -o json | jq '.spec.volumes[0]'
{
  "configMap": {
    "defaultMode": 420,
    "name": "sample-configmap"
  },
  "name": "config-volume"
}
$ kubectl get pod sample-configmap-multi-volume -o json | jq '.spec.containers[0].volumeMounts[0]'
{
  "mountPath": "/config",
  "name": "config-volume"
}
$ kubectl exec -it sample-configmap-multi-volume ls /config
connection.max	connection.min	nginx.conf  sample.properties  thread
$ kubectl exec -it sample-configmap-multi-volume cat /config/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
{{< /highlight >}}

[sample-configmap-single-env.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-configmap-single-env.yaml)  
[sample-configmap-multi-volume.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-configmap-multi-volume.yaml)  

### Secret
