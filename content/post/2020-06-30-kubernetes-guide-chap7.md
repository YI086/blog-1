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
- [Volume](#volume)

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

`ConfigMap`とは異なり, 秘密情報を扱うためのリソース.  
対応する`Secret`を利用する`Pod`がある場合のみ`etcd`から`Node`の一時的な領域(`tmpfs`)にKey-Valueのデータが送られるようになっているので, `ConfigMap`に比べて機密性が高い.  
また, 安全のためにKey-ValueのValueがbase64エンコードされていて少し見えにくくなっている.  

`Opaque`タイプの`Secret`は基本的に`ConfigMap`と同じような使い方ができる.  
{{< highlight bash >}}
# 平文ファイルからSecretを作成
$ echo -n "root" > ./username
$ echo -n "rootpassword" > ./password
$ kubectl create secret generic --save-config sample-db-auth-from-file \
    --from-file=./username --from-file=./password
secret/sample-db-auth-from-file created

# envfileからSecretを作成
$ cat << 'EOF' > env-secret.txt
username=root
password=rootpassword
EOF
$ kubectl create secret generic --save-config sample-db-auth-from-env-file \
    --from-env-file ./env-secret.txt
secret/sample-db-auth-from-env-file created

# kubectlで値を直接入力してSecretを作成
$ kubectl create secret generic --save-config sample-db-auth-from-literal \
    --from-literal=username=root --from-literal=password=rootpassword
secret/sample-db-auth-from-literal created

# マニフェストファイルからSecretを作成
$ kubectl apply -f sample-db-auth.yaml
secret/sample-db-auth created

# describeしても見えない
$ kubectl describe secret sample-db-auth-from-literal
Name:         sample-db-auth-from-literal
Namespace:    default
Labels:       <none>
Annotations:
Type:         Opaque

Data
====
password:  12 bytes
username:  4 bytes

# 中身はbase64エンコードされている
$ kubectl get secret sample-db-auth-from-literal -o json | jq '.data'
{
  "password": "cm9vdHBhc3N3b3Jk",
  "username": "cm9vdA=="
}
$ kubectl get secret sample-db-auth-from-literal -o json | jq -r '.data.username' | base64 -D
root
$ kubectl get secret sample-db-auth-from-literal -o json | jq -r '.data.password' | base64 -D
rootpassword
{{< /highlight >}}

[sample-db-auth.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-db-auth.yaml)  

`Ingress`などからTLSに必要な証明書と秘密鍵を扱うための`kubernetes.io/tls`タイプの`Secret`もある.  

{{< highlight bash >}}
# 秘密鍵(tls.key)とオレオレ証明書(tls.crt)を同時に作成
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout ./tls.key -out ./tls.crt -subj "/CN=sample1.example.com"
Generating a 2048 bit RSA private key
......+++
.........................................................................................................................................................................................................+++
writing new private key to './tls.key'
-----
$ ls
tls.crt tls.key

# 秘密鍵と証明書からSecretを作成
$ kubectl create secret tls --save-config tls-sample --key ./tls.key --cert ./tls.crt
secret/tls-sample created

# 秘密鍵と証明書のpemをbase64エンコードしたものが登録されている
$ kubectl get secret tls-sample -o json | jq '.data'
{
  "tls.crt": "LS0tLS...tLS0K",
  "tls.key": "LS0tLS...tLS0K"
}
$ kubectl get secret tls-sample -o json | jq -r .data'["tls.crt"]' | base64 -D | openssl x509 -noout -text
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 12508610311645361173 (0xad9782ce1368f415)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=sample1.example.com
        Validity
            Not Before: Jul  8 14:54:37 2020 GMT
            Not After : Jul  8 14:54:37 2021 GMT
        Subject: CN=sample1.example.com
        ...
{{< /highlight >}}

`DockerHub`のプライベートリポジトリなど, 認証のかかった`Docker`レジストリから`image`を取得するための認証情報を扱う場合は`kubernetes.io/dockerconfigjson`タイプの`Secret`を使用する.  
`Pod`から認証がかかったリポジトリの`image`をpullするにはマニフェストファイルの`.spec.imagePullSecrets`に`Secret`名を指定する(例 : [sample-pull-secret.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-pull-secret.yaml)).  

{{< highlight bash >}}
# 認証情報からSecretを作成
$ kubectl create secret docker-registry --save-config sample-registry-auth \
    --docker-server=SERVER \
    --docker-username=USER \
    --docker-password=PASSWORD \
    --docker-email=EMAIL
secret/sample-registry-auth created

# dockercfg形式の認証情報が保存されている
$ kubectl get secret sample-registry-auth -o json | jq -r .data'[".dockerconfigjson"]' | base64 -D
{"auths":{"SERVER":{"username":"USER","password":"PASSWORD","email":"EMAIL","auth":"VVNFUjpQQVNTV09SRA=="}}}
{{< /highlight >}}

作成した`Secret`は`Pod`から環境変数またはマウントしたファイルとして扱うことができる.  

{{< highlight bash >}}
# Secret(sample-db-auth)のValue(Key=username)が環境変数DB_USERNAMEとして利用されている
$ kubectl get pod sample-secret-single-env -o json | jq '.spec.containers[].env[]'
{
  "name": "DB_USERNAME",
  "valueFrom": {
    "secretKeyRef": {
      "key": "username",
      "name": "sample-db-auth"
    }
  }
}
$ kubectl exec -it sample-secret-single-env env | grep DB_USERNAME
DB_USERNAME=root

# Secret(sample-db-auth)のKey-ValueがVolume(config-volume)として/configにマウントされている
$ kubectl get pod sample-secret-multi-volume -o json | jq '.spec.volumes[0]'
{
  "name": "config-volume",
  "secret": {
    "defaultMode": 420,
    "secretName": "sample-db-auth"
  }
}
$ kubectl get pod sample-secret-multi-volume -o json | jq '.spec.containers[].volumeMounts[0]'
{
  "mountPath": "/config",
  "name": "config-volume"
}
$ kubectl exec -it sample-secret-multi-volume ls /config
password  username
$ kubectl exec -it sample-secret-multi-volume cat /config/username
root
{{< /highlight >}}

### Volume

`Pod`からディスクを扱うためのリソース.  
いくつか種類があるが, どれも`Pod`上に静的に領域を指定して使用する.  

`emptyDir`は`Pod`に一時的なディスク領域を作成し, `Pod`が終了すると同時に削除される.  
`Pod`内の複数のコンテナ間でファイルを共有したりするのに使える.  
{{< highlight bash >}}
# emptyDirが /cache にマウントされている
$ kubectl get pod sample-emptydir -o json | jq '.spec.volumes[0]'
{
  "emptyDir": {},
  "name": "cache-volume"
}
$ kubectl get pod sample-emptydir -o json | jq '.spec.containers[0].volumeMounts[0]'
{
  "mountPath": "/cache",
  "name": "cache-volume"
}
# 中身は空
$ kubectl exec -it sample-emptydir -- ls -la /cache
total 8
drwxrwxrwx 2 root root 4096 Jul  9 14:26 .
drwxr-xr-x 1 root root 4096 Jul  9 14:26 ..
{{< /highlight >}}
[sample-emptydir.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-emptydir.yaml)  

`hostPath`は`Node`上の領域を指定して`Pod`のコンテナにマウントする.  
`Node`に直接影響を与えるので扱いには注意が必要.  
{{< highlight bash >}}
# Node(gke-k8s-01-pool-2-641104a4-7r06)の /etc が /srv としてPodにマウントされている
$ kubectl get pod sample-hostpath -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP         NODE                              NOMINATED NODE   READINESS GATES
sample-hostpath   1/1     Running   0          3m22s   10.0.1.5   gke-k8s-01-pool-2-641104a4-7r06   <none>           <none>
$ kubectl get pod sample-hostpath -o json | jq '.spec.volumes[0]'
{
  "hostPath": {
    "path": "/etc",
    "type": "DirectoryOrCreate"
  },
  "name": "hostpath-sample"
}
$ kubectl get pod sample-hostpath -o json | jq '.spec.containers[0].volumeMounts[0]'
{
  "mountPath": "/srv",
  "name": "hostpath-sample"
}

# マウントしたNodeの領域にアクセスする
$ kubectl exec -it sample-hostpath cat /srv/os-release | grep PRETTY_NAME
PRETTY_NAME="Container-Optimized OS from Google" # NodeのOS
# コンテナ自体の領域にアクセスする
$ kubectl exec -it sample-hostpath cat /etc/os-release | grep PRETTY_NAME
PRETTY_NAME="Debian GNU/Linux 9 (stretch)" # コンテナのOS
# 実際にNodeの同じ領域のファイルを確認する
$ gcloud compute ssh gke-k8s-01-pool-2-641104a4-7r06
gke-k8s-01-pool-2-641104a4-7r06 ~ $ cat /etc/os-release | grep PRETTY_NAME
PRETTY_NAME="Container-Optimized OS from Google"
{{< /highlight >}}
[sample-hostpath.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-hostpath.yaml)  

`downwardAPI`は`Pod`の情報を`Pod`上の領域にファイルとして配置する.  
{{< highlight bash >}}
# /srv/podnameと/srv/cpu-requestにmetadata.nameとrequests.cpuの値がファイルとして配置されている
$ kubectl get pod sample-downward-api -o json | jq '.spec.volumes[0]'
{
  "downwardAPI": {
    "defaultMode": 420,
    "items": [
      {
        "fieldRef": {
          "apiVersion": "v1",
          "fieldPath": "metadata.name"
        },
        "path": "podname"
      },
      {
        "path": "cpu-request",
        "resourceFieldRef": {
          "containerName": "nginx-container",
          "divisor": "0",
          "resource": "requests.cpu"
        }
      }
    ]
  },
  "name": "downward-api-volume"
}
$ kubectl get pod sample-downward-api -o json | jq '.spec.containers[0].volumeMounts[0]'
{
  "mountPath": "/srv",
  "name": "downward-api-volume"
}

# ファイルを確認する
$ kubectl exec -it sample-downward-api cat /srv/podname
sample-downward-api
$ kubectl exec -it sample-downward-api cat /srv/cpu-request
1
{{< /highlight >}}
[sample-downward-api.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-downward-api.yaml)  

`projected`は複数の`Volume`, `Secret`, `ConfigMap`などを1つの`volumeMounts`にまとめる.  
{{< highlight bash >}}
# Secret(sample-db-auth), ConfigMap(sample-configmap), downwardAPIが /srv にまとめてマウントされている
$ kubectl get secret sample-db-auth
NAME             TYPE     DATA   AGE
sample-db-auth   Opaque   2      24h
$ kubectl get configmap sample-configmap
NAME               DATA   AGE
sample-configmap   5      12d
$ kubectl get pod sample-projected -o yaml | yq read - 'spec.volumes[0]'
name: projected-volume
projected:
  defaultMode: 420
  sources:
    - secret:
        items:
          - key: username
            path: secret/username.txt
        name: sample-db-auth
    - configMap:
        items:
          - key: nginx.conf
            path: configmap/nginx.conf
        name: sample-configmap
    - downwardAPI:
        items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
            path: podname
$ kubectl get pod sample-projected -o yaml | yq read - 'spec.containers[0].volumeMounts[0]'
mountPath: /srv
name: projected-volume

# 中身を確認
$ kubectl exec -it sample-projected ls /srv
configmap  podname  secret
$ kubectl exec -it sample-projected cat /srv/secret/username.txt
root
$ kubectl exec -it sample-projected cat /srv/configmap/nginx.conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
$ kubectl exec -it sample-projected cat /srv/podname
sample-projected
{{< /highlight >}}
[sample-projected.yaml](https://github.com/MasayaAoyama/kubernetes-perfect-guide/blob/master/samples/chapter07/sample-projected.yaml)  

### PersistentVolume
