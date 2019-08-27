---
title: GitHub PagesとHugoでブログをつくった
date: 2019-08-07T23:20:05+09:00
draft: false
tags: ["作業ログ", "Hugo"]
---

# ブログをつくった
ブログをつくった.  
特に大きな目的があるわけではないが学んだことのアウトプットに使ったりうちのかわいいネッコについて書いたり趣味について書いたりしたい.  
つまりなんでも書き残したい.  

<!--more-->
---

# GitHub PagesとHugoでブログをつくった
さっそくアウトプットの練習として今回ブログを作った手順を書いていく.  
突然記憶喪失になったときのためになるべくわかりやすく書きたい.  

## なんでGitHub Pages?
無料だし, エンジニアっぽくてかっこいいから(偏見).  

## なんでHugo?
参考になる記事が多めだったから.  
あと人気っぽかったから.  

## つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- brew(Homebrew 2.1.9)
    - [homebrew](https://brew.sh/)
- GitHubのアカウント(uzimihsr)
    - [GitHub](https://github.com/)
- Hugo v0.56.3 (後でインストールする)
    - [Hugo](https://gohugo.io/)
- git version 2.17.2
- テキストエディタ(何でも良い, 今回はVimを使った)
- ちょっとだけGitの操作
- そこそこターミナルの操作

## 参考にしたもの
- [GitHub Pages](https://pages.github.com/)
- [Hugo quick start](https://gohugo.io/getting-started/quick-start/)
- [Hugo hosting on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

## ブログを作るまでの手順
- GitHub repositoryを作成
- Hugoでページを作成する
- GitHub Pagesでページを公開する
- 記事を追加する
- サンプルページから脱却する

### GitHub repositoryを作成
[GitHub](https://github.com/new)で新たに`blog`という名前のリポジトリを作成する.  
このrepoはHugoのプロジェクトを置くのに使う.  
今回作った例 : [uzimihsr/blog.git](https://github.com/uzimihsr/blog.git)

同様に, `username.github.io`リポジトリを作成する.  
`username`はアカウント名で置き換える.  
ここに配置したファイルがGitHub Pagesとして公開される.  
例 : [uzimihsr/uzimihsr.github.io](https://github.com/uzimihsr/uzimihsr.github.io.git)  

以上2つのリポジトリを使ってブログページを作っていく.  

### Hugoでページを作成する
Hugoをインストールして, サンプルページを作ってみる.  
以下, ターミナルで操作する.  
<br>
まずはHugoをインストール.
```
$ brew install hugo
```
<br>
適当な作業用ディレクトリに移動(自分の場合は~/Workspaceを使う),  
hugoで新規プロジェクトを作成する.  
`Workspace/blog`ディレクトリが作成される.
```
$ cd ~/Workspace
$ brew install hugo
$ hugo new site blog
$ cd blog
```
<br>
Theme(いい感じのテンプレートみたいなもん)をインストールする.  
Themeの一覧は[ここ](https://themes.gohugo.io/)で見られるが, [Beautiful Hugo](https://themes.gohugo.io/beautifulhugo/)が良さそうなのでこれを使う.  
`git submodule add`で`themes/beautifulhugo`にBeautiful Hugoのリポジトリを追加する.  
他のThemeでもたぶん同じような操作でいけるはず.  
```
$ git init
$ git submodule add https://github.com/halogenica/beautifulhugo.git themes/beautifulhugo
```
<br>
Beautiful Hugoは親切なのでサンプルページ(exampleSite)を用意してくれている.  
今回はそのまま使うので全部`blog`直下にコピーする.  
コピーできたら, 早速ローカルで確認するためにhugo serverを立ち上げる.  
http://localhost:1313/ をブラウザで開くとexampleSiteのページが確認できる. 便利.  
だいたいわかったらCtrl+Cでhugo serverを止める.
```
$ cp -r themes/beautifulhugo/exampleSite/* .
$ hugo server -D
```
<br>
次はいよいよ実際にページをビルドしてGitHub Pagesに公開する.  
が, その前に設定ファイルをいじっておく.  
`config.toml`に指定した情報をThemeが読み込んでいい感じのページを生成してくれているらしいので, 自分用にいろいろ変更する.  
```
$ vim config.toml
```
<br>
`config.toml`  
```
baseurl = "https://uzimihsr.github.io"
DefaultContentLanguage = "en"
title = "meow.md"
theme = "beautifulhugo"
metaDataFormat = "yaml"
pygmentsStyle = "trac"
pygmentsUseClasses = true
pygmentsCodeFences = true
pygmentsCodefencesGuessSyntax = true
author = false

[Params]
  subtitle = "にゃーん"
  logo = "img/avatar-icon.png" # Expecting square dimensions
  favicon = "img/favicon.ico"
  dateFormat = "January 2, 2006"
  commit = false
  rss = false
  comments = true
  readingTime = false
  wordCount = false
  useHLJS = true
  socialShare = true
  delayDisqus = true
  showRelatedPosts = true

[Author]
  name = "uzimihsr"
  github = "uzimihsr"
  twitter = "uzimihsr"

[[menu.main]]
  name = "Blog"
  url = ""
  weight = 1

[[menu.main]]
  name = "Tags"
  url = "tags"
  weight = 2

[[menu.main]]
  name = "About"
  url = "page/about/"
  weight = 3
```
<br>
### GitHub Pagesでページを公開する
ローカルの`blog`ディレクトリにリモートの`blog`リポジトリを紐付け,  
さらに`blog/public`ディレクトリに`uzimihsr.github.io`を紐付ける.  
Hugoでページをビルドすると`blog/public`ディレクトリに必要なファイルが吐き出されるので,  
これを`uzimihsr.github.io`にpushすることでGitHub Pagesが更新されていく.  
```
$ git remote add origin https://github.com/uzimihsr/blog.git
$ git submodule add -b master https://github.com/uzimihsr/uzimihsr.github.io.git public
```
<br>
いろいろごちゃごちゃしてきたので, かんたんに記事を更新できるようにスクリプト`deploy.sh`を書く.  
このスクリプトはhugoコマンドでblogディレクトリの内容を元に静的ページをビルドし,  
`blog/public`内に生成されたファイルをまとめて`uzimihsr.github.io`リポジトリにpushしてくれる.  
```
$ vim deploy.sh
$ chmod +x deploy.sh
```
<br>
`deploy.sh`
```
#!/bin/sh

# If a command fails then the deploy stops
set -e

printf "\033[0;32mDeploying updates to GitHub...\033[0m\n"

# Build the project.
hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

# Go To Public folder
cd public

# Add changes to git.
git add .

# Commit changes.
msg="rebuilding site $(date)"
if [ -n "$*" ]; then
	msg="$*"
fi
git commit -m "$msg"

# Push source and build repos.
git push origin master
```
<br>
作成したスクリプトを使ってページをビルドし, GitHub Pagesにpushする.  
すなわち, ブログを公開する. 更新されるのに少し時間がかかるが,  
https://uzimihsr.github.io をブラウザで開くとローカルで確認したものと同じページが確認できる.
```
$ ./deploy.sh "Initial commit"
```
<br>
### 記事を追加する
せっかくなので新しく記事を追加する.  
`blog/content/post`にMarkdownが追加されるので, これをいい感じに編集する.  
一番上のテーブルにある`draft: false`(下書き設定)を`draft: true`に変えると記事が公開される.  
記事を更新したら, スクリプトを使って更新した情報をGitHub Pagesに反映させる.  
```
$ hugo new post/2019-08-07-create-blog-1.md
$ vim content/post/2019-08-07-create-blog-1.md
$ ./deploy.sh "2019-08-07-create-blog-1.md"
```
<br>

### サンプルページから脱却する
サンプルページのままだといらない記事があるので, 下書き設定(`draft: false`)に変更する.
```
$ vim content/post/2015-01-04-first-post.md # 他の記事に対しても同様
$ ./deploy.sh "サンプル記事を非公開にした"
```
<br>

また, 日本語のフォントが気に食わないので`blog/themes/beautifulhugo/static/css/main.css`をいじる.  
`font-family: 'Lora', 'Times New Roman', serif;`となっている行を  
`font-family: 'arial', sans-serif;`に変更する.
```
$ vim themes/beautifulhugo/static/css/main.css
$ ./deploy.sh "フォントを変更した"
```
注意すべきなのは, submoduleを編集しているためにこの変更をgitで管理できないこと.  
少し気持ち悪いので本当はBeautiful HugoをForkしてくるべき. あとで気が向いたらやる.
<br>

アイコンも自分用に変更する.  
`blog/static/images/`に`sotochan.jpg`を配置して,  
`config.toml`の`logo = "img/avatar-icon.png"`となっている部分を  
`logo = "/images/sotochan.jpg"`に書き換える.  
```
$ mkdir static/images
$ cp ~/Desktop/sotochan.jpg static/images
$ vim config.toml
$ ./deploy.sh "アイコンを変更した"
```
<br>

トップページとaboutページを編集する.  
特にトップページに表示したいものも無いので`content/_index.md`を削除する.  
また, `content/page/about.md`を好きに編集する.  
```
$ rm content/_index.md
$ vim content/page/about.md
```
<br>

以上でだいたいブログのセットアップは完了.  
あとは3日坊主にならないよう頻繁に書いていきたい.
