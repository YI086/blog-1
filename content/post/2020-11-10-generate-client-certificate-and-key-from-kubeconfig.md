---
title: "kubeconfigにベタ書きされたclient-certificate-dataをファイル化して使う"
date: 2020-11-10T21:43:47+09:00
draft: false
tags: ["メモ", "Kubernetes", "yq", "awk"]
---

## それはそう
冷静に考えれば大したことないんだけどちょっと詰まったので一応メモ.  

<!--more-->
---

## まとめ
`kubeconfig`にベタ書きされている**client-certificate-data**(クライアント証明書)と**client-key-data**(秘密鍵)は`base64`デコードするとファイルとして普通に使える.  
```bash
# kubeconfigにベタ書きされたクライアント証明書(client.crt)と秘密鍵(client.key)をファイルに出力する
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-certificate-data}' | base64 -D -o client.crt
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-key-data}' | base64 -D -o client.key

# kubectlじゃなくてもOK
## awkでやるパターン
$ cat ~/.kube/config | grep client-certificate-data | awk '{print $2}' | base64 -D -o client.crt
$ cat ~/.kube/config | grep client-key-data | awk '{print $2}' | base64 -D -o client.crt
## yqつかうパターン
$ cat ~/.kube/config | yq r - "users[*].user.client-certificate-data" | base64 -D -o client.crt
$ cat ~/.kube/config | yq r - "users[*].user.client-key-data" | base64 -D -o client.key

# クライアント証明書(client.crt)と秘密鍵(client.key)を使ってcurlでKubernetes APIにアクセス
$ curl --insecure --cert ./client.crt --key ./client.key https://<Kubernetes API>
```

## 環境

- macOS Mojave 10.14
- [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac)
  - Version 2.5.0.0
  - Kubernetes: v1.19.3
- [yq](https://github.com/mikefarah/yq) version 3.3.2

## kubectlからクライアント証明書と秘密鍵を生成

`Docker Desktop`で作った`Kubernetes`クラスタへのクライアント認証の情報は`~/.kube/config`(`kubeconfig`)に自動でベタ書きされている.  

```bash
# client-certificate-data : クライアント証明書
# client-key-data : クライアントの秘密鍵
$ kubectl config view --minify --raw
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0t...Cg==
    server: https://kubernetes.docker.internal:6443
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: LS0t...LS0K
    client-key-data: LS0t...Cg==
```

このクラスタのAPIへ`curl`で接続しようとすると, [クライアント認証](https://uzimihsr.github.io/post/2020-06-26-client-certification-practice/)で弾かれてしまう.  

```bash
# k8s API側がオレオレ証明書なので --insecure が必要
$ curl --insecure https://kubernetes.docker.internal:6443
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

おとなしく`kubectl`を使えばいいんだけど, 理由はさておきどうしても`curl`で接続したいことがあった.  
[curlでクライアント認証](https://uzimihsr.github.io/post/2020-06-26-client-certification-practice/)を突破するには`--cert`(クライアント証明書)と`--key`(クライアント秘密鍵)が必要なので,  
`kubeconfig`にベタ書きされている値を`base64`でデコードする.  

```bash
# クライアント証明書をbase64デコード
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-certificate-data}' | base64 -D
-----BEGIN CERTIFICATE-----
MIIDFTCCAf2gAwIBAgIICzCv4rM20vUwDQYJKoZIhvcNAQELBQAwFTETMBEGA1UE
...
QCUVil5khgn66X0Pd2GAs37k66Yyx7urGw==
-----END CERTIFICATE-----

# クライアントの秘密鍵をbase64デコード
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-key-data}' | base64 -D
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAx6UtMTtlcvZWKPiQcDSlP7Ic2b2QOpigVifG6HOU5OtBc+Fn
...
PEUMoWLF7jKfAGRtKrnY9DTE4HeDoohXlDG47KC+TLrm1bSwxMIn
-----END RSA PRIVATE KEY-----
```

よくみる`PEM`形式の証明書と秘密鍵が確認できる.  
あとはこれをファイルに出力するだけ.  

```bash
# クライアント証明書(client.crt)と秘密鍵(client.key)をファイルに出力する
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-certificate-data}' | base64 -D -o client.crt
$ kubectl config view --minify --raw -o jsonpath='{.users[].user.client-key-data}' | base64 -D -o client.key
```

作成したクライアント証明書と秘密鍵のファイルを使って再度`Kubernetes`クラスタのAPIを叩いてみる.  

```bash
# クライアント証明書(client.crt)と秘密鍵(client.key)を使って認証を突破
$ curl --insecure --cert ./client.crt --key ./client.key https://kubernetes.docker.internal:6443
{
  "paths": [
    "/api",
    "/api/v1",
    ...,
    "/version"
  ]
}
```

クライアント認証を突破できた.  
やったぜ.  

今回はクライアント証明書と秘密鍵を`kubeconfig`から取り出すのに`kubectl`を使ったけど,  
`yaml`形式のファイルから任意の値を取り出せるならもちろん他の方法でもできる.  
`kubectl`が使えなくて`kubeconfig`しかないような場合はこっちの方法を使うことが多そう.  
```bash
# grepとawkでやる例
$ cat ~/.kube/config | grep client-certificate-data | awk '{print $2}' | base64 -D
$ cat ~/.kube/config | grep client-key-data | awk '{print $2}' | base64 -D
# yqでやる例
$ cat ~/.kube/config | yq r - "users[*].user.client-certificate-data" | base64 -D
$ cat ~/.kube/config | yq r - "users[*].user.client-key-data" | base64 -D
# いずれも出力はkubectl config viewから取り出したときと同じ
```

## おわり
`kubectl`は使えないけど`kubeconfig`は存在する, みたいな環境でどうしても`Kubernetes`クラスタのAPIをたたく必要があったので一応やってみた.  
`kubectl`が使える環境だけど`curl`でのアクセスを試したい, というような場合は事前に`ServiceAccount`を作って`Bearer Token`を発行したほうがたぶんはやい[^1].  

## おまけ
食パンクッションに乗るねこ  
![そとちゃん](/images/2020-11-10/sotochan.jpg)  

## 参考

[^1]: [Kubenretes Docs](https://kubernetes.io/ja/docs/reference/access-authn-authz/authentication/#%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%82%A2%E3%82%AB%E3%82%A6%E3%83%B3%E3%83%88%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3)  
