---
title: "AngularアプリをnginxとGitHub Pagesにデプロイ"
date: 2020-05-04T10:43:39+09:00
draft: true
tags: ["作業ログ", "Angular", "JavaScript", "Node.js"]
---

## フロントエンド開発したい
GWだけど特に遊ぶ予定もないので普段あまりやらないフロントエンドの開発をやってみようと思った.  

<!--more-->
---

## やったことのまとめ

- `anyenv`と`nodenv`で`Node.js`の環境構築をした
- `Angular CLI`をインストールした
- エディタのセットアップをした
- `Angular`アプリを`GitHub Pages`で公開した

## つかうもの

- macOS Mojave 10.14
    - Go, Docker, Kubernetesはこちらで実行
- anyenv
    - https://github.com/anyenv/anyenv
    - anyenv 1.1.1
    - インストール済み
- nodenv
    - https://github.com/nodenv/nodenv
    - nodenv 1.3.2+2.2578d8d
    - **今回入れる**

## やったこと

- [Node.jsのインストール](#nodejsのインストール)
- [Angular CLIのインストール](#angular-cliのインストール)
- [VSCodeのインストール](#vscodeのインストール)
- [デプロイ](#デプロイ)
    - [nginxにデプロイ](#nginxにデプロイ)
    - [GitHub Pagesにデプロイ](#github-pagesにデプロイ)

### Node.jsのインストール

`nodenv`は`Node.js`のバージョン管理をやってくれるやつ.  
これがなくても困らないけど, 入れておくとバージョン更新でトラブったときとかに多分便利.  

まずは`anyenv`で`nodenv`をインストールしてみる.  
めちゃくちゃかんたん. `anyenv`神.  

```bash
# anyenvでインストールできる**envの確認
$ anyenv install -l | grep nodenv
  nodenv

# nodenvのインストール
$ anyenv install nodenv
...
Install nodenv succeeded!
Please reload your profile (exec $SHELL -l) or open a new session.
$ exec $SHELL -l

# 動作確認
$ nodenv
nodenv 1.3.2+2.2578d8d
Usage: nodenv <command> [<args>]

Some useful nodenv commands are:
   commands    List all available nodenv commands
   local       Set or show the local application-specific Node version
   global      Set or show the global Node version
   shell       Set or show the shell-specific Node version
   install     Install a Node version using node-build
   uninstall   Uninstall a specific Node version
   rehash      Rehash nodenv shims (run this after installing executables)
   version     Show the current Node version and its origin
   versions    List installed Node versions
   which       Display the full path to an executable
   whence      List all Node versions that contain the given executable

See 'nodenv help <command>' for information on a specific command.
For full documentation, see: https://github.com/nodenv/nodenv#readme
```

`nodenv`がインストールできたので, 今度はこれを使って`Node.js`をインストールする.  
`Node.js`はとんでもない数のバージョンがあるんだけど,  
初心者でよくわかんないので[公式](https://nodejs.org/ja/)で現在(2020年5月4日)推奨版とされている **12.16.3** を入れる.  

```bash
# nodenvでインストール可能できるバージョンの一覧を確認
$ nodenv install -l | grep -e "^12.*$"
12.0.0
12.x-dev
12.x-next
12.1.0
12.2.0
12.3.0
12.3.1
12.4.0
12.5.0
12.6.0
12.7.0
12.8.0
12.8.1
12.9.0
12.9.1
12.10.0
12.11.0
12.11.1
12.12.0
12.13.0
12.13.1
12.14.0
12.14.1
12.15.0
12.16.0
12.16.1
12.16.2
12.16.3

# Node.js 12.16.3をインストール
$ nodenv install 12.16.3
...
nodenv: default-packages file not found

# なんか怒られたので対処する
# nodenv installしたときに自動で入れるパッケージを指定するファイルが必要らしいので空ファイルを作る
$ touch $(nodenv root)/default-packages

# 再度インストール
$ nodenv install 12.16.3
...
Installed node-v12.16.3-darwin-x64 to /Users/uzimihsr/.anyenv/envs/nodenv/versions/12.16.3

Installed default packages for 12.16.3

# インストール済みのバージョンを確認し使用するバージョンを指定
$ nodenv versions
  12.16.3
$ nodenv global 12.16.3
$ nodenv versions
* 12.16.3 (set by /Users/uzimihsr/.anyenv/envs/nodenv/version)

# 動作確認
$ exec $SHELL -l
$ node -v
v12.16.3
$ npm -v
6.14.4
```

以上で`Node.js`のインストールは完了.  
めっちゃ簡単だった. `nodenv`も神.  

ついでに入ってる`npm`は`Node.js`のパッケージ管理ツールで,  
`python`でいう`pip`みたいなやつ.  
(そういえば最近全然`python`触ってないな...)  

### Angular CLIのインストール

続いて`Angular`の開発をするために`npm`を使ってCLIをインストールする.  
無くても開発はできるみたいだけど, 入れない理由はない.  

```bash
# Angular CLIのインストール
$ npm install -g @angular/cli
...
/Users/uzimihsr/.anyenv/envs/nodenv/versions/12.16.3/bin/ng -> /Users/uzimihsr/.anyenv/envs/nodenv/versions/12.16.3/lib/node_modules/@angular/cli/bin/ng

> @angular/cli@9.1.4 postinstall /Users/uzimihsr/.anyenv/envs/nodenv/versions/12.16.3/lib/node_modules/@angular/cli
> node ./bin/postinstall/script.js

...
+ @angular/cli@9.1.4
added 271 packages from 206 contributors in 24.329s

# 動作確認
$ exec $SHELL -l
$ ng version

...

Angular CLI: 9.1.4
Node: 12.16.3
OS: darwin x64

Angular:
...
Ivy Workspace:

Package                      Version
------------------------------------------------------
@angular-devkit/architect    0.901.4
@angular-devkit/core         9.1.4
@angular-devkit/schematics   9.1.4
@schematics/angular          9.1.4
@schematics/update           0.901.4
rxjs                         6.5.4
```

CLIがインストールできたので, 実際に`Angular`アプリを動かしてみる.  
CLIは`ng`で呼び出せる.  

```bash
# 適当なディレクトリで作業
$ cd workspace

# 新規ワークスペースとサンプルアプリを作成
# パッケージをいくつか入れるので時間がかかる
$ ng new my-app
? Would you like to add Angular routing? Yes
? Which stylesheet format would you like to use? CSS
CREATE my-app/README.md (1022 bytes)
...
✔ Packages installed successfully.
    Successfully initialized git.

# 作成されたディレクトリでアプリを起動する
$ cd my-app
$ ng serve
...
** Angular Live Development Server is listening on localhost:4200, open your browser on http://localhost:4200/ **
: Compiled successfully.
# 動作確認が終わったらCtrl+Cで終了
```

ブラウザで **http://localhost:4200/** を開く.  
サンプルアプリが起動していることが確認できる.  

![Chrome](/images/2020-05-04/sc01.png)  

これで`Angular`のアプリを動かせるようになった.  
やったぜ.  

### VSCodeのインストール

普段開発用のエディタは[Atom](https://atom.io/)を使ってるんだけど,  
`Angular`向けの良いパッケージが見つからなかったので[Visual Studio Code](https://code.visualstudio.com/)(`VSCode`)を使ってみる.  

ホントは`Atom`で頑張りたかったんだけど  
[公式のIDEリスト](https://angular.io/resources?category=development)でも推奨されてるし,  
チュートリアルとかで使ってる[StackBlitz](https://stackblitz.com/)も`VSCode`っぽいIDEなので逆らえなかった.  

普通に[公式のダウンロードページ](https://code.visualstudio.com/Download)から落としてくる.  
勝手に展開されるので, **Visual Studio Code.app** を **/Applications** に移動する.  

![Applications](/images/2020-05-04/sc02.png)  

とは言っても毎回アプリを探して起動するのは不便なので`PATH`を通してコマンドラインから起動できるようにする.  
実際に起動し, `F1`でコマンドパレットを開き`Install 'code' command in PATH`を選択する.  

![VSCode](/images/2020-05-04/sc03.png)  

画面右下に`Shell command 'code' successfully installed in PATH.`と表示されれば設定は完了.  

試しに先程作成したワークスペースをコマンドラインから開いてみる.  

```bash
$ cd my-app

# VSCodeでカレントディレクトリを開く
$ code .
```

![VSCode](/images/2020-05-04/sc04.png)  

さらに`Angular`を扱いやすくするために[公式のExtention](https://marketplace.visualstudio.com/items?itemName=Angular.ng-template)を入れておく.  
`Atom`でいうパッケージみたいな拡張機能を`VSCode`では`Extentions`と呼ぶらしい.  

![VSCode](/images/2020-05-04/sc05.png)  

これでエディタのセットアップも完了.  

### デプロイ

開発環境のセットアップができたので, 試しにアプリのデプロイをやってみる.  

`Angular`公式の[入門](https://angular.jp/start)で作ったアプリがダウンロードできたので,  
これを試しにビルド, デプロイしてみる.  

サンプルアプリを作ったときと同じように, 新たにワークスペースを作成する.  
今回はサンプルアプリは作らず空のワークスペースを作ってみる.  

```bash
# 空のワークスペースを作成
$ cd workspace
$ ng new --create-application=false angular-first-app
$ ls angular-first-app
README.md         angular.json      node_modules      package-lock.json package.json      tsconfig.json     tslint.json
```

自分の作った[StackBlitzプロジェクト](https://stackblitz.com/angular/odpeknvxnlq)から`Download Project`してきた **<プロジェクトID>.angular** をコピーして,  
実際にアプリを立ち上げてみる.  

```bash
# 入門で作ったプロジェクトをコピー
$ cd angular-first-app
$ cp -rf ~/Downloads/<プロジェクト名>.angular/* ./

# そのまま起動すると依存関係が足りなくて失敗する
$ ng serve
An unhandled exception occurred: Cannot find module '@angular-devkit/build-angular/package.json'
Require stack:
...
See "/private/var/folders/t7/qck11mhn5fj4q6r1mbdf2nxw0000gn/T/ng-D1yxxF/angular-errors.log" for further details.

# package.jsonに記述された依存パッケージをインストールしてから再度起動
$ npm install
$ ng serve
...
** Angular Live Development Server is listening on localhost:4200, open your browser on http://localhost:4200/ **
: Compiled successfully.
# 動作確認ができたらCtrl+Cで終了する
```

ブラウザで **http://localhost:4200/** を開くと[入門でつくったもの](https://odpeknvxnlq.angular.stackblitz.io)と同じアプリが起動していることが確認できる.  

![Chrome](/images/2020-05-04/sc06.png)  

#### nginxにデプロイ
これでアプリの動作確認はできたので, 実際にビルドしてみる.  
ビルドが成功すると成果物として **dist** ディレクトリが作成されていることがわかる.  

```bash
# ビルド前の状態
$ cd angular-first-app
$ ls
README.md          karma.conf.js      package-lock.json  src                tsconfig.json      tslint.json
angular.json       node_modules       package.json       tsconfig.app.json  tsconfig.spec.json

# アプリをビルド
$ ng build --prod
Generating ES5 bundles for differential loading...
ES5 bundle generation complete.

...
Date: 2020-05-04T01:48:14.298Z - Hash: 19cf3332dd4d450b70af - Time: 39294ms

# ビルド後
$ ls
README.md          dist               node_modules       package.json       tsconfig.app.json  tsconfig.spec.json
angular.json       karma.conf.js      package-lock.json  src                tsconfig.json      tslint.json
$ ls dist
3rdpartylicenses.txt                     main-es2015.6d0587fd878af4417329.js      polyfills-es5.30e587ebdc07016ad8d1.js    styles.c7ea3b8058a0e880ad91.css
assets                                   main-es5.6d0587fd878af4417329.js         runtime-es2015.1eba213af0b233498d9d.js
index.html                               polyfills-es2015.f8d7ae8b8a28c567fae7.js runtime-es5.1eba213af0b233498d9d.js
```

ビルドした成果物は`nginx`にデプロイできる.  
作成された **dist** ディレクトリをドキュメントルートに設定するだけ. かんたん.  

```bash
# 絶対パスを確認
$ cd dist
$ pwd
/path/to/angular-first-app/dist

# nginx設定ファイルを編集して起動
$ vim /usr/local/etc/nginx/nginx.conf
$ nginx

# 動作確認が終わったら止める
$ nginx -s stop
```

<details><summary>`nginx.conf`</summary><div>
```nginx
worker_processes  1;
events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;

    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  localhost;

        location / {
            # Angularアプリのdistディレクトリを指定
            root   /path/to/angular-first-app/dist;
            index  index.html index.htm;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    include servers/*;
}
```
</div></details>

`nginx`が問題なく動いたらブラウザで **http://localhost:8080/** を開く.  
`ng serve`したときと同じアプリが動いていることが確認できる.  

![Chrome](/images/2020-05-04/sc07.png)  

今回はMacの`nginx`だったので手動で止めたけど,  
本番環境で`nginx`がdaemon化されている場合も同様に`nginx.conf`をいじればアプリがデプロイできる. はず.  

#### GitHub Pagesにデプロイ
