---
title: "Kubernetes完全に理解したい 7章"
date: 2020-05-14T21:57:31+09:00
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

### ConfigMap

`Pod`から利用できる情報をKey-Value形式で保持するリソース.  
