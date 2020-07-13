---
title: "yqでKubernetesのYAMLをいじる"
date: 2020-07-06T17:58:37+09:00
draft: false
tags: ["メモ", "yq", "Kubernetes"]
---

## yqすごい
yqでKubernetesのYAMLから値を抽出したり値を書き換えたりした.  

<!--more-->
---

## まとめ
`yq`はべんり.  
{{< highlight bash >}}
# YAMLの値(key=spec.containers[0].image)を抽出
$ yq read 'pod.yaml' 'spec.containers[0].image'

# 正規表現でimageとtagを抜き出す
$ yq read 'pod.yaml' 'spec.containers[0].image' | grep -o '^[^:]\+'
$ yq read 'pod.yaml' 'spec.containers[0].image' | grep -o '[^:]\+$'

# YAMLの値(key=spec.containers[0].image)を上書き
$ yq write -i 'pod.yaml' 'spec.containers[0].image' 'nginx:1.19.0'
{{< /highlight >}}

## 環境
- macOS Mojave 10.14
- [yq](https://github.com/mikefarah/yq) version 3.3.2

## やりかた

こんな感じの`Kubernetes`のマニフェストファイル(`YAML`)を扱っていたときに,  
使用している`image`([nginx](https://hub.docker.com/_/nginx?tab=tags))の`tag`が **latest** だったのを **1.19.0** みたいな`image`の`digest`が一意に定まるようなバージョンごとの`tag`に変更したくなった.  

`pod.yaml`  
<script src="https://gist.github.com/uzimihsr/3717717a866f8373755a2384426c89c4.js"></script>

手で直接編集するのもいいんだけど,  
ゆくゆくは自動化したりしたいので`yq`を使ってスクリプトを書いてみる.  

`image-tag-fix.sh`  
<script src="https://gist.github.com/uzimihsr/d15238c659548c585731c6484a3c2aec.js"></script>

作成したスクリプトを実行する.  

{{< highlight bash >}}
$ ls
image-tag-fix.sh pod.yaml

# タグ置換スクリプト実行
$ chmod +x ./image-tag-fix.sh
$ ./image-tag-fix.sh
image : nginx
tag : latest
replaced: nginx:latest => nginx:1.19.0

# 内容確認
$ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.19.0
{{< /highlight >}}

いい感じ.  
`yq`を使って`YAML`の値を書き換えることができた.  

## おわり
`yq`は[jq](https://github.com/stedolan/jq)みたいな感じで`YAML`を扱えるのでかなり便利.  
うまく使いこなせるようになりたい.  

## おまけ
ほっぺがかわいいねこ  
![そとちゃん](/images/2020-07-06/sotochan.jpg)  

## 参考
- https://github.com/mikefarah/yq/blob/master/README.md
