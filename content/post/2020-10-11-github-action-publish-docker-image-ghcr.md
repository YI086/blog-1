---
title: "Github Actionsを使ってDocker ImageをGitHub Container RegistryにPushする"
date: 2020-10-11T23:01:22+09:00
draft: false
tags: ["Docker", "GitHub", "メモ"]
---

## CI/CDっぽいことがしたい
GitHub公式のCI/CDサービスGitHub Actionsを使って, リポジトリ上のDockerfileからimageをbuildしてpushする手順を試した.  

<!--more-->
---

## まとめ
[GitHub](https://github.com/)のPAT(個人アクセストークン)をリポジトリの`Secrets`に**CR_PAT**として登録した状態で以下のような`.github/workflows/docker-publish.yml`を作成すると,  
**master**ブランチへのcommitやrelease(tag)の作成時に`Docker image`を`GitHub Action`で`GitHub Container Registry`にpushすることができる.  
pushした`image` : https://github.com/users/uzimihsr/packages/container/package/echo
{{< highlight yaml >}}
# ghcr.io/<GitHubアカウント>/<image名>:<タグ>のimageをGitHub Packagesにpushするworkflow
name: Docker

on:
  # masterブランチまたはvから始まるtag(例:`v1.2.3`)のpushでjobsを実行する
  push:
    branches:
      - master
    tags:
      - v*
  # 全ブランチのPRに対してもjobsを実行
  pull_request:
env:
  # <image名>を指定
  IMAGE_NAME: echo
jobs:
  # テスト用のjob
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: |
          if [ -f docker-compose.test.yml ]; then
            docker-compose --file docker-compose.test.yml build
            docker-compose --file docker-compose.test.yml run sut
          else
            docker build . --file Dockerfile
          fi
  # imageをpushする
  push:
    # test jobが成功した場合のみトリガーされる
    needs: test
    runs-on: ubuntu-latest
    # pushイベント以外(PRなど)では実行されない
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v2
      - name: Build image
        run: docker build . --file Dockerfile --tag $IMAGE_NAME
      - name: Log into GitHub Container Registry
        # `read:packages`と`write:packages`の権限を持つPAT(個人アクセストークン)をSecretに`CR_PAT`として登録しておく
        # PATの作成手順 : https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token
        run: echo "${{ secrets.CR_PAT }}" | docker login https://ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Push image to GitHub Container Registry
        run: |
          # <GitHubアカウント>が自動で選択される
          IMAGE_ID=ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME
          IMAGE_ID=$(echo $IMAGE_ID | tr '[A-Z]' '[a-z]')
          VERSION=$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')
          # git tagの値を<タグ>として付与
          [[ "${{ github.ref }}" == "refs/tags/"* ]] && VERSION=$(echo $VERSION | sed -e 's/^v//')
          # masterブランチへのcommitでトリガーされている場合はlatestを<タグ>として使う
          [ "$VERSION" == "master" ] && VERSION=latest
          echo IMAGE_ID=$IMAGE_ID
          echo VERSION=$VERSION
          docker tag $IMAGE_NAME $IMAGE_ID:$VERSION
          docker push $IMAGE_ID:$VERSION
{{< /highlight >}}
https://github.com/uzimihsr/echo-image/blob/master/.github/workflows/docker-publish.yml  

## 環境
- macOS Mojave 10.14
- [GitHub](https://github.com/)
- [GitHub Packages](https://github.com/features/packages)
- [Git](https://git-scm.com/)
  - git version 2.20.1 (Apple Git-117)
- [Docker Desktop for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac)
  - Version 2.1.0.3
  - Docker Engine 19.03.13

## やりかた

- [PATの発行とログイン](#patの発行とログイン)
- [手動でpush](#手動でpush)
- [GitHub Actionsでpush](#github-actionsでpush)

### PATの発行とログイン
まずは公式ドキュメント[^1]を参考に, `GitHub Container Registry`の認証に必要な`PAT`(個人アクセストークン)を発行する.  

`GitHub`にログインした状態で  
https://github.com/settings/tokens  
を開き, **Generate new token**をクリック.  
![GitHub](/images/2020-10-11/sc01.png)  

`PAT`の権限設定画面では`write:packages`, `read:packages`にのみチェックを入れて**Genetate token**をクリック.  
![GitHub](/images/2020-10-11/sc02.png)  

`PAT`が発行される.  
![GitHub](/images/2020-10-11/sc03.png)  

発行された`PAT`を使って, コマンドラインから`GitHub Container Registry`にログインする.  

{{< highlight bash >}}
# GitHub Container Registryにログイン
$ echo <PAT> | docker login ghcr.io -u <GitHubアカウント> --password-stdin
Login Succeeded
{{< /highlight >}}

これでログインは成功.  

### 手動でpush
上記の手順で`GitHub Container Registry`にログインした状態で, まずはコマンドラインから`image`をpushしてみる.  

適当な`image`を作成.  

{{< highlight bash >}}
# 適当なディレクトリで適当なDockerfileを作成
$ cd /path/to/workspace
$ touch Dockerfile

# Docker imageをビルド, 動作確認
$ docker image build -t echo:latest .
$ docker container run --rm echo:latest
hello, world!
{{< /highlight >}}

`Dockerfile`
<script src="https://gist.github.com/uzimihsr/cb9e6994a2db25bc888f7aebd41b189f.js"></script>

次にこの`image`に`GitHub Container Registry`用のタグをつけ, pushする.  

{{< highlight bash >}}
# imageのタグをつけ直す
$ docker image tag echo:latest ghcr.io/uzimihsr/echo:latest

# Docker imageをpush
$ docker image push ghcr.io/uzimihsr/echo:latest
The push refers to repository [ghcr.io/uzimihsr/echo]
be8b8b42328a: Pushed
latest: digest: sha256:c7926eae1c1aef6291e5eef1673c9815b5644fa9c417bfab65a4baba50c046a2 size: 527
{{< /highlight >}}

pushした`image`の情報は`GitHub Packages`のUIから確認できる.  
https://github.com/users/uzimihsr/packages/container/package/echo  
![GitHub](/images/2020-10-11/sc04.png)  

もちろんこの`image`をpullして使うこともできる.  

{{< highlight bash >}}
# ローカルのimageを削除
$ docker image rm ghcr.io/uzimihsr/echo:latest

# imageをpull
$ docker image pull ghcr.io/uzimihsr/echo:latest
latest: Pulling from uzimihsr/echo
Digest: sha256:c7926eae1c1aef6291e5eef1673c9815b5644fa9c417bfab65a4baba50c046a2
Status: Downloaded newer image for ghcr.io/uzimihsr/echo:latest
ghcr.io/uzimihsr/echo:latest

# 動作確認
$ docker container run --rm ghcr.io/uzimihsr/echo:latest
hello, world!
{{< /highlight >}}

これで`GitHub Container Registry`を使った`image`のpush/pullができるようになった.  

### GitHub Actionsでpush
次に`GitHub Actions`を使って`image`を`GitHub Container Registry`にpushしてみる.  

新たに`GitHub`リポジトリを[作成](https://github.com/new)し, 先程作成した`Dockerfile`をgit pushする.  
作ったリポジトリ : https://github.com/uzimihsr/echo-image  

{{< highlight bash >}}
# Dockerfileが存在する状態
$ ls
Dockerfile

# git initからpushまで
$ git init
$ git remote add origin https://github.com/uzimihsr/echo-image.git
$ git add .
$ git commit -m "initial commit"
$ git push origin master
{{< /highlight >}}

リポジトリの**Actions**タブを開き, `Publish Docker Container`アクションの**Set up this workflow**をクリック.  
![GitHub](/images/2020-10-11/sc05.png)  

`workflow`の定義ファイル`docker-publish.yml`を編集する画面が開く.  
このままでも動くけど, 自分用に`env.IMAGE_NAME`だけ任意の`image`名に修正する.  

<details><summary>`docker-publish.yml`</summary><div>
{{< highlight yaml >}}
# ghcr.io/<GitHubアカウント>/<image名>:<タグ>のimageをGitHub Packagesにpushするworkflow
name: Docker

on:
  # masterブランチまたはvから始まるtag(例:`v1.2.3`)のpushでjobsを実行する
  push:
    branches:
      - master
    tags:
      - v*
  # 全ブランチのPRに対してもjobsを実行
  pull_request:
env:
  # <image名>を指定
  IMAGE_NAME: echo
jobs:
  # テスト用のjob
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: |
          if [ -f docker-compose.test.yml ]; then
            docker-compose --file docker-compose.test.yml build
            docker-compose --file docker-compose.test.yml run sut
          else
            docker build . --file Dockerfile
          fi
  # imageをpushする
  push:
    # test jobが成功した場合のみトリガーされる
    needs: test
    runs-on: ubuntu-latest
    # pushイベント以外(PRなど)では実行されない
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v2
      - name: Build image
        run: docker build . --file Dockerfile --tag $IMAGE_NAME
      - name: Log into GitHub Container Registry
        # `read:packages`と`write:packages`の権限を持つPAT(個人アクセストークン)をSecretに`CR_PAT`として登録しておく
        # PATの作成手順 : https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token
        run: echo "${{ secrets.CR_PAT }}" | docker login https://ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Push image to GitHub Container Registry
        run: |
          # <GitHubアカウント>が自動で選択される
          IMAGE_ID=ghcr.io/${{ github.repository_owner }}/$IMAGE_NAME
          IMAGE_ID=$(echo $IMAGE_ID | tr '[A-Z]' '[a-z]')
          VERSION=$(echo "${{ github.ref }}" | sed -e 's,.*/\(.*\),\1,')
          # git tagの値を<タグ>として付与
          [[ "${{ github.ref }}" == "refs/tags/"* ]] && VERSION=$(echo $VERSION | sed -e 's/^v//')
          # masterブランチへのcommitでトリガーされている場合はlatestを<タグ>として使う
          [ "$VERSION" == "master" ] && VERSION=latest
          echo IMAGE_ID=$IMAGE_ID
          echo VERSION=$VERSION
          docker tag $IMAGE_NAME $IMAGE_ID:$VERSION
          docker push $IMAGE_ID:$VERSION
{{< /highlight >}}
</div></details>
https://github.com/uzimihsr/echo-image/blob/master/.github/workflows/docker-publish.yml  

`docker-publish.yml`の編集が終わったら, **Start commit** -> **Commit new file**と進み`workflow`を作成する.  
![GitHub](/images/2020-10-11/sc07.png)  

次にこの`workflow`で`GitHub Container Registry`の認証を行うための`Secrets`の設定をする.  

リポジトリの**Settings**タブを開き, **New sectet**をクリック.  
![GitHub](/images/2020-10-11/sc08.png)  

`Secret`の作成画面では`Name`を**CR_PAT**(`docker-publish.yml`内で参照している名前)に設定し, `Value`には先程作成した`PAT`を貼り付けて**Add secret**をクリック.  
![GitHub](/images/2020-10-11/sc09.png)  

`Secret`が作成された.  
![GitHub](/images/2020-10-11/sc10.png)  

`Secret`を作成した状態でリポジトリの**Actions**タブを開くと,  
先程の`docker-publish.yml`作成時のcommitで起動した`workflow`が失敗している.  
(**CR_PAT** の`Secret`を作る前に実行されたため, 認証部分でコケている)  
**Re-run all jobs**から再度`workflow`を実行してみる.  
![GitHub](/images/2020-10-11/sc11.png)  

今度は**CR_PAT**が作成済みなので`workflow`が正常に終了し, `image`が**ghcr.io/uzimihsr/echo:latest**にpushされた.  
![GitHub](/images/2020-10-11/sc12.png)  

再度`GitHub Package`の画面  
https://github.com/users/uzimihsr/packages/container/package/echo  
を開くと, 確かに**Last published**が更新されている.  
![GitHub](/images/2020-10-11/sc13.png)  

以上で`GitHub Actions`を使って`image`を`GitHub Container Registry`にpushする設定ができたので, 試しに`image`を更新してみる.  

{{< highlight bash >}}
# masterからhogeブランチを切る
$ git checkout master
$ git pull origin master
$ git checkout -b hoge

# Dockerfileを修正
$ vim Dockerfile
$ cat Dockerfile
FROM busybox

ENTRYPOINT [ "echo" ]

CMD [ "good morning!" ] # CMDを変更した

# commitしてhogeブランチをpush
$ git add ./Dockerfile
$ git commit -m "fixed Dockerfile: CMD"
$ git push origin hoge
{{< /highlight >}}

`Dockerfile`の修正commitを積んだ**hoge**ブランチがpushされた状態でPR(**master** <- **hoge**)を作成すると, PRにトリガーされた`workflow`が実行される.  
**test**の`job`のみが実行され**push**の`job`がスキップされているが, これは今回作成した`docker-publish.yml`で`GitHub event`がpushの場合にのみ実行するよう設定しているため.  
```yaml
if: github.event_name == 'push'
```
![GitHub](/images/2020-10-11/sc14.png)  

このPRをmergeすると今度は**master**ブランチへのpushが行われるので, それにトリガーされた`workflow`で**test**と**push**の`job`が実行される.  
以降の`image`がpushされるまでの流れは先程試した流れと同じで, **latest**タグの`image`がpushされる.  

**latest**タグだけでは`image`のバージョン管理が不便なので,  
最新版の`image`にバージョンタグを付与してみる.  
やり方としては最新のcommitにgit tagをつけてpushするだけ.  

{{< highlight bash >}}
# masterブランチで作業
$ git checkout master
$ git pull origin master

# masterブランチの最新のcommitにvから始まるtagをつけてpushする
$ git tag -a v0.0.1 -m "hogehoge"
$ git push origin v0.0.1
{{< /highlight >}}

するとtagのpushにトリガーされて`workflow`が実行される.  
今度はtagで指定したvからはじまる文字列が`image`のタグとして付与されていることがわかる.  
![GitHub](/images/2020-10-11/sc15.png)  

再度`Packages`の画面を開くと確かにgit tagで指定した**0.0.1**の`image`がpushされている.  
![GitHub](/images/2020-10-11/sc16.png)  

今回はコマンドラインでtagをつけたけど, UIからreleaseを作っても同様に`image`のpushができる.  
![GitHub](/images/2020-10-11/sc17.png)  

最後に`image`の動作確認を行う.  

{{< highlight bash >}}
# 動作確認
$ docker container run --rm ghcr.io/uzimihsr/echo:0.0.1
good morning! # CMDの変更が反映されている
{{< /highlight >}}

やったぜ.  
`GitHub Actions`を使って`GitHub Container Registry`に`image`をタグ付けしてpushできるようになった.  

## おわり
`GitHub`リポジトリ上の`Dockerfile`から`image`を作れるようになった.  
`GitHub Actions`を使うのも初めてだったので, 練習にもなってよかった.  
`Docker Hub`が無料プランだといろいろ制限が厳しくなってきてるので[^2], (今のところは無料の)`GitHub Container Registry`に乗り換えていこうと思う.  

## おまけ
おもちゃで遊ぶねこ  
![そとちゃん](/images/2020-10-11/sotochan.jpg)  

## 参考

[^1]: https://docs.github.com/ja/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token  
[^2]: https://www.docker.com/pricing/resource-consumption-updates  
