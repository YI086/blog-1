---
title: GitHub PagesとHugoでブログをつくった①
date: 2019-08-07T23:20:05+09:00
draft: true
---

## ブログをつくった
ブログをつくった.  
特に大きな目的があるわけではないがアウトプットの練習に使ったりうちのかわいいネッコについて書いたりしたい.  

---

## GitHub PagesとHugoでブログをつくった
さっそくアウトプットの練習として今回ブログを作った手順を書いていく.  

### つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- テキストエディタ(何でも良い, 今回はVimを使った)
- 多少のgit知識


### 前提条件
- `brew`がインストールされていること
    - [homebrew](https://brew.sh/)
- GitHubのアカウントがあること
    - [GitHub](https://github.com/)

### 参考にしたもの
- [GitHub Pagesの作り方](https://pages.github.com/)
- [Hugoクイックスタート](https://gohugo.io/getting-started/quick-start/)
- [HugoでつくったページをGitHub Pagesに置く](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

### やったこと

#### GitHub repositoryを作成
https://github.com/new  
で, 新たに`blog`という名前のリポジトリを作成する.  
このrepoはHugoのプロジェクトを置くのに使う.  
[uzimihsr/blog.git](https://github.com/uzimihsr/blog.git)

同様に, `username.github.io`リポジトリを作成する.  
`username`はアカウント名で置き換える. 例 : `uzimihsr.github.io`  
ここに配置したファイルがGitHub Pagesとして公開される.  
[uzimihsr/uzimihsr.github.io](https://github.com/uzimihsr/uzimihsr.github.io.git)

#### Hugoでページを作成する
以下, ターミナルで操作
```
# 適当な作業用ディレクトリに移動(自分の場合はWorkspaceを使う)
$ cd ~/Workspace

# Hugoをインストール
$ brew install hugo

# Hugoで新規プロジェクトを生成
# Workspace/blogが生成される
$ hugo new site blog
$ cd blog

# Hugo Themeを追加
# 今回はBeautiful Hugo(https://themes.gohugo.io/beautifulhugo/)を使用
$ git init
$ git submodule add https://github.com/halogenica/beautifulhugo.git themes/beautifulhugo

# Beautiful HugoのexampleSiteを利用する
$ cp -r themes/beautifulhugo/exampleSite/* .

# ページを生成し, ローカルで立ち上げる
# http://localhost:1313/ をブラウザで開くとexampleSiteのページが確認できる.
# だいたいわかったらCtrl+Cでhugo serverを止める.
$ hugo server -D

# ビルド前にconfigを自分用に編集しておく.
# 基本はusernameとなっている部分をアカウント名に変えれば良い.
$ vim config.toml
```

`config.toml`
```
baseurl = "https://uzimihsr.github.io"
DefaultContentLanguage = "en"
#DefaultContentLanguage = "ja"
title = "meow.md"
theme = "beautifulhugo"
metaDataFormat = "yaml"
pygmentsStyle = "trac"
pygmentsUseClasses = true
pygmentsCodeFences = true
pygmentsCodefencesGuessSyntax = true
#pygmentsUseClassic = true
#pygmentOptions = "linenos=inline"
#disqusShortname = "XXX"
#googleAnalytics = "XXX"

[Params]
#  homeTitle = "Beautiful Hugo Theme" # Set a different text for the header on the home page
  subtitle = "にゃーん"
  logo = "img/avatar-icon.png" # Expecting square dimensions
  favicon = "img/favicon.ico"
  dateFormat = "January 2, 2006"
  commit = false
  #rss = true
  comments = true
  readingTime = true
  wordCount = true
  useHLJS = true
  socialShare = true
  delayDisqus = true
  showRelatedPosts = true
#  gcse = "012345678901234567890:abcdefghijk" # Get your code from google.com/cse. Make sure to go to "Look and Feel" and change Layout to "Full Width" and Theme to "Classic"

#[[Params.bigimg]]
#  src = "img/triangle.jpg"
#  desc = "Triangle"
#[[Params.bigimg]]
#  src = "img/sphere.jpg"
#  desc = "Sphere"
#  # position: see values of CSS background-position.
#  position = "center top"
#[[Params.bigimg]]
#  src = "img/hexagon.jpg"
#  desc = "Hexagon"

[Author]
  name = "uzimihsr"
  #website = "yourwebsite.com"
  email = "uzimihsr@domain.com"
  #facebook = "uzimihsr"
  #googleplus = "+uzimihsr" # or xxxxxxxxxxxxxxxxxxxxx
  github = "uzimihsr"
  #gitlab = "uzimihsr"
  #bitbucket = "uzimihsr"
  twitter = "uzimihsr"
  #reddit = "uzimihsr"
  #linkedin = "uzimihsr"
  #xing = "uzimihsr"
  #stackoverflow = "users/XXXXXXX/uzimihsr"
  #snapchat = "uzimihsr"
  #instagram = "uzimihsr"
  #youtube = "user/uzimihsr" # or channel/channelname
  #soundcloud = "uzimihsr"
  #spotify = "uzimihsr"
  #bandcamp = "uzimihsr"
  #itchio = "uzimihsr"
  #vk = "uzimihsr"
  #paypal = "uzimihsr"
  #telegram = "uzimihsr"
  #500px = "uzimihsr"

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

# [[menu.main]]
#     identifier = "samples"
#     name = "Samples"
#     weight = 4
#
# [[menu.main]]
#     parent = "samples"
#     name = "Big Image Sample"
#     url = "post/2017-03-07-bigimg-sample"
#     weight = 1
#
# [[menu.main]]
#     parent = "samples"
#     name = "Math Sample"
#     url = "post/2017-03-05-math-sample"
#     weight = 2
#
# [[menu.main]]
#     parent = "samples"
#     name = "Code Sample"
#     url = "post/2016-03-08-code-sample"
#     weight = 3

# [[menu.main]]
#     name = "Tags"
#     url = "tags"
#     weight = 3

```

#### GitHub Pagesで公開する
そのままblogディレクトリで操作.
```
# 作成したリモートリポジトリを紐付ける
$ git remote add origin https://github.com/uzimihsr/blog.git
$ git submodule add -b master https://github.com/uzimihsr/uzimihsr.github.io.git public

# 自動で更新できるようにスクリプトを生成
$ vim deploy.sh
$ chmod +x deploy.sh

# スクリプトを使ってサイトを更新
# https://uzimihsr.github.io をブラウザで開くとローカルで確認したものと同じページが確認できる.
# 更新にすこし時間がかかるかもしれない.
$ ./deploy.sh "Initial commit"
```

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
`deploy.sh`がやってることの要約
 - hugoコマンドでblogディレクトリの内容を元に静的ページをビルド
 - public内に生成されたファイルをまとめてuzimihsr.github.ioリポジトリにpush

#### 記事を追加する
```
# 新しい記事を追加する
$ hugo new post/2019-08-07-create-blog-1.md

# 記事を編集する
$ vim content/post/2019-08-07-create-blog-1.md

# ブログを更新する
$ ./deploy.sh "2019-08-07-create-blog-1.md"
```

以上, ブログを作って記事を追加するところまでやった.  
このままだとほとんどexampleSiteのままなので自分用にいろいろ調整していきたい.
