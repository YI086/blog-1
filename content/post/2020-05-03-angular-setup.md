---
title: "MacでAngularの開発環境構築"
date: 2020-05-03T18:43:39+09:00
draft: false
tags: ["作業ログ", "Angular", "JavaScript", "Node.js", "VSCode"]
---

## フロントエンド開発したい
GWだけど特に遊ぶ予定もないので普段あまりやらないフロントエンドの開発をやってみようと思った.  
まずは環境構築からやってみた.  

<!--more-->
---

## やったことのまとめ

- `anyenv`と`nodenv`で`Node.js`の環境構築をした
- `Angular CLI`をインストールした
- `VSCode`をインストールして`Extention`を入れた

## つかうもの

- macOS Mojave 10.14
- anyenv
    - https://github.com/anyenv/anyenv
    - anyenv 1.1.1
    - インストール済み
- nodenv
    - https://github.com/nodenv/nodenv
    - nodenv 1.3.2+2.2578d8d
    - **今回入れる**
- Angular CLI
    - https://cli.angular.io/
    - バージョン: 9.1.4
    - **今回入れる**
- Visual Studio Code
    - https://code.visualstudio.com/
    - Version: 1.44.2
    - **今回入れる**

## やったこと

- [Node.jsのインストール](#nodejsのインストール)
- [Angular CLIのインストール](#angular-cliのインストール)
- [VSCodeのインストール](#vscodeのインストール)

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

![Chrome](/images/2020-05-03/sc01.png)  

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

![Applications](/images/2020-05-03/sc02.png)  

とは言っても毎回アプリを探して起動するのは不便なので`PATH`を通してコマンドラインから起動できるようにする.  
実際に起動し, `F1`でコマンドパレットを開き`Install 'code' command in PATH`を選択する.  

![VSCode](/images/2020-05-03/sc03.png)  

画面右下に`Shell command 'code' successfully installed in PATH.`と表示されれば設定は完了.  

試しに先程作成したワークスペースをコマンドラインから開いてみる.  

```bash
$ cd my-app

# VSCodeでカレントディレクトリを開く
$ code .
```

![VSCode](/images/2020-05-03/sc04.png)  

さらに`Angular`を扱いやすくするために[公式のExtention](https://marketplace.visualstudio.com/items?itemName=Angular.ng-template)を入れておく.  
`Atom`でいうパッケージみたいな拡張機能を`VSCode`では`Extentions`と呼ぶらしい.  
メニューから検索すればすぐ見つかるので`Install`を押せば入る.  

![VSCode](/images/2020-05-03/sc05.png)  

これでエディタのセットアップも完了.  
やったぜ.  

## おわり
以上の手順でMacに`Angular`アプリの開発環境をつくることができた.  
フロントエンド初心者だけどちょっとずつ頑張っていきたい.  
実際の成果物をデプロイする手順は長くなりそうなので[別の記事](https://uzimihsr.github.io/post/2020-05-04-angular-deploy-fix/)に書く.  

## おまけ
紙袋だいすきねこ  
![そとちゃん](/images/2020-05-03/sotochan.jpg)  

## 参考

- Node.jsのインストール
    - https://github.com/nodenv/nodenv/blob/master/README.md#installing-node-versions
- Angular CLIのインストール
    - https://angular.jp/cli
    - https://angular.jp/start/start-deployment
    - https://angular.jp/guide/setup-local
- VSCodeのインストール
    - https://code.visualstudio.com/docs/setup/mac#_launching-from-the-command-line
