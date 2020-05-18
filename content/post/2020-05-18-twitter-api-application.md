---
title: "Twitter APIを使ってコマンドラインからツイートする"
date: 2020-05-18T19:56:42+09:00
draft: false
tags: ["作業ログ", "Twitter", "Ruby"]
---

## API使ってみたい
TwitterのAPIをつかってちょっと遊んでみたくなったのでやってみた.  

<!--more-->
---

## やったことのまとめ

- `Twitter`の開発者アカウントの利用申請をした
- APIを利用するための`Twitter app`を作成した
- 公式のコマンドラインツール`twurl`を使ってAPI経由でツイートした
    - ついでに`anyenv`で`Ruby`のセットアップをした

## つかうもの

- macOS Mojave 10.14
- twurl
    - https://github.com/twitter/twurl
    - version 0.9.5
    - **今回入れる**
- jq
    - https://github.com/stedolan/jq
    - jq-1.6
    - インストール済み
    - **必須ではないがAPIレスポンスの整形に使用**
- anyenv
    - https://github.com/anyenv/anyenv
    - anyenv 1.1.1
    - インストール済み
- rbenv
    - https://github.com/rbenv/rbenv
    - rbenv 1.1.2-30-gc879cb0
    - **twurlのために今回入れる**
- Twitterのアカウント

## やったこと

