---
title: "Docker Get Startedを読む Part3"
date: 2019-10-06T16:58:10+09:00
draft: false
tags: ["Docker", "メモ"]
---

## Dockerおさらい
[前回](https://uzimihsr.github.io/post/2019-10-05-docker-get-started-02/)のつづき  

<!--more-->
---

## 読んだもの
[Get Started, Part 3: Services](https://docs.docker.com/get-started/part3/)  
コンテナを実際に使うためのサービスに関する内容.  

## 事前準備
このパートに入る前に以下の条件をクリアすること.  

- バージョン1.13以降のDockerがインストール済みであること
- Docker Composeが入っていること
    - Docker Desktop for Macには入ってるので大丈夫
- [Part 1](https://docs.docker.com/get-started/), [Part 2](https://docs.docker.com/get-started/part2/)の内容を理解していること
- Part2で作成したイメージがDocker Hubにアップロードされていて, 正常に動くこと

## はじめに
このパートではDockerにおけるアプリ開発の3つの[段階](https://uzimihsr.github.io/post/2019-10-05-docker-get-started-02/#%E3%81%AF%E3%81%98%E3%82%81%E3%81%AB)のうち,  
アプリのスケーリングとロードバランシング(負荷分散)を行うサービスについて説明する.  

## サービスとは
- 分散アプリケーションを構成するそれぞれのアプリケーション
    - 例:動画共有サイト(分散アプリケーション)
        - DBに動画を保存するサービス(アプリ)
        - アップロードされた動画をエンコードするサービス(アプリ)
        - フロントエンド用のサービス(アプリ)
- 本番環境のコンテナ
    - あくまで1つのイメージを使用
    - イメージ(コンテナ)の実行方法を定義するもの
        - 使用するポートの指定
        - サービスの規模に合わせたコンテナのレプリカ数の指定など
    - (超意訳)イメージの使い方を細かく設定した実用性の高いコンテナのラッパー?
- `docker-compose.yml`で定義する

## はじめてのdocker-compose.yml
`docker-compose.yml`はYAML形式でコンテナの動作を定義するファイルである.  

### docker-compose.yml
実際に作ってみる.  
どのディレクトリでもいいので`docker-compose.yml`を以下の内容で作成する.  

```
$ mkdir workspace
$ cd workspace
$ vim docker-compose.yml
```

<details><summary>`docker-compose.yml`</summary><div>

```
version: "3"
services:
  # webという名前のサービスを定義
  web:
    # Part2でDocker Hubに登録したイメージを指定
    image: uzimihsr/get-started:part2
    deploy:
      # イメージインスタンスの数を5に指定
      replicas: 5
      resources:
        limits:
          # 各インスタンスのCPU性能を10%に制限
          cpus: "0.1"
          # 各インスタンスのメモリを50MBに制限
          memory: 50M
      restart_policy:
        # コンテナが停止した場合すぐに再起動するよう設定
        condition: on-failure
    ports:
      # ローカルマシンの4000番ポートをwebの80番に割り当て
      - "4000:80"
    networks:
      # webnetという名前のネットワークでポートを共有するよう設定
      - webnet
networks:
  # webnetという名前のネットワークを定義
  # 何も指定しない場合は負荷分散ネットワークになる
  webnet:
```

</div></details>

ざっくり説明するとこの`docker-compose.yml`は

- webという名前のサービスを定義
    - Docker Hubの[イメージ](https://hub.docker.com/r/uzimihsr/get-started/tags)を使用する
    - コンテナレプリカは5個稼働させる
    - 各インスタンス(レプリカ?)のCPUはシステムの10%, メモリは50MBに設定
    - コンテナが死んだら再起動させる
    - ホスト(Dockerを起動しているマシン)の4000番ポートをwebサービスの80番ポートに割り当てる
    - webnetというネットワークを使用する
- webnetという名前のネットワークを定義
    - 負荷分散ネットワーク

という設定を行っている.  

## ロードバランスしたアプリを動かす
`docker-compose.yml`の内容を実行するため, まずはswarmクラスタを初期化する.  
swarmについては[Part 4](https://docs.docker.com/get-started/part4/)で説明するので, 今はおまじないだと思って大丈夫.  
```
$ docker swarm init
Swarm initialized: current node (hw3rcr1q9vlpp3qgfw959knwb) is now a manager.
```
それでは`docker stack deploy`コマンドを使用して実際にアプリを立ち上げる.  
`-c FILE_PATH`オプションで使用する`docker-compose.yml`へのパスを指定できる.  
今回は現在のディレクトリにある`docker-compose.yml`を指定して, getstartedlabという名前のアプリ(スタック)を起動する.  
```
$ ls
docker-compose.yml
$ docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
```
実際に起動したサービスの一覧は`docker service ls`で確認できる.  
または, `docker stack services getstartedlab`でもgetstartedlabスタックに関連するサービスの一覧が表示される.  
今回はgetstartedlabスタックの中でgetstartedlab_webというサービスが作成されていることが確認できる.  
```
$ docker service ls
ID              NAME                MODE          REPLICAS     IMAGE                         PORTS
bggiqgkl98zv    getstartedlab_web   replicated    5/5          uzimihsr/get-started:part2    *:4000->80/tcp
$ docker stack services getstartedlab
ID              NAME                MODE          REPLICAS     IMAGE                         PORTS
bggiqgkl98zv    getstartedlab_web   replicated    5/5          uzimihsr/get-started:part2    *:4000->80/tcp
```
また, サービスの中で起動されているコンテナはタスクと呼ばれ, それぞれにユニークなIDが振られて管理される.  
今回は`docker service ps`コマンドでgetstartedlab_webサービスのタスクを確認する.  
または, 他のコンテナが何も起動していない場合に限り`docker container ls -q`コマンドでコンテナIDの一覧を取得することができる.  
(タスクのIDとは違うことに注意)
```
$ docker service ps getstartedlab_web
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
p8z32hd4620k        getstartedlab_web.1   uzimihsr/get-started:part2   docker-desktop      Running             Running 8 minutes ago
x6qtw0s78rl4        getstartedlab_web.2   uzimihsr/get-started:part2   docker-desktop      Running             Running 8 minutes ago
d5u6qaw9y74a        getstartedlab_web.3   uzimihsr/get-started:part2   docker-desktop      Running             Running 8 minutes ago
qmhi68ujtazx        getstartedlab_web.4   uzimihsr/get-started:part2   docker-desktop      Running             Running 8 minutes ago
j692c5bbtilh        getstartedlab_web.5   uzimihsr/get-started:part2   docker-desktop      Running             Running 8 minutes ago
$ docker container ls -q
0d3641cc5d70
3fccce9c79e8
7ec5196a0211
374fec7c58a7
152251abda79
```
ロードバランシング(負荷分散)が行われているか確認する.  
ブラウザでも良いが, 今回はcurlで何回かURLを叩いてみる.  
```
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 374fec7c58a7<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
~/Workspace/docker-tutorial
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 3fccce9c79e8<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
~/Workspace/docker-tutorial
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 0d3641cc5d70<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
~/Workspace/docker-tutorial
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 7ec5196a0211<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
~/Workspace/docker-tutorial
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> 152251abda79<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
~/Workspace/docker-tutorial
(⎈ |docker-desktop:default)❯
```
注目すべきはHostnameの部分で, アクセスする度にホストが変わっていることから負荷分散(ラウンドロビン方式)が正常に行われていることがわかる.  

getstartedlabスタック内のタスクは`docker stack ps getstartedlab`で確認できるが,  
今回はgetstartedlab_webサービスしか動いていないため,  
先程`docker service ps`で確認したのと同じタスクの一覧が表示される.  
```
$ docker stack ps getstartedlab
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
p8z32hd4620k        getstartedlab_web.1   uzimihsr/get-started:part2   docker-desktop      Running             Running 22 minutes ago
x6qtw0s78rl4        getstartedlab_web.2   uzimihsr/get-started:part2   docker-desktop      Running             Running 22 minutes ago
d5u6qaw9y74a        getstartedlab_web.3   uzimihsr/get-started:part2   docker-desktop      Running             Running 22 minutes ago
qmhi68ujtazx        getstartedlab_web.4   uzimihsr/get-started:part2   docker-desktop      Running             Running 22 minutes ago
j692c5bbtilh        getstartedlab_web.5   uzimihsr/get-started:part2   docker-desktop      Running             Running 22 minutes ago
```

## アプリをスケーリングする
`docker-compose.yml`の`services.web.deploy.replicas`の値を変更することでアプリのスケーリングができる.  
以下の手順でgetstartedlabスタックを更新してみる.  
なお, 更新前にスタックを停止したりコンテナを削除する必要はない(自動でやってくれる).  
```
$ vim docker-compose.yml
$ docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: bggiqgkl98zvy8ayiqmwxyt79)
```

<details><summary>レプリカ数を変更した`docker-compose.yml`</summary><div>

```
version: "3"
services:
  web:
    image: uzimihsr/get-started:part2
    deploy:
      # イメージインスタンスの数を3に指定
      replicas: 3
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      う設定
      - webnet
networks:
  webnet:
```

</div></details>

再びgetstartedlabスタックのタスクを確認してみると, タスクが減っている(今回は3を指定した)ことが確認できる.  
```
$ docker stack ps getstartedlab
ID                  NAME                  IMAGE                        NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
p8z32hd4620k        getstartedlab_web.1   uzimihsr/get-started:part2   docker-desktop      Running             Running 35 minutes ago
x6qtw0s78rl4        getstartedlab_web.2   uzimihsr/get-started:part2   docker-desktop      Running             Running 35 minutes ago
d5u6qaw9y74a        getstartedlab_web.3   uzimihsr/get-started:part2   docker-desktop      Running             Running 35 minutes ago
```
余談だが, `docker inspect`コマンドでコンテナ(タスク)の詳細を確認することもできる.  
<details><summary>`docker inspect`</summary><div>

```
$ docker inspect p8z32hd4620k
[
    {
        "ID": "p8z32hd4620kaipbv9uzlte2j",
        "Version": {
            "Index": 22
        },
        "CreatedAt": "2019-10-06T09:01:36.6249845Z",
        "UpdatedAt": "2019-10-06T09:01:41.9956973Z",
        "Labels": {},
        "Spec": {
            "ContainerSpec": {
                "Image": "uzimihsr/get-started:part2@sha256:ca9c71fd6d4195a2dd6e83f708383451b612910d64e7103a26f710b4428fbbc9",
                "Labels": {
                    "com.docker.stack.namespace": "getstartedlab"
                },
                "Privileges": {
                    "CredentialSpec": null,
                    "SELinuxContext": null
                },
                "Isolation": "default"
            },
            "Resources": {
                "Limits": {
                    "NanoCPUs": 100000000,
                    "MemoryBytes": 52428800
                }
            },
            "RestartPolicy": {
                "Condition": "on-failure",
                "MaxAttempts": 0
            },
            "Placement": {
                "Platforms": [
                    {
                        "Architecture": "amd64",
                        "OS": "linux"
                    }
                ]
            },
            "Networks": [
                {
                    "Target": "uvihn12ykm2owgxebox1wxrb7",
                    "Aliases": [
                        "web"
                    ]
                }
            ],
            "ForceUpdate": 0
        },
        "ServiceID": "bggiqgkl98zvy8ayiqmwxyt79",
        "Slot": 1,
        "NodeID": "hw3rcr1q9vlpp3qgfw959knwb",
        "Status": {
            "Timestamp": "2019-10-06T09:01:41.9350846Z",
            "State": "running",
            "Message": "started",
            "ContainerStatus": {
                "ContainerID": "374fec7c58a77a554ebdd5b079fcd72b36462275f99c730131f0a0b83744dfeb",
                "PID": 71366,
                "ExitCode": 0
            },
            "PortStatus": {}
        },
        "DesiredState": "running",
        "NetworksAttachments": [
            {
                "Network": {
                    "ID": "5w9oj87tachjift8phcql9ayo",
                    "Version": {
                        "Index": 6
                    },
                    "CreatedAt": "2019-10-06T08:55:54.1014898Z",
                    "UpdatedAt": "2019-10-06T08:55:54.117508Z",
                    "Spec": {
                        "Name": "ingress",
                        "Labels": {},
                        "DriverConfiguration": {},
                        "Ingress": true,
                        "IPAMOptions": {
                            "Driver": {},
                            "Configs": [
                                {
                                    "Subnet": "10.255.0.0/16",
                                    "Gateway": "10.255.0.1"
                                }
                            ]
                        },
                        "Scope": "swarm"
                    },
                    "DriverState": {
                        "Name": "overlay",
                        "Options": {
                            "com.docker.network.driver.overlay.vxlanid_list": "4096"
                        }
                    },
                    "IPAMOptions": {
                        "Driver": {
                            "Name": "default"
                        },
                        "Configs": [
                            {
                                "Subnet": "10.255.0.0/16",
                                "Gateway": "10.255.0.1"
                            }
                        ]
                    }
                },
                "Addresses": [
                    "10.255.0.4/16"
                ]
            },
            {
                "Network": {
                    "ID": "uvihn12ykm2owgxebox1wxrb7",
                    "Version": {
                        "Index": 12
                    },
                    "CreatedAt": "2019-10-06T09:01:32.1749865Z",
                    "UpdatedAt": "2019-10-06T09:01:32.1787669Z",
                    "Spec": {
                        "Name": "getstartedlab_webnet",
                        "Labels": {
                            "com.docker.stack.namespace": "getstartedlab"
                        },
                        "DriverConfiguration": {
                            "Name": "overlay"
                        },
                        "Scope": "swarm"
                    },
                    "DriverState": {
                        "Name": "overlay",
                        "Options": {
                            "com.docker.network.driver.overlay.vxlanid_list": "4097"
                        }
                    },
                    "IPAMOptions": {
                        "Driver": {
                            "Name": "default"
                        },
                        "Configs": [
                            {
                                "Subnet": "10.0.0.0/24",
                                "Gateway": "10.0.0.1"
                            }
                        ]
                    }
                },
                "Addresses": [
                    "10.0.0.3/24"
                ]
            }
        ]
    }
]
```

</div></details>

### アプリとswarmの停止
アプリ(スタック)を停止するには`docker stack rm`コマンドを使用する.  
```
$ docker stack rm getstartedlab
Removing service getstartedlab_web
Removing network getstartedlab_webnet
```
また, swarmも`docker swarm leave`コマンドで停止する.  
`--force`オプションで強制的に停止できる.  
```
$ docker swarm leave --force
Node left the swarm.
```

## 感想(まとめ)
仕方ない部分があるが, swarmとスタックに関する説明が後回しなのはちょっとわかりづらいと思った.  
あと後半の説明が力尽きてる感があった.  
要点としては,  

- コンテナを実用化するには負荷分散やスケーリングが必要になること
- それらの設定を管理するためにサービスを使うこと
- サービス(とネットワーク)は`docker-compose.yml`で定義できること
- アプリを起動する前にswarmクラスタを初期化すること
    - swarmに関してはPart4で説明
- アプリはスタックという単位で扱うこと
    - スタックに関してはPart5で説明
- レプリカ数を変化させることでアプリのスケーリングができること

がわかれば十分だと思う.  

次は[Get Started, Part 4: Swarms](https://docs.docker.com/get-started/part4/)を読む.
