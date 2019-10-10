---
title: "[Deprecated]Docker Get Startedを読む Part4"
date: 2019-10-09T22:14:07+09:00
draft: false
tags: ["Docker", "メモ"]
---

## Dockerおさらい
[前回](https://uzimihsr.github.io/post/2019-10-06-docker-get-started-03/)のつづき  

## Deprecated
公式の[Get Started](https://docs.docker.com/get-started/)が更新されてしまったので,  
この記事の内容は古くなっている. 非推奨.  

<!--more-->
---

## 読んだもの
[Get Started, Part 4: Swarms](https://docs.docker.com/get-started/part4/)  
複数のマシンでDockerを稼働させるためのswarmに関する内容.  

## 事前準備
このパートに入る前に以下の条件をクリアすること.  

- バージョン1.13以降のDockerがインストール済みであること
- [Part 3](https://docs.docker.com/get-started/part3/)で説明したDocker Composeが入っていること
- Docker Machineが入っていること
    - Docker Desktop for Macには入ってるので大丈夫
- [Part 1](https://docs.docker.com/get-started/), [Part 2](https://docs.docker.com/get-started/part2/)の内容を理解していること
- Part2で作成したイメージがDocker Hubにアップロードされていて, 正常に動くこと
- Part3で作成した`docker-compose.yml`があること

## はじめに
このパートでは複数台のマシンで構成されるswarmクラスタにアプリをデプロイする.  
swarmクラスタとは複数のマシンをDocker化し1つにしたもので, 複数クラスタ, 複数マシンで動作するアプリケーションを実現するものである.  

## swarmクラスタを理解する
swarmとは:  

- Dockerが稼働する複数台のマシンが1つのクラスタにまとまったもの
    - swarmを構成するマシンはノードと呼ばれる
    - swarmを構成するマシンは物理マシン/VMを問わない
    - swarmマネージャとワーカーで構成される
        - swarmマネージャ
            - 1つのswarmに1台だけ存在するノード
            - dockerコマンドを実行する
            - 他のマシン(ノード)をワーカーとして管理/指示を行う役割
            - コンテナを実行するときの方針を定める役割
                - 方針は`docker-compose.yml`で定義できる
        - ワーカー
            - swarmマネージャ以外のノード
            - リソースを提供する
            - 他のワーカーへ指示を出すことはできない

これまではシングルホストモードでDockerを使用してきたが, このパートではswarmモードでDockerを使用していく.  
swarmモードでは操作中のローカルマシンではなく, クラスタ単位でDockerのコマンドが実行されるようになる  

## swarmをセットアップする
上記の通りswarmは物理/VMを問わない複数のノード(マシン)で構成されており,  
1つのマシンで`docker swarm init`を実行するとdockerがswarmモードになり, そのマシンがswarmマネージャになる.  
他のマシンで`docker swarm join`を実行するとそのマシンがswarmクラスタにワーカーとして登録される.  
ここでは例として2台のVMでswarmクラスタを構成してみる.  

### クラスタを作成する
まずは手元のマシンでVMを建てる準備をする.  
自分の場合はMacなので[VirtualBox](https://www.virtualbox.org/wiki/Downloads)をインストールした.  

`docker-machine`コマンドを使ってDocker用のVMを2台作成する.  
今回はmyvm1, myvm2という名前のVMを作成する.  
```bash
$ docker-machine create --driver virtualbox myvm1
$ docker-machine create --driver virtualbox myvm2
```

作成したVM(docker-machine)のIPアドレスなどの情報は`docker-machine ls`コマンドで確認できる.  
```bash
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.9
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.9
```

1台目のVM(myvm1)はswarmマネージャとして, 2台目(myvm2)はワーカーとして使用する.  
VM(docker-machine)上でコマンドを実行するには`docker-machine ssh`コマンドを利用する.  
ここではmyvm1に`docker swarm init`コマンドを実行させてswarmマネージャにする.  
```bash
$ docker-machine ssh myvm1 "docker swarm init --advertise-addr 192.168.99.100"
Swarm initialized: current node (zfpm1vq86ladj9dn5cs5pdg7x) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1gpfpimuxhtsz3i8mywpwb3lmsfwzldgpq7h70iuapkuq83id6-27xtpkdnuf42c7uwu6xrmz9sz 192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

`docker swarm init`コマンドの出力結果を見ると, このswarmに新たにワーカーを追加するためのコマンドが表示されている.  
これに従って, myvm2をこのswarmに追加する.  
```bash
$ docker-machine ssh myvm2 "docker swarm join --token SWMTKN-1-1gpfpimuxhtsz3i8mywpwb3lmsfwzldgpq7h70iuapkuq83id6-27xtpkdnuf42c7uwu6xrmz9sz 192.168.99.100:2377"
This node joined a swarm as a worker.
```

これでswarmが構築できた.  
試しにswarmマネージャ(myvm1)から`docker node ls`コマンドを使用するとこのswarmに存在するノードの一覧が表示される.  
```bash
$ docker-machine ssh myvm1 "docker node ls"
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zfpm1vq86ladj9dn5cs5pdg7x *   myvm1               Ready               Active              Leader              18.09.9
2t7nqlaes8gl08swfe5d81lim     myvm2               Ready               Active                                  18.09.9
```

## アプリをswarmクラスタにデプロイする
難しいのはここまで.  
ここからはpart3と同様の手順を進めていく.  
ただしdockerコマンドはswarmマネージャ(myvm1)からのみ実行可能であることに注意.  

### docker-machineコマンドを実行するシェルをswarmマネージャに合わせて設定する
ここまでdockerコマンドを実行するためには手元のシェルで`docker-machine ssh`を使用してswarmマネージャ(myvm1)にコマンドを送信していた.  
このままでも問題ないが, 手元の`docker-compose.yml`を使いたい場合に少し面倒になる.  
そのため, 手元のシェルをmyvm1のDockerデーモンと疎通させる設定を行う.  
手元のシェルで`docker-machine env myvm1`コマンドを実行するとmyvm1と疎通するためのコマンドが出力されるので, これを貼り付けるだけで設定が完了する.  
```bash
$ docker-machine env myvm1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/ryota/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
$ eval $(docker-machine env myvm1)
```

この状態でswarmのノード確認コマンド`docker node ls`を実行すると, myvm1にssh経由で実行した際と同じ出力が得られる.  
```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zfpm1vq86ladj9dn5cs5pdg7x *   myvm1               Ready               Active              Leader              18.09.9
2t7nqlaes8gl08swfe5d81lim     myvm2               Ready               Active                                  18.09.9
```

ちなみに別のシェルを開いて疎通の設定をせずに同じコマンドを打つと当然ながらswarmマネージャとして振る舞えないのでエラーになる.  
```bash
$ docker node ls
Error response from daemon: This node is not a swarm manager. Use "docker swarm init" or "docker swarm join" to connect this node to swarm and try again.
```

### swarmマネージャからアプリをデプロイする
シェルの設定が完了し, 直接swarmマネージャ(myvm1)でコマンドが実行できるようになったのでいよいよアプリをデプロイする.  
Part3と同じ`docker-compose.yml`が手元にあることを確認し, `docker stack deploy`コマンドを実行する.
```bash
$ ls
docker-compose.yml
$ docker stack deploy -c ./docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
```

これでアプリがデプロイされた. すごい簡単.  
Part3と同様にタスクの確認を行う.  
```bash
$ docker stack ps getstartedlab
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
ikvg0oxrqbc7        getstartedlab_web.1   uzimihsr/get-started:part2   myvm1               Running             Running 3 minutes ago
uy3juv1txuon        getstartedlab_web.2   uzimihsr/get-started:part2   myvm2               Running             Running 3 minutes ago
j6545cpwqmvl        getstartedlab_web.3   uzimihsr/get-started:part2   myvm2               Running             Running 3 minutes ago
```

Part3で確認したものと同じく3つのサービスが表示される(元々は5個だがPart3の後半で3個にスケーリングしている).  
また, 注目すべきは`NODE`の列で, サービスがそれぞれmyvm1とmyvm2に分かれて稼働していることがわかる.  

### クラスタにアクセスする
デプロイされたアプリにはmyvm1, myvm2両方のIPアドレスでアクセスできる.  
```bash
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.9
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.9
```

curlで何度か叩いてみる.  
今回使用するアプリはdockerが稼働するマシンの4000番ポートをコンテナの80番ポートに割り当てていることに注意.  
```bash
$ curl http://192.168.99.100:4000
<h3>Hello World!</h3><b>Hostname:</b> bf3c2f00e239<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
$ curl http://192.168.99.100:4000
<h3>Hello World!</h3><b>Hostname:</b> df6c4458369c<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
$ curl http://192.168.99.100:4000
<h3>Hello World!</h3><b>Hostname:</b> 0541e646a6ce<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
$ curl http://192.168.99.101:4000
<h3>Hello World!</h3><b>Hostname:</b> df6c4458369c<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
$ curl http://192.168.99.101:4000
<h3>Hello World!</h3><b>Hostname:</b> bf3c2f00e239<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
$ curl http://192.168.99.101:4000
<h3>Hello World!</h3><b>Hostname:</b> 0541e646a6ce<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
```

負荷分散で3種類のホストがランダムに呼び出されているのが確認できる.  

myvm1, myvm2どちらのIPアドレスでも同じように動作するのは, swarmのノードがingressルーティングメッシュに属しているためである.  
(超意訳)今回のように`docker-compose.yml`でマシンの4000番ポートをコンテナの80番ポートに割り当てる設定をした場合,  各ノード自身が4000番ポートをswarmでデプロイされたサービスのために確保する. また, 各ノードの4000番ポートにはロードバランサーが割り当てられ, このロードバランサーはswarm内でノードに関係なく全てのコンテナに対して負荷分散を行う.  

## アプリの繰り返しとスケーリング
Part3と同様に, `docker-compose.yml`を編集することでアプリのスケーリングを行うことができる.  
また, この状態からswarmにノードを増やすこともできる.  
今回はmyvm3を作成し, myvm2と同様にワーカーとしてswarmに追加する.  
さらにコンテナ数を現在の3から8に増やしてみる.  
```bash
$ docker-machine create --driver virtualbox myvm3
$ docker-machine ssh myvm3 "docker swarm join --token SWMTKN-1-1gpfpimuxhtsz3i8mywpwb3lmsfwzldgpq7h70iuapkuq83id6-27xtpkdnuf42c7uwu6xrmz9sz 192.168.99.100:2377"
This node joined a swarm as a worker.
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zfpm1vq86ladj9dn5cs5pdg7x *   myvm1               Ready               Active              Leader              18.09.9
2t7nqlaes8gl08swfe5d81lim     myvm2               Ready               Active                                  18.09.9
p4n9y3kez6u35fkduojjloa17     myvm3               Ready               Active                                  19.03.3
$ vim docker-compose.yml
$ docker stack deploy -c ./docker-compose.yml getstartedlab
$ docker stack ps getstartedlab
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
ikvg0oxrqbc7        getstartedlab_web.1   uzimihsr/get-started:part2   myvm1               Running             Running 23 hours ago
uy3juv1txuon        getstartedlab_web.2   uzimihsr/get-started:part2   myvm2               Running             Running 23 hours ago
j6545cpwqmvl        getstartedlab_web.3   uzimihsr/get-started:part2   myvm2               Running             Running 23 hours ago
hax39564wx4d        getstartedlab_web.4   uzimihsr/get-started:part2   myvm3               Running             Running 23 seconds ago
t9qx3iyptfi7        getstartedlab_web.5   uzimihsr/get-started:part2   myvm1               Running             Running 33 seconds ago
4e84bwdkq8hj        getstartedlab_web.6   uzimihsr/get-started:part2   myvm1               Running             Running 33 seconds ago
vpf7nwgvonlk        getstartedlab_web.7   uzimihsr/get-started:part2   myvm2               Running             Running 34 seconds ago
aca3i7e0f615        getstartedlab_web.8   uzimihsr/get-started:part2   myvm3               Running             Running 23 seconds ago
```

<details><summary>レプリカ数を変更した`docker-compose.yml`</summary><div>

```yaml
version: "3"
services:
  web:
    image: uzimihsr/get-started:part2
    deploy:
      # イメージインスタンスの数を8に指定
      replicas: 8
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```

</div></details>

コンテナ数が増え, 新たに追加したワーカー(myvm3)にもサービスがデプロイされている.  
以上, 簡単にノードの追加, アプリのスケーリングができることがわかった.  

## クリーンアップと再起動
アプリの削除と再起動も簡単に行える.  

### スタックとswarm
スタックは`docker stack rm`コマンドで削除できる.  
```bash
$ docker stack rm getstartedlab
Removing service getstartedlab_web
Removing network getstartedlab_webnet
```

各ノードで`docker swarm leave`コマンドを使用するとそのノードを現在のswarmから開放することができる. が, Part5で使用するので現時点でswarmの解体は行わない.  

### docker-machineシェルの変数を解除
swarmマネージャ(myvm1)と疎通してコマンドを実行するために設定したシェルは次のように設定を解除することができる.  
```bash
$ eval $(docker-machine env -u)
$ docker node ls
Error response from daemon: This node is not a swarm manager. Use "docker swarm init" or "docker swarm join" to connect this node to swarm and try again.
```

### docker-machineを再起動する
VM(docker-machine)はローカルマシンを停止するか, `docker-machine stop`コマンドで停止することができる.  
今回は全部のVMを停止してみる.  
```bash
$ docker-machine stop $(docker-machine ls -q)
Stopping "myvm3"...
Stopping "myvm1"...
Stopping "myvm2"...
Machine "myvm1" was stopped.
Machine "myvm3" was stopped.
Machine "myvm2" was stopped.
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
myvm1   -        virtualbox   Stopped                 Unknown
myvm2   -        virtualbox   Stopped                 Unknown
myvm3   -        virtualbox   Stopped                 Unknown
```

また, 停止したVMは`docker-machine start`コマンドで再起動できる.  
```bash
$ docker-machine start myvm1
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.9
myvm2   -        virtualbox   Stopped                                       Unknown
myvm3   -        virtualbox   Stopped                                       Unknown
$ docker-machine start $(docker-machine ls -q)
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER     ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.09.9
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.09.9
myvm3   -        virtualbox   Running   tcp://192.168.99.102:2376           v19.03.3
```

## 感想(まとめ)
swarmに関する内容がもりもりだった.  
個人的にはdocker-machineを利用するとdockerをインストールしたVMが簡単に立てられて便利だと思った. VirtualBoxすごい.  
要点としては,

- swarmを利用すると複数台のマシンのリソースを利用してDockerが動かせること
- swarmはマネージャとワーカーの2種類のノードで構成されること
    - ワーカーは計算リソースを提供するのみでコンテナやノードの操作は行わない
    - マネージャはワーカーの管理を行い, これまでローカルマシンで実行していたdockerコマンドを使用してswarmクラスタにアプリをデプロイ/スケーリングする
- docker-machineを利用してローカルマシン1台の中に複数のノード用VMを簡単に作成できること
- 複数のノードで構成される1つのswarmはまるで1つのマシンのように扱うことができ, 実際にどのノードでコンテナが動くかに関係なくswarm内でロードバランシングが行えること

がわかれば十分.  

次は[Get Started, Part 5: Stacks](https://docs.docker.com/get-started/part5/)を読む.  
そろそろ疲れてきた.  
