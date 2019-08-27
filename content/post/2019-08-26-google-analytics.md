---
title: "GitHub Pages+HugoでつくったブログにGoogle Analyticsを埋め込む"
date: 2019-08-26T22:15:39+09:00
draft: false
tags: ["Hugo", "Google Analytics", "作業ログ"]
---

## 需要があるかはわからない
ねこ記事でアクセス数稼ぎをするのはいいが, そのアクセス数を数える方法がなかったのでやってみた.  

<!--more-->
---

## なにをしたか
Google AnalyticsをGitHub Pages上に公開している自分のブログ([Hugoで生成](https://uzimihsr.github.io/post/2019-08-07-create-blog-1/))に埋め込み, トラフィックを見られるようにした

## 必要なもの
- Googleアカウント

## どうやったか
- Google Analyticsの利用登録
- トラッキングコードをHugoに埋め込む

### Google Analyticsの利用登録

[Google Analytics](https://analytics.google.com/analytics/web/)にアクセスし, `Googleアナリティクスの利用を開始 -> 登録`と進む.  

![Google Analytics](/images/2019-08-26-google-analytics-1.png)  

`アカウントの設定`では自分のGoogleアカウント名を入力し, アカウントのデータ共有設定で必要なものにチェックをつける. スクショ失敗したので画像が無いが, 自分は全部チェックをつけた.  

![アカウントの設定](/images/2019-08-26-google-analytics-2.png)

`測定の対象を指定します。`ではウェブを指定する.  

![測定の対象を指定します。](/images/2019-08-26-google-analytics-3.png)

`プロパティの設定`でウェブサイトの名前, URLを入力する.  

![プロパティの設定](/images/2019-08-26-google-analytics-4.png)

最後に利用規約に同意したらGoogle Analyticsのダッシュボードが開く.  
ここで表示されるトラッキングコード(UA-123456789-1)を控えておく.  

![ダッシュボード](/images/2019-08-26-google-analytics-5.png)

### トラッキングコードをHugoに埋め込む
Google Analyticsの準備ができたら, `config.toml`に控えたトラッキングコードを記述する. ここに記述しておくと, Hugoで静的ページをビルドした際にトラッキング用のタグをすべてのhtmlに書き込んでくれるので便利.  

<details><summary>`config.toml`</summary><div>

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
googleAnalytics = "UA-123456789-1" # この行を追記
...(以下省略)
```
</div></details>

静的ページをビルドする. 自分の場合は[ビルド用スクリプト`deploy.sh`](https://uzimihsr.github.io/post/2019-08-07-create-blog-1/#github-pages%E3%81%A7%E3%83%9A%E3%83%BC%E3%82%B8%E3%82%92%E5%85%AC%E9%96%8B%E3%81%99%E3%82%8B)を作成しているのでこれをそのまま使う.  

```
$ ./deploy.sh "Google Analytics"
```

もう一度ダッシュボードを更新すると, 設定したブログへのトラフィックが確認できる.  

![ダッシュボード](/images/2019-08-26-google-analytics-6.png)

まだぜんぜんトラフィックがない. 悲しい...  

## おまけ
パソコンの横で眠くなるねこ  
![パソコンの横で眠くなるそとちゃん](/images/2019-08-26-sotochan-omake.jpg)
