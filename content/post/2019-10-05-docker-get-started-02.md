---
title: "Docker Get Startedを読む Part2"
date: 2019-10-05T12:47:56+09:00
draft: false
tags: ["Docker", "メモ"]
---

## Dockerおさらい
[前回](https://uzimihsr.github.io/post/2019-10-03-docker-get-started-01/)のつづき  

<!--more-->
---

## 読んだもの
[Get Started, Part 2: Containers](https://docs.docker.com/get-started/part2/)  
主にコンテナをつくるためのDockerfileの書き方とか, イメージの取り扱いに関する内容.  

## 事前準備
このパートに入る前に以下の条件をクリアすること.  

- バージョン1.13以降のDockerがインストール済みであること
- [Part 1](https://docs.docker.com/get-started/part2/)の内容を理解していること
- `$ docker run hello-world`が正常に動くこと

## はじめに
Dockerによるアプリ開発には大きく分けて3つの段階がある.  

- スタック
    - 一番上
    - 全てのサービスの挙動を定義する
    - Part 5で説明
- サービス
    - スタックとコンテナの中間
    - 実際のコンテナの挙動を定義する
    - Part 3で説明
- コンテナ
    - 一番下
    - **このパートで説明**

## 新しい開発環境
Pythonでのアプリ開発を例に通常のマシンとDockerでの開発を比較する.  

- 通常のマシンの場合:
    - Pythonランタイムのインストールが必要
    - アプリの動作/開発環境のセットアップが必要
        - (超意訳)アプリが動作するための依存関係とかを全部クリアする必要があるので, マシンが他の用途に使いづらくなる
- Dockerの場合:
    - Pythonランタイムが動くイメージがすでにあるのでインストールが不要
    - コードや依存するパッケージなどを全て1つのイメージにまとめることができる
    - そのイメージは`Dockerfile`を使って定義できる

## Dockerfileによるコンテナの定義
`Dockerfile`はコンテナの中身を定義する.  
コンテナ内のネットワークインターフェースとかディスクドライブは,  
コンテナを動作させるシステムの環境からは切り離されて仮想化されている.  
したがって, 開発者はコンテナのポートを外部に開放する設定と,  
アプリに必要なファイルのコンテナ内へのコピーに気を配るだけで良い.  
たったそれだけで, どの環境でもDockerさえあれば同じように動作するコンテナを作ることができる.  

### Dockerfile
実際にDockerfileを書いてみる.  
適当なディレクトリで, `Dockerfile`という名前のファイルを以下の内容で作成する.  
```
$ mkdir workspace
$ cd workspace
$ vim Dockerfile
```

<details><summary>`Dockerfile`</summary><div>

```
# FROM : 親イメージの指定を行う.
# pythonの公式イメージをこれからつくるイメージの親イメージとして使用する
FROM python:2.7-slim

# WORKDIR : イメージ内での作業ディレクトリを指定する.
# このイメージ内での作業ディレクトリを/appにする
# /appがない場合は自動で生成される
WORKDIR /app

# COPY : ローカルの環境からイメージにファイルをコピーする.
# このDockerfileを開いている現在のディレクトリ(workspace)の内容をイメージ内の/appにコピーする
COPY . /app

# RUN : イメージの中でコマンドを実行する.
# pip(pythonのパッケージマネージャ)を使用して, requirements.txt(コピーしてきたもの)に記述されている必要なパッケージを全てこのイメージにインストールする
# このDockerfileを編集している今のシステムにはインストールされない
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# EXPOSE : コンテナのポートを外部に開放する.
# このコンテナの80番ポートを外部からアクセス可能にする
EXPOSE 80

# ENV : コンテナ内での環境変数を定義する
# 環境変数NAMEにworldを定義する
ENV NAME World

# CMD : コンテナが起動した際に実行するコマンドを定義する.
# コンテナが起動した際に`$ python app.py`を実行する
CMD ["python", "app.py"]
```

</div></details>

ざっくり説明するとこの`Dockerfile`は

- pythonが動かせるイメージを持ってくる
- 必要なファイルをローカルからコピーする
- アプリに必要なパッケージとかの設定をする
- コンテナのポートを開放する
- アプリの起動コマンドを設定する

という一連の操作を定義している.  
`requirements.txt`と`app.py`はこのあとイメージをビルドする前に作成する.  

## 今回作るアプリ
`Dockerfile`内で使用する`requirements.txt`と`app.py`を作成する.  
```
$ pwd
/path/to/workspace
$ vim requirements.txt
$ vim app.py
```

<details><summary>`requirements.txt`</summary><div>

```
Flask
Redis
```
`Flask` : pythonでwebアプリを作るためのフレームワーク  
`Redis` : データベース(Redis)を扱うためのパッケージ  

</div></details>

<details><summary>`app.py`</summary><div>

```
from flask import Flask
from redis import Redis, RedisError
import os
import socket

# Connect to Redis
redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3>" \
           "<b>Hostname:</b> {hostname}<br/>" \
           "<b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

</div></details>

`requirements.txt`は今回作成するFlaskアプリに必要なパッケージを定義し,  
`app.py`はHTTPアクセスに対して変数`html`で定義された内容(環境変数`NAME`, 接続してきたホスト名, DBで保存しているカウント数)を表示する.  
これで必要なファイルの準備は完了.  
しかし`Redis`が実際にイメージの中でまだ動いていないため,  
このまま起動してもエラーメッセージが表示されることに注意.   
(`resuirements.txt`で入れたのはあくまでpythonからRedisを使うためのパッケージ)  

以上でアプリを動かす準備ができた.  
本来このFlaskアプリを動作させるためにはPythonやパッケージ(`Flask`, `Redis`)をシステムにインストールする必要があるが,  
今回はそれらがすべてイメージ内で行われるためその必要がない.   
また, Dockerで使用するイメージはそれ単体が存在するだけで使用できるので,  
イメージをシステムにインストールする必要もない. 超便利!  
(実際にDockerを使わずに何もない環境からこのアプリを作るのはちょっと面倒)

## アプリのビルド
アプリに必要なものは全て準備できたので, さっそくビルドする.  
まずは現在のディレクトリに`Dockerfile`, `app.py`, `requirements.txt`があることを確認する.  
```
$ pwd
/path/to/workspace
$ ls
Dockerfile       app.py           requirements.txt
```
ビルド用のコマンド`docker build`を使用してイメージのビルドを行う.  
`-t repository:tag`のオプションでリポジトリ名:タグ(≒イメージ名)の指定ができる.  
(`tag`を指定しない場合は自動的に`latest`が付与される. 今はそんなに重要じゃない.)  
最後の`.`は現在のディレクトリにある`Dockerfile`を使用することを示す.  
今回はfriendlyhelloというリポジトリ名でイメージをビルドする.  
```
$ docker build -t friendlyhello .
```

余談だが, ビルド時のログを見ると`Dockerfile`の内容を1行ずつ実行していることがわかる.  

<details><summary>ビルド時のログ</summary><div>

```
$ docker build -t friendlyhello .
Sending build context to Docker daemon   5.12kB
Step 1/7 : FROM python:2.7-slim
2.7-slim: Pulling from library/python
b8f262c62ec6: Pull complete
8cbb51e0b077: Pull complete
82627a456962: Pull complete
33f3f5c560fe: Pull complete
Digest: sha256:68bb099b780cf7aa60df3af68d573dc420907acfa54cbb2a53ade8886d965272
Status: Downloaded newer image for python:2.7-slim
 ---> f462855313cd
Step 2/7 : WORKDIR /app
 ---> Running in 4d73545dac95
Removing intermediate container 4d73545dac95
 ---> 9cd55a4d5845
Step 3/7 : COPY . /app
 ---> 689b85f40a7f
Step 4/7 : RUN pip install --trusted-host pypi.python.org -r requirements.txt
 ---> Running in e8a28b64a049
DEPRECATION: Python 2.7 will reach the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 won't be maintained after that date. A future version of pip will drop support for Python 2.7. More details about Python 2 support in pip, can be found at https://pip.pypa.io/en/latest/development/release-process/#python-2-support
Collecting Flask (from -r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/9b/93/628509b8d5dc749656a9641f4caf13540e2cdec85276964ff8f43bbb1d3b/Flask-1.1.1-py2.py3-none-any.whl (94kB)
Collecting Redis (from -r requirements.txt (line 2))
  Downloading https://files.pythonhosted.org/packages/bd/64/b1e90af9bf0c7f6ef55e46b81ab527b33b785824d65300bb65636534b530/redis-3.3.8-py2.py3-none-any.whl (66kB)
Collecting click>=5.1 (from Flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/fa/37/45185cb5abbc30d7257104c434fe0b07e5a195a6847506c074527aa599ec/Click-7.0-py2.py3-none-any.whl (81kB)
Collecting Werkzeug>=0.15 (from Flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/ce/42/3aeda98f96e85fd26180534d36570e4d18108d62ae36f87694b476b83d6f/Werkzeug-0.16.0-py2.py3-none-any.whl (327kB)
Collecting itsdangerous>=0.24 (from Flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/76/ae/44b03b253d6fade317f32c24d100b3b35c2239807046a4c953c7b89fa49e/itsdangerous-1.1.0-py2.py3-none-any.whl
Collecting Jinja2>=2.10.1 (from Flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/65/e0/eb35e762802015cab1ccee04e8a277b03f1d8e53da3ec3106882ec42558b/Jinja2-2.10.3-py2.py3-none-any.whl (125kB)
Collecting MarkupSafe>=0.23 (from Jinja2>=2.10.1->Flask->-r requirements.txt (line 1))
  Downloading https://files.pythonhosted.org/packages/fb/40/f3adb7cf24a8012813c5edb20329eb22d5d8e2a0ecf73d21d6b85865da11/MarkupSafe-1.1.1-cp27-cp27mu-manylinux1_x86_64.whl
Installing collected packages: click, Werkzeug, itsdangerous, MarkupSafe, Jinja2, Flask, Redis
Successfully installed Flask-1.1.1 Jinja2-2.10.3 MarkupSafe-1.1.1 Redis-3.3.8 Werkzeug-0.16.0 click-7.0 itsdangerous-1.1.0
Removing intermediate container e8a28b64a049
 ---> 5aeaca8be74e
Step 5/7 : EXPOSE 80
 ---> Running in 7ee28830810e
Removing intermediate container 7ee28830810e
 ---> a8c8153b4bd3
Step 6/7 : ENV NAME World
 ---> Running in 98d95719d709
Removing intermediate container 98d95719d709
 ---> cfd1c0282f2e
Step 7/7 : CMD ["python", "app.py"]
 ---> Running in 5e9ef2b09c2a
Removing intermediate container 5e9ef2b09c2a
 ---> c807461f0dca
Successfully built c807461f0dca
Successfully tagged friendlyhello:latest
```

</div></details>

ビルドしたイメージを確認する.  
`docker image ls`コマンドでローカルマシンにあるイメージの一覧が取得できる.  
```
$ docker image ls
REPOSITORY      TAG       IMAGE ID        CREATED         SIZE
friendlyhello   latest    c807461f0dca    9 minutes ago   148MB
```

## アプリの実行
いよいよ作成したイメージからコンテナを起動し, アプリを実行する.  
`docker run`コマンドで使用するイメージを指定するとコンテナが立ち上がる.  
`-p localport:containerport`オプションでコンテナを起動するローカルマシンのポート(localport)をコンテナのポート(containerport)に割り当てることができる.  
今回はローカルマシンの4000番ポートを起動するコンテナの80番ポートに割り当てる.  
```
$ docker run -p 4000:80 friendlyhello
* Serving Flask app "app" (lazy loading)
* Environment: production
  WARNING: This is a development server. Do not use it in a production deployment.
  Use a production WSGI server instead.
* Debug mode: off
* Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
Flaskのログが表示され, `http://0.0.0.0:80/`にアクセスするよう促されるが,  
これはあくまでコンテナ内でのメッセージなので,  
コンテナの80番ポートに割り当てられているローカルマシンの4000番ポート(http://localhost:4000) をブラウザで開く.  
![アプリを開いた画面](/images/2019-10-05-screenshot.png)  

アプリが動作して, Hello, World!とホスト名が表示される.  
前述の通り, イメージ内にRedisがないのでその旨を示すエラーメッセージも表示されている.  

一旦`Ctrl+C`でコンテナを停止し, 今度は`-d`オプションをつけてバックグラウンドでコンテナを起動する.  
この起動方法をデタッチドモードと言う.  
```
$ docker run -d -p 4000:80 friendlyhello
```
先程はブラウザで開いたので, 今度は`curl`コマンドで動作確認する.  
```
$ curl http://localhost:4000
<h3>Hello World!</h3><b>Hostname:</b> xxxxxxxxxxxx<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>
```
ブラウザのときと同じHTTPレスポンスが返ってくることがわかる.  

また, 起動中のコンテナ一覧を`docker container ls`コマンドで確認できる.  
```
$ docker container ls
CONTAINER ID    IMAGE           COMMAND           CREATED           STATUS          PORTS                   NAMES
cc47aa120f15    friendlyhello   "python app.py"   58 seconds ago    Up 57 seconds   0.0.0.0:4000->80/tcp    zen_liskov
```
今度は起動中のコンテナを停止してみる.  
コンテナの停止には`docker container stop`コマンドを使用する.  
停止するコンテナは`CONTAINER ID`で指定する.  
```
$ docker container stop cc47aa120f15
```

## イメージをシェアする
ビルドしたイメージを別の環境でも動かすためにはレジストリにアップロードする必要がある.  
レジストリとはリポジトリが集まる場所で, リポジトリとはイメージの集まりのことである.  
レジストリの1つのアカウントは複数のリポジトリを作ることができるので,  
感覚的にはレジストリがGitHub, リポジトリがGitHubリポジトリに似ている.  
デフォルトでは[Docker Hub](https://hub.docker.com/)がリポジトリとして使用される.  

### Docker IDでログインする
事前に[hub.docker.com](https://hub.docker.com/)でアカウントを作成しておく.  
ローカルマシンからDocker Hubにログインするには`docker login`コマンドを使用する.  
```
$ docker login
```

### イメージにタグをつける
レジストリ上でイメージは`username/repository:tag`の形式で識別される(usernameはレジストリID).  
Dockerイメージにはタグで意味のあるバージョン名または番号を付与する必要があるので,  
イメージをアップロードする前にはタグをつけ直す必要がある.  
タグの付与には`docker tag`コマンドを使用する.  
今回は作成したfriendlyhelloイメージに`uzimihsr/get-started:part2`という名前をつける.  
```
$ docker tag friendlyhello uzimihsr/get-started:part2
```
イメージ一覧を表示すると, 先程まで使用していたfriendlyhelloと同じイメージIDのuzimihsr/get-startedが作成されている.  
```
$ docker image ls
REPOSITORY              TAG       IMAGE ID        CREATED             SIZE
friendlyhello           latest    c807461f0dca    About an hour ago   148MB
uzimihsr/get-started    part2     c807461f0dca    About an hour ago   148MB
```

### イメージを公開する
タグ付けしたイメージは`docker push`コマンドでアップロードできる.  
先程タグを付け直したuzimihsr/get-startedをアップロードする.  
```
$ docker push uzimihsr/get-started:part2
```

実際にアップロードしたイメージはここ.  
https://hub.docker.com/r/uzimihsr/get-started/tags  
![Docker Hub](/images/2019-10-05-docker-hub.png)  

### リモートリポジトリから入手したイメージを実行する
Docker Hubにイメージの公開ができたので, 試しに公開したイメージを使ってコンテナを起動してみる.  
確実にリモートリポジトリから取得したイメージを使用するため, 現在手元にあるイメージとコンテナを削除する.  
コンテナの削除には`docker container rm`コマンドを使用する.  
削除するコンテナのIDを指定するか, `$(docker container ls -a -q)`を指定すると全てのコンテナを削除してくれる.  
同様にイメージの削除には`docker image rm`コマンドを使用する.  
今回はコンテナを全削除し, 作成したイメージを削除する.  
```
$ docker container rm $(docker container ls -a -q)
$ docker container ls
CONTAINER ID    IMAGE   COMMAND   CREATED   STATUS    PORTS   NAMES
$ docker image rm -f c807461f0dca
$ docker image ls
REPOSITORY    TAG   IMAGE ID    CREATED   SIZE
```
Docker Hubにあるイメージを指定してコンテナを起動させる.  
[Part1](https://uzimihsr.github.io/post/2019-10-03-docker-get-started-01/)で[hello-world](https://hub.docker.com/_/hello-world/)を起動したときと同様に, イメージが手元に無いので自動でダウンロードしてくる.  
```
$ docker run -p 4000:80 uzimihsr/get-started:part2
Unable to find image 'uzimihsr/get-started:part2' locally
part2: Pulling from uzimihsr/get-started
b8f262c62ec6: Already exists
8cbb51e0b077: Already exists
82627a456962: Already exists
33f3f5c560fe: Already exists
c94901432fd6: Pull complete
15e44dce546a: Pull complete
7f08569cb4d3: Pull complete
Digest: sha256:ca9c71fd6d4195a2dd6e83f708383451b612910d64e7103a26f710b4428fbbc9
Status: Downloaded newer image for uzimihsr/get-started:part2
 * Serving Flask app "app" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```
再度 http://localhost:4000/ にアクセスすると今までと同様にアプリが起動していることが確認できる.  

## 感想
Dockerfileのあたりはイメージの根幹を成す部分なのでかなり内容がもりだくさんだった.  
要点としては,  

- `Dockerfile`を使ってイメージ(コンテナ)を定義できること  
- 親となるイメージにいろいろ操作を加えて自分のアプリ用の新しいイメージが作れること  
- イメージ内の環境は隔離されていて, ローカルマシンのシステムには影響がないこと
- イメージはタグで管理されること
- ビルドしたイメージはDocker Hubを使って別環境からも参照できること

がわかればこのパートは十分だと思う.  

次は[Get Started, Part 3: Services](https://docs.docker.com/get-started/part3/)を読みたい.
