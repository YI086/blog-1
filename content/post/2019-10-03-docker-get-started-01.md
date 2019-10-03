---
title: "Dockerをおさらいする 1"
date: 2019-10-03T23:54:16+09:00
draft: false
tags: ["Docker", "メモ"]
---

## Dockerおさらい
Dockerについて[Katacoda](https://www.katacoda.com/courses/docker)で一通り勉強したので, 復習も兼ねて公式ドキュメントを読んでみる.  

<!--more-->
---

## 読んだもの
[Get Started, Part 1: Orientation and setup](https://docs.docker.com/get-started/)  
この記事の内容を自分の主観バリバリで超意訳しながらまとめていく.  

## Dockerのコンセプト
- Dockerとはなんぞや?  
  - コンテナでアプリを開発, デプロイ, 実行するためのプラットフォーム.  
  - コンテナ自体は前からある技術だけど, コンテナを使ってアプリを簡単にデプロイできる点が新しい.  
- コンテナ化すると良いこと
  - どんなに複雑なアプリでもコンテナ化が可能
  - 複数のコンテナで同じカーネルを活用するため軽量
  - 臨機応変にアプリの更新が可能
  - ローカルでも, クラウド上でも, どこでも利用できる
  - コンテナの複製が簡単なのでスケーラビリティが高い
  - コンテナ同士を柔軟に組み合わせて利用できる

### イメージとコンテナ
- コンテナってどう作るの?
  - イメージを実行することで起動する
- じゃあイメージってなに?
  - アプリを動かすために必要なものが全部入った実行可能なパッケージ
  - イメージが実行状態になるとコンテナになる

### コンテナと仮想マシンのちがい
- コンテナ
  - アプリの実行に必要な環境をつくる(OSは1つ)
  - コンテナ単体で実行できる
  - 実行時に必要な分のリソース(メモリ)しか食わない
  - 一言で言えば軽いが目的の用途(アプリの実行)にしか使えない
- 仮想マシン(VM)
  - 本当のマシンとほぼ同じ環境を作る(それぞれにOSがある)
  - VMを管理するハイパーバイザを通さないとアクセスできない
  - VMを維持している間はそれらを管理するマシンのリソースを喰っている
  - 一言で言えば重いが用途が多彩

## Dockerの環境構築
自分の場合は[Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac)をインストールした.  
docker-composeとKubernetesも入っているので便利.  

### Dockerのバージョン確認
さっそくバージョン確認用のコマンドを打ってみる.  
```
# 普通のアプリと同様に--versionまたは-vオプションで確認可能
$ docker --version
Docker version 19.03.2, build 6a30dfc
```

<details><summary>さらに詳細な情報を表示する</summary><div>

```
# 詳細情報を表示
# イメージ, コンテナの数とかが表示される
$ docker info
Client:
 Debug Mode: false

Server:
 Containers: 70
  Running: 32
  Paused: 0
  Stopped: 38
 Images: 74
 Server Version: 19.03.2
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc version: 425e105d5a03fabd737a126ad93d62a9eeede87f
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.9.184-linuxkit
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 4
 Total Memory: 1.952GiB
 Name: docker-desktop
 ID: K77N:CV6M:AWUD:E3CZ:B44M:DJSX:4GLO:SNXJ:LXBZ:HUML:A46J:SYHZ
 Docker Root Dir: /var/lib/docker
 Debug Mode: true
  File Descriptors: 128
  Goroutines: 122
  System Time: 2019-10-03T14:18:37.8900535Z
  EventsListeners: 2
 HTTP Proxy: gateway.docker.internal:3128
 HTTPS Proxy: gateway.docker.internal:3129
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 Product License: Community Engine
```
</div></details>

### Dockerの動作確認
DockerでHello, World!する.  
```
# Hello, World!を表示させるイメージを実行する
# hello-worldという名前のイメージはまだローカルにないので, Docker Hubから自動で取ってくる.
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
Digest: sha256:b8ba256769a0ac28dd126d584e0a2011cd2877f3f76e093a7ae560f2a5301c00
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
`Docker Hub` : いろんなイメージが共有されてるクラウドサービス, イメージ専門のGitHubみたいなもん  

```
# 手元にあるイメージを一覧表示
$ docker image ls
REPOSITORY    TAG       IMAGE ID        CREATED         SIZE
hello-world   latest    fce289e99eb9    9 months ago    1.84kB
```
`REPOSITORY` : イメージの名前(タグがついていないもの)  
`TAG` : イメージに付くタグ, バージョンとかを表す  
`IMAGE ID` : イメージに振られるID  
`CREATED` : イメージがいつ作成されたか  
`SIZE` : イメージのサイズ  

## 感想
イメージより先にいきなりコンテナの話から始まるのでちょっととっつきづらかった.  
コンテナの特徴とかVMとの違いとか自分の中であやふやだったのでなんとなく整理できてよかった.  

次は[Get Started, Part 2: Containers](https://docs.docker.com/get-started/part2/)を読みたい.  
