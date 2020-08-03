---
title: "YAMLを使わずにkubectl run/createでサクッとリソースを作る"
date: 2020-07-29T20:43:33+09:00
draft: false
tags: ["メモ", "Kubernetes"]
---

## YAMLめんどくさい
[CKAD](https://www.cncf.io/certification/ckad/)対策で[CKAD-exercises](https://github.com/dgkanatsios/CKAD-exercises)の問題をやってみたんだけどいちいちYAMLを書くのがめんどくさすぎたのでメモ.  

<!--more-->
---

## まとめ
`kubectl run`と`kubectl create`でだいたいのリソースは作れそう.  

{{< highlight bash >}}
# Pod
$ kubectl run nginx --image=nginx --restart=Never

# Deployment
$ kubectl create deployment nginx --image=nginx

# Job
$ kubectl create job busybox --image=busybox -- /bin/sh -c 'date; echo Hello'

# CronJob
$ kubectl create cronjob busybox --image=busybox --schedule="*/10 * * * *"  -- /bin/sh -c 'date; echo Hello'

# ConfigMap
$ kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2

# Secret
$ kubectl create secret generic my-secret --from-literal=key1=supersecret --from-literal=key2=topsecret
{{< /highlight >}}

## 環境
- macOS Mojave 10.14
- [kubectl](https://github.com/kubernetes/kubectl) v1.14.6
- [Kubernetes](https://github.com/kubernetes/kubernetes) v1.14.10-gke.36

## やりかた

### Pod
`kubectl run`のオプションで`--restart=Never`をつけてあげると`Pod`が作成できる.  

{{< highlight bash >}}
# nginxのPodを作成
$ kubectl run nginx --image=nginx --restart=Never
pod/nginx created
{{< /highlight >}}

`Pod`と同時に`Service`(`ClusterIP`)を作成することもできる.  
また, 使い捨ての`Pod`を作成することもできる.  
他の`Pod`の疎通確認するときなんかに便利.  

{{< highlight bash >}}
# PodとServiceを同時に作成
$ kubectl run nginx --image=nginx --restart=Never --port=80 --expose
service/nginx created
pod/nginx created
$ kubectl get service nginx
NAME    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.4.0.85    <none>        80/TCP    14m

# 使い捨てのbusyboxのPodでServiceの疎通確認をする
$ kubectl run busybox --image=busybox --restart=Never --rm -it -- wget -O- 10.4.0.85:80
Connecting to 10.4.0.85:80 (10.4.0.85:80)
writing to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
written to stdout
pod "busybox" deleted

# 使い捨てのbusyboxのPodを作成してshに入る
$ kubectl run busybox --image=busybox --restart=Never --rm -it -- /bin/sh
/ $ hostname
busybox
/ $ # Ctrl+Dで抜ける
pod "busybox" deleted
{{< /highlight >}}

実際の`Pod`はつくらずにマニフェストのYAMLだけ生成することもできる.  
とりあえず下書きだけでも用意してさらに詳細な設定を書きたいときとかに便利.  

{{< highlight bash >}}
# Podは作成せずYAMLだけを表示させる
$ kubectl run nginx --image=nginx --restart=Never --port=80 --env=key1=value1 --limits='cpu=200m,memory=512Mi' --labels=app=v1 -o yaml --dry-run
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    app: v1
  name: nginx
spec:
  containers:
  - env:
    - name: key1
      value: value1
    image: nginx
    name: nginx
    ports:
    - containerPort: 80
    resources:
      limits:
        cpu: 200m
        memory: 512Mi
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
{{< /highlight >}}

### Deployment
`Pod`以外のリソースは`kubectl create`で作れる.  
`kubectl run`でも作れるんだけど, Deprecatedなのであんまり使わないほうがよさそう.  
`kubectl create`だと今の所レプリカ数を指定するオプションがないっぽいのが残念.  

{{< highlight bash >}}
# kubectl runでのDeployment作成はDeprecatedらしい
$ kubectl run nginx --image=nginx
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created

# Deploymentを作成
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
{{< /highlight >}}

`kubectl run`と同様に実際のリソース作成はせずYAMLだけ生成することもできる.  
こっちのほうが出番が多そう.  

{{< highlight bash >}}
# Deploymentを作成せずYAMLを表示させる
$ kubectl create deployment nginx --image=nginx -o yaml --dry-run
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
{{< /highlight >}}

### CronJob, Job

`CronJob`と`Job`も`Deployment`と同様に作成できる.  

{{< highlight bash >}}
# Jobの作成
$ kubectl create job busybox --image=busybox -- /bin/sh -c 'date; echo Hello'
$ kubectl get pods -l job-name=busybox
NAME            READY   STATUS      RESTARTS   AGE
busybox-rvsr9   0/1     Completed   0          34s
$ kubectl logs busybox-rvsr9
Tue Jul 28 14:48:24 UTC 2020
Hello

# CronJobの作成
$ kubectl create cronjob busybox --image=busybox --schedule="*/10 * * * *"  -- /bin/sh -c 'date; echo Hello'
{{< /highlight >}}

また, すでに存在する`CronJob`から`Job`部分だけを抜き出して作成することもできる.  

{{< highlight bash >}}
# CronJobからJobを作成
$ kubectl create job busybox --from=cronjob/busybox -o yaml --dry-run
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    cronjob.kubernetes.io/instantiate: manual
  creationTimestamp: null
  name: busybox
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containers:
      - command:
        - /bin/sh
        - -c
        - date; echo Hello
        image: busybox
        imagePullPolicy: Always
        name: busybox
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: OnFailure
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status: {}
{{< /highlight >}}

### ConfigMap, Secret
`ConfigMap`や`Secret`も`kubectl create`のオプションにKey-Valueの情報を渡して作ることができる.  

{{< highlight bash >}}
# ConfigMapの作成
$ kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2
$ kubectl get configmap my-config -o jsonpath={.data.key1}
config1

# Secretの作成
$ kubectl create secret generic my-secret --from-literal=key1=supersecret --from-literal=key2=topsecret
$ kubectl get secret my-secret -o jsonpath={.data.key1} | base64 -D
supersecret
{{< /highlight >}}

## おわり
`kubectl run/create`でかんたんなリソースが作れることを確認できた. いちいちYAMLを書く必要がないので便利.  
さらに`--dry-run`と`-o yaml`を使うとYAMLの雛形を生成できるので, 細かい設定を書きたい場合も便利に使えそう.  

## おまけ
あくびするねこ  
![そとちゃん](/images/2020-07-29/sotochan.jpg)  

## 参考
- https://github.com/dgkanatsios/CKAD-exercises