- [開発者アカウントの利用申請](#開発者アカウントの利用申請)
- [appの作成](#appの作成)
- [twurlを使ってツイート](#twurlを使ってツイート)

### 開発者アカウントの利用申請

以前遊んだ[WikipediaのAPI](https://uzimihsr.github.io/post/2020-03-09-golang-api-client/)とかは誰でも自由に利用できるけど,  
[TwitterのAPI](https://developer.twitter.com/en/docs/api-reference-index)を使うには事前に開発者アカウントの利用申請が必要になるらしいので実際にやってみる.  

ブラウザで[Twitterにログイン](https://twitter.com/login?lang=ja)した状態で  
https://developer.twitter.com/ja/apply-for-access を開く.  
**開発者アカウントに申し込む** に進む.  

![Twitter](/images/2020-05-18/sc01.png)  

利用目的を聞かれるので, 適当なものを選択する.  
今回は **Exploring the API** にチェックをつけて進む.  

![Twitter](/images/2020-05-18/sc02.png)  

開発者権限を申請するアカウント(**@uzimihsr**)の情報を確認する.  

![Twitter](/images/2020-05-18/sc03.png)  

住んでる国と`Twitter`からのメールで呼ばれたい名前?を設定して次に進む.    

![Twitter](/images/2020-05-18/sc04.png)  

APIの使いみちを聞かれるので, **200文字以上の英語** で記入する.  
自分の場合は  
*"趣味でのアプリに使用します. Linux上のコマンドラインツールからツイートの投稿, タイムラインの取得, ツイートの削除などを行う予定です. 今の所はTwitterのデータを分析して何かする予定はありません."*  
的な文章を埋めた.  

![Twitter](/images/2020-05-18/sc05.png)  

用途が以下のどれかにあたる場合はさらに詳細な情報を追加しなきゃいけないらしい.  

1. `Twitter`のデータ解析を行う場合
1. ツイート, リツイート, いいね, フォロー, DMの機能を使用する場合
1. ツイートや`Twitter`コンテンツの集計情報を`Twitter`以外の場所で公開する場合
1. 用途が政府機関に関係する場合

自分は2のケースに該当するので,  
*"curlやjqなどのLinux上のコマンドラインツールを使ってツイートの投稿, タイムラインの取得, ツイートの検索, いいねなどを試そうと思います."*  
的な文章を埋めた.  
用途について記入したら次に進む.  

![Twitter](/images/2020-05-18/sc06.png)  

これまでの入力内容を確認して次に進む.  

![Twitter](/images/2020-05-18/sc07.png)  

最後に利用規約を読んでチェックボックスにチェックを入れて, **Submit Application** を押す.  

![Twitter](/images/2020-05-18/sc08.png)  

確認用メールを送った旨が表示される.  

![Twitter](/images/2020-05-18/sc09.png)  

申請したアカウントに紐付いたメールアドレスにこんな感じのメールが来るので, **Confirm your email** を押す.  

![Mail](/images/2020-05-18/sc10.png)  

利用申請が完了したことを示す画面が開く.  
完了までに時間がかかることもあるみたいだけど, 自分の場合は一瞬だった.  
用途とかで書いた英文を[Grammarly](https://www.grammarly.com/)で文法チェックしたおかげかも?  

![Twitter](/images/2020-05-18/sc11.png)  

とりあえず, これで`Twitter`の開発者アカウントを使えるようになった.  

### appの作成
`Twitter API`の機能を使うには, `app`を作成して認証用の`consumer key`と`consumer secret`を作成する必要があるらしいのでやってみる.  

先程の利用完了画面か,  
https://developer.twitter.com/en/account/get-started  
から **Create an app** に進む.  

![Twitter](/images/2020-05-18/sc12.png)  

以下の必須項目を埋めて **Create** を押す.  

|必須項目|詳細|
|---|---|
|**App name**|`app`の名前(適当につける)|
|**Application description**|`app`の説明|
|**Website URL**|`app`に関連するウェブサイトのURL<br>(自分が管理しているものが望ましい)|
|**Tell us how this app will be used**|`app`の用途<br>(`Twitter`チームがチェックするのでちゃんと書く)|

![Twitter](/images/2020-05-18/sc13.png)  

利用規約的なやつの確認モーダルが表示されるのでちゃんと確認して **Create** する.  

![Twitter](/images/2020-05-18/sc14.png)  

`app`(**uzimihsr-twurl**)が作成されて詳細画面が開くので, **Keys and tokens** に進む.  

![Twitter](/images/2020-05-18/sc15.png)  

表示された **API key**(`consumer key`), **API secret key**(`consumer secret`)は  
後ほど使用するのでメモしておく.  

![Twitter](/images/2020-05-18/sc16.png)  

これでAPIを使うための`app`の準備は完了.  

### twurlを使ってツイート

今回はcurlっぽく`Twitter API`を叩くための公式のコマンドラインツール[twurl](https://github.com/twitter/twurl)を使って実際にツイートしてみる.  

まずは`twurl`を`gem`でインストール...  

しようとしたら権限エラーでコケた.  
Macに最初から入ってるシステムの`Ruby`だとうまく行かないみたい.  

```bash
# twurlのインストール...に失敗
$ gem install twurl
Fetching: oauth-0.5.4.gem (100%)
ERROR:  While executing gem ... (Gem::FilePermissionError)
...
```

以下の手順で`Ruby`のセットアップを行う.  
<details><summary>**anyenvを使ったRubyのセットアップ**</summary><div>
`Ruby`全然使ったことないけど`anyenv`は神なので簡単にセットアップできる.  
[公式](https://www.ruby-lang.org/ja/downloads/)によると現在の安定版は **2.7.1** なのでこれを入れる.  

```bash
# rbenvをインストール
$ anyenv install -l | grep rbenv
  rbenv
$ anyenv install rbenv
...
Install rbenv succeeded!
Please reload your profile (exec $SHELL -l) or open a new session.
$ exec $SHELL -l

# 動作確認
$ rbenv --version
rbenv 1.1.2-30-gc879cb0

# Ruby(2.7.1)をインストール
$ rbenv install -l | grep 2.7.1
2.7.1
$ rbenv install 2.7.1
$ rbenv versions
* system
  2.7.1

# Rubyのバージョンを選択
$ rbenv global 2.7.1
$ exec $SHELL -l
$ rbenv versions
  system
* 2.7.1 (set by /Users/uzimihsr/.anyenv/envs/rbenv/version)

# 動作確認
$ ruby -v
ruby 2.7.1p83 (2020-03-31 revision a0c7c23c9c) [x86_64-darwin18]
$ gem -v
3.1.2
```
</div></details>

再度`twurl`をインストール.  
今度は普通にできた.  

```bash
# twurlのインストール
$ gem install twurl
...
Done installing documentation for oauth, twurl after 1 seconds
2 gems installed

# 動作確認
$ twurl -v
0.9.5
```

インストールができたのでさっそく使ってみる.  

まずは手元の`twurl`と先ほど作成した`app`, そして`app`を使用するアカウントを紐付ける.  

`twurl authorize`の引数に先程メモした`app`の`consumer key`と`consumer secret`を渡すと,  
認証用のURLが表示される.  

```bash
# 認証画面を開く
$ key='keykeykeykeykeykeykeykey'    # consumer key
$ secret='secretsecretsecretsecret' # consumer secret
$ twurl authorize --consumer-key $key --consumer-secret $secret
Go to https://api.twitter.com/oauth/authorize?oauth_consumer_key=keykeykeykeykeykeykeykey...&oauth_version=1.0 and paste in the supplied PIN
# メッセージに従ってURLをブラウザで開く
```

指定されたURLをブラウザで開くとアカウントに`app`を連携させるための確認画面が表示されるので,  
**Authorize app** を選択して自分のアカウントと`app`を連携させる.  
![Twitter](/images/2020-05-18/sc17.png)  

連携に成功すると7桁の`PIN`が表示されるので,  
これを`twurl`のメッセージに従って入力する.  

![Twitter](/images/2020-05-18/sc18.png)  

```bash
# 先程のつづき
$ twurl authorize --consumer-key $key --consumer-secret $secret
Go to https://api.twitter.com/oauth/authorize?oauth_consumer_key=keykeykeykeykeykeykeykey...&oauth_version=1.0 and paste in the supplied PIN
1234567 # PINを入力
Authorization successful

# 認証に成功したアカウントの確認
$ twurl accounts
uzimihsr
  keykeykeykeykeykeykeykey (default)
```

これで`twurl`と自分のアカウントが紐付いた.  

次に[ツイート投稿API](https://developer.twitter.com/en/docs/tweets/post-and-engage/api-reference/post-statuses-update)を叩いてツイートしてみる.  

`twurl`を使って  
`https://api.twitter.com/1.1/hoge/huga.json`  
のようなエンドポイントを叩く場合は,  
`twurl /1.1/hoge/huga.json`のようにすればいいみたい.  

```bash
# APIからツイートしてみる
# レスポンスがJSONで返ってくるのでjqでパースする
$ twurl -X POST -d 'status=そとちゃんかわいい' /1.1/statuses/update.json | jq
{
  "created_at": "Mon May 18 10:09:54 +0000 2020",
  "id": 1262324524051075000,
  "id_str": "1262324524051075074",
  "text": "そとちゃんかわいい",
  "truncated": false,
  "entities": {
    "hashtags": [],
    "symbols": [],
    "user_mentions": [],
    "urls": []
  },
  "source": "<a href=\"https://uzimihsr.github.io/\" rel=\"nofollow\">uzimihsr-twurl</a>",
  "in_reply_to_status_id": null,
  "in_reply_to_status_id_str": null,
  "in_reply_to_user_id": null,
  "in_reply_to_user_id_str": null,
  "in_reply_to_screen_name": null,
  "user": {
    "id": 1146420174272073700,
    "id_str": "1146420174272073733",
    "name": "ずみし",
    "screen_name": "uzimihsr",
    "location": "日本 東京",
    "description": "超絶かわいい元保護猫そとちゃんのしもべです",
    "url": "https://t.co/mM0q9F0x2t",
    "entities": {
      "url": {
        "urls": [
          {
            "url": "https://t.co/mM0q9F0x2t",
            "expanded_url": "https://instagram.com/uzimihsr",
            "display_url": "instagram.com/uzimihsr",
            "indices": [
              0,
              23
            ]
          }
        ]
      },
      "description": {
        "urls": []
      }
    },
    "protected": false,
    "followers_count": 35,
    "friends_count": 52,
    "listed_count": 0,
    "created_at": "Wed Jul 03 14:07:23 +0000 2019",
    "favourites_count": 32,
    "utc_offset": null,
    "time_zone": null,
    "geo_enabled": true,
    "verified": false,
    "statuses_count": 227,
    "lang": null,
    "contributors_enabled": false,
    "is_translator": false,
    "is_translation_enabled": false,
    "profile_background_color": "F5F8FA",
    "profile_background_image_url": null,
    "profile_background_image_url_https": null,
    "profile_background_tile": false,
    "profile_image_url": "http://pbs.twimg.com/profile_images/1220189466368692224/VkTo35n4_normal.jpg",
    "profile_image_url_https": "https://pbs.twimg.com/profile_images/1220189466368692224/VkTo35n4_normal.jpg",
    "profile_banner_url": "https://pbs.twimg.com/profile_banners/1146420174272073733/1584881012",
    "profile_link_color": "1DA1F2",
    "profile_sidebar_border_color": "C0DEED",
    "profile_sidebar_fill_color": "DDEEF6",
    "profile_text_color": "333333",
    "profile_use_background_image": true,
    "has_extended_profile": false,
    "default_profile": true,
    "default_profile_image": false,
    "following": false,
    "follow_request_sent": false,
    "notifications": false,
    "translator_type": "none"
  },
  "geo": null,
  "coordinates": null,
  "place": null,
  "contributors": null,
  "is_quote_status": false,
  "retweet_count": 0,
  "favorite_count": 0,
  "favorited": false,
  "retweeted": false,
  "lang": "ja"
}
```

実際に投稿されたツイートがこちら.  
連携した`app`(**uzimihsr-twurl**)から投稿されている.  

{{< tweet 1262324524051075074 >}}
https://twitter.com/uzimihsr/status/1262324524051075074  

やったぜ.  
コマンドラインから`Twitter API`を使ってツイートすることができた.  

## おわり
以上の手順で開発者アカウントの利用申請から`Twitter API`を用いたツイートまでをやってみた.  
`Twitter`大好き芸人なので`twurl`のコード読んだりAPIリファレンス読んだりしていろいろ遊んでみたい.  

## おまけ
テンションが高いときのねこ(かわいい)  
![そとちゃん](/images/2020-05-18/sotochan.jpg)  

## 参考
- [開発者アカウントの利用申請](#開発者アカウントの利用申請)
    - https://help.twitter.com/ja/rules-and-policies/twitter-api
    - https://developer.twitter.com/ja/apply-for-access
- [appの作成](#appの作成)
    - https://developer.twitter.com/ja/docs/basics/apps/overview
- [twurlを使ってツイート](#twurlを使ってツイート)
    - https://developer.twitter.com/en/docs/tutorials/using-twurl
    - https://github.com/twitter/twurl
    - https://developer.twitter.com/en/docs/tweets/post-and-engage/api-reference/post-statuses-update
