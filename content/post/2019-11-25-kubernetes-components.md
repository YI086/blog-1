---
title: "Kubernetesの主要なコンポーネントについてまとめる"
date: 2019-11-25T21:20:53+09:00
draft: false
tags: ["Kubernetes", "メモ"]
---

## もうちょっとくわしくなりたい
これまでKubernetesクラスタについてちょっとずつ勉強してMasterガーnodeガーって自分なりに解釈してきたけど,  
そのMasterとNodeがどんなコンポーネントで構成されているか気になったのでまとめた.  

<!--more-->
---

## 読んだもの

- [Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055) 19章 Kubernetesのアーキテクチャを知る
- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

## コンポーネントの構成
Kubernetesクラスタを構成するコンポーネントの構成はこんな感じ.  
ほぼすべてのコンポーネントは`Master`の`kube-apiserver`とつながっている.  

![構成図](/images/2019-11-25-architecture.png)
<div>Icons made by <a href="https://www.flaticon.com/authors/smashicons" title="Smashicons">Smashicons</a> from <a href="https://www.flaticon.com/" title="Flaticon">www.flaticon.com</a></div>
<br>


- Masterコンポーネント
    - etcd
    - kube-apiserver
    - kube-scheduler
    - kube-controller-manager
    - cloud-controller-manager
- Nodeコンポーネント
    - kubelet
    - kube-proxy
    - コンテナランタイム(Docker)
- アドオン
    - DNS
    - Web UI(ダッシュボード)
    - コンテナリソース監視
    - クラスターレベルログ

## Masterコンポーネント
- クラスタの制御(スケジューリングなど)を行う
- クラスタイベントの検出/応答を行う  
- クラスタ内のどのマシンでも実行可能だが, 基本的に同じマシンで実行する
    - シンプルさを保つため全てを1つにまとめる
    - このマシンではユーザーのコンテナは実行しない

### etcd
- 分散型KVS(Key-Value Store)
    - 冗長化のためにetcdクラスタを組んで運用する
    - 分散合意アルゴリズム(Raft)を用いるため奇数台で運用する
- クラスタの全ての情報を保存する場所

### kube-apiserver
- Kubernetes APIを提供するコンポーネント
    - kubectlではこれにリクエストを送ることでリソースの操作を行う
    - kube-scheduler, kube-controller-manager, kubelet等はこれにリクエストを送ることで処理をしている
    - 受け取ったリクエストの情報はetcdに保存される
- ロードバランサの下に複数台並べて運用する

### kube-scheduler
- etcdに新たに登録されたリソース(Pod)をNodeに割り当てるコンポーネント
    - kube-apiserverにリクエストを送ってNodeが未定のPod情報を取得
    - 各Nodeのステータス等を考慮して割り当てをおこなう
    - 割り当てが決定したらkube-apiserverにリクエストを送ってPodをスケジューリングする
- 冗長化のため, 複数台で運用する

### kube-controller-manager
- 各種Controllerを実行するコンポーネント
    - 各種リソースの状態を監視して必要な操作を決定
    - 操作が決定したらkube-apiserverにリクエストを送る
- 冗長化のため, 複数台で運用する

### cloud-controller-manager
- クラウドサービス固有の制御を担当するコンポーネント
    - クラウドプロバイダーと対話を行う
    - クラウド固有の必要な操作を行うためのControllerを扱う
        - kube-controller-manegerとControllerがかぶらないようにする
- クラウドサービスを利用しない場合は特に気にする必要なし

## Nodeコンポーネント
- Nodeで稼働中のPodの管理を行う
- Kubernetesの実行環境を提供する

### kubelet
- 実際にコンテナの管理を行うコンポーネント
    - Podの動作を保証する
    - Kubernetesが作成したコンテナのみを管理する
    - kube-apiserverにリクエストを送って自身のNodeにスケジュールされたPodを確認, 起動する
    - コンテナランタイムと連携する
- 各Node上で動作する

### kube-proxy
- 外部からのトラフィックを正常に転送するためのコンポーネント
    - Service宛のトラフィックをPodに転送する
    - 各Nodeで動作

### コンテナランタイム(Docker)
- コンテナの実行を担当するソフトウェア
    - ユーザーのコンテナはここで実行
- Docker以外のランタイムも利用可能
    - containerd, cri-o, rktletなど

## アドオン
- クラスタの機能を実装する
    - Deployment, DaemonSetなどのリソースを使用
    - kube-systemというnamespaceに置かれている
- 以下は代表的なアドオン

### DNS(CoreDNS)
- ほぼ必須のアドオン
- クラスタ内のDNSを管理
    - PodとServiceの名前解決を行う

### Web UI(ダッシュボード)
- グラフィカルな操作が可能なUI
    - リソース情報やログの閲覧が可能

### コンテナリソース監視
- コンテナのメトリクスをDBに保存
- メトリクスを可視化するためのUIを提供
- Prometheusなどが利用可能

### クラスターレベルログ
- コンテナのログを様々な方法で取得する
    - 各Nodeでログを取得してバックエンドに送るためのロギングエージェントPodを実行する
    - アプリコンテナのログを取得して標準出力するSidecarコンテナを各Podに追加する
    - 各コンテナのログを直接バックエンドに出力する

## おわり
各種コンポーネントの役割についてなんとなくわかった気になった.  
Masterではkube-apiserverとetcdが, Nodeではkubeletが一番重要なんだと思う.  
アドオンに関してはDNSが重要なのはわかったけど他はあんまりわかんなかったので必要に応じて勉強したい.  
次は[Kubernetes完全ガイド](https://book.impress.co.jp/books/1118101055)の内容とかを少しずつまとめていきたい.  
