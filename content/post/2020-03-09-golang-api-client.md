---
title: "GoでWikipediaのAPIを叩いて記事検索した"
date: 2020-03-09T21:29:36+09:00
draft: false
tags: ["Go", "作業ログ"]
---

## 久しぶりのGo
最近k8sの勉強ばっかで開発っぽいことをやってなかったので, Go(golang)で遊んでみた.  

<!--more-->
---

## やったことのまとめ

- WikipediaのAPI(Wikimedia API)を叩いてみた
- Go(golang)でAPIクライアントもどきを作った

作ったクライアントはこんなかんじ.  
Wikipediaの記事を検索した結果とそのURLが表示できる.  
```bash
$ wikipedia -srlimit=5 -lang=en 'cat'
---------------------------------------------------
Cat
https://en.wikipedia.org/?curid=6678
---------------------------------------------------
Cat (disambiguation)
https://en.wikipedia.org/?curid=434590
---------------------------------------------------
.cat
https://en.wikipedia.org/?curid=1978706
---------------------------------------------------
Bengal cat
https://en.wikipedia.org/?curid=63064
---------------------------------------------------
Cat Stevens
https://en.wikipedia.org/?curid=78747
---------------------------------------------------
```

## つかうもの
- MacBook Pro (Retina, 15-inch, Mid 2015)
    - macOS Mojave 10.14
- Go(golang)
    - go version go1.13 darwin/amd64
- [Wikipedia API](https://www.mediawiki.org/wiki/API:Main_page/ja)

## やったこと

- [WikipediaAPIをたたく](#wikipediaapiをたたく)
- [GoでAPIをたたく](#goでapiをたたく)

### WikipediaAPIをたたく
[APIのページ](https://www.mediawiki.org/wiki/API:Main_page/ja)によると,  
Wikipedia(日本語版)APIのURLは  
https://ja.wikipedia.org/w/api.php  
となっている.  

今回は[記事検索API](https://www.mediawiki.org/wiki/API:Search)を使ってみる.  
記事検索をする場合はクエリパラメータに  
`action=query`, `list=search`, `srsearch=<検索したい文字列>`を指定して  
`GET`すればいいみたい.  
レスポンスボディの形式は`format=json`(JSONの場合)でできるっぽい.  

例: "猫"で検索する場合  
GET http://ja.wikipedia.org/w/api.php?format=json&action=query&list=search&srsearch=猫  

試しに`curl`で叩いてみる.  
デフォルトでは検索結果の上位10件が返ってくる.  

<details><summary>**APIでの検索結果**</summary><div>
```bash
# なんかリダイレクトされるみたいなのでLオプションは必須
# JSONはjqでいい感じに整形する
$ curl -sSL 'http://ja.wikipedia.org/w/api.php?action=query&format=json&list=search&srsearch=猫' | jq
{
  "batchcomplete": "",
  "continue": {
    "sroffset": 10,
    "continue": "-||"
  },
  "query": {
    "searchinfo": {
      "totalhits": 26227
    },
    "search": [
      {
        "ns": 0,
        "title": "ネコ",
        "pageid": 1215264,
        "size": 123050,
        "wordcount": 16730,
        "snippet": "黒<span class=\"searchmatch\">猫</span> - 全身の毛が黒色の<span class=\"searchmatch\">猫</span>。 白猫 - 全身の毛が白色の<span class=\"searchmatch\">猫</span>。 トラネコ（タビー） - トラのような縞模様がある<span class=\"searchmatch\">猫</span>。茶トラ<span class=\"searchmatch\">猫</span>、キジ<span class=\"searchmatch\">猫</span>、サバ<span class=\"searchmatch\">猫</span>など。 三毛<span class=\"searchmatch\">猫</span> - 3色（一般的に白・茶色・黒）の<span class=\"searchmatch\">猫</span>。 錆び<span class=\"searchmatch\">猫</span> - 黒と茶色の2色の<span class=\"searchmatch\">猫</span>。 はちわれ - 顔面が鼻筋を境にした八の字形の2色になっている<span class=\"searchmatch\">猫</span>。",
        "timestamp": "2020-03-03T14:46:03Z"
      },
      {
        "ns": 0,
        "title": "三毛猫ホームズシリーズ",
        "pageid": 227387,
        "size": 55409,
        "wordcount": 7737,
        "snippet": "収録作品　三毛猫ホームズの運動会・三毛<span class=\"searchmatch\">猫</span>ホームズのスクープ・三毛<span class=\"searchmatch\">猫</span>ホームズのバカンス・三毛<span class=\"searchmatch\">猫</span>ホームズの温泉旅行・三毛<span class=\"searchmatch\">猫</span>ホームズの殺人展覧会・三毛<span class=\"searchmatch\">猫</span>ホームズのバースデー・パーティ （9）三毛<span class=\"searchmatch\">猫</span>ホームズのびっくり箱 収録作品　三毛<span class=\"searchmatch\">猫</span>ホームズのびっくり箱・三毛<span class=\"searchmatch\">猫</span>ホームズの名演奏・三毛<span class=\"searchmatch\">猫</span>ホームズのパニック・三毛<span class=\"searchmatch\">猫</span>ホームズの幽霊退治・三毛<span class=\"searchmatch\">猫</span>",
        "timestamp": "2020-02-20T12:43:57Z"
      },
      {
        "ns": 0,
        "title": "1905年",
        "pageid": 2506,
        "size": 26827,
        "wordcount": 3310,
        "snippet": "が、1962年からは公式な場では使用されていない。 1月1日 - 日露戦争：旅順開城 1月1日 - 夏目漱石が『ホトトギス』1月号で、処女作『吾輩は<span class=\"searchmatch\">猫</span>である』を連載開始 1月22日 - サンクトペテルブルクで血の日曜日事件発生 1月23日 - 奈良県鷲家口（現・吉野郡東吉野村）でニホンオオカミの捕",
        "timestamp": "2020-01-23T07:22:49Z"
      },
      {
        "ns": 0,
        "title": "吾輩は猫である",
        "pageid": 13241,
        "size": 32502,
        "wordcount": 4469,
        "snippet": "『吾輩は<span class=\"searchmatch\">猫</span>である』（わがはいはねこである）は、夏目漱石の長編小説であり、処女小説である。1905年（明治38年）1月、『ホトトギス』に発表され、好評を博したため、翌1906年（明治39年）8月まで継続した。 「吾輩は<span class=\"searchmatch\">猫</span>である。名前はまだ無い。どこで生れたかとんと見当がつかぬ。」という書き出しで始ま",
        "timestamp": "2020-02-03T23:11:05Z"
      },
      {
        "ns": 0,
        "title": "猫騙し",
        "pageid": 229647,
        "size": 3049,
        "wordcount": 432,
        "snippet": "<span class=\"searchmatch\">猫</span>騙し（ねこだまし）とは相撲の戦法の一種である。 立合いと同時に相手力士の目の前に両手を突き出して掌を合わせて叩くもので、相手の目をつぶらせることを目的とする奇襲戦法の一つ。相手に隙を作り、有利な体勢を作るために使われる。普通の立合いではかなわないような、はるかに強い相手に対する一発勝負に使われる",
        "timestamp": "2019-07-18T01:55:22Z"
      },
      {
        "ns": 0,
        "title": "クイズRPG 魔法使いと黒猫のウィズ",
        "pageid": 2969296,
        "size": 32428,
        "wordcount": 4747,
        "snippet": "『クイズRPG 魔法使いと黒<span class=\"searchmatch\">猫</span>のウィズ』（クイズRPG まほうつかいとくろねこのウィズ）は、2013年にコロプラで配信を開始したソーシャルゲーム。 2013年3月にAndroid版が、4月22日にiOS版が配信開始された。 2013年8月20日に英語版を日本・韓国・中国以外の全世界で、韓国語版を韓国で、それぞれGoogle",
        "timestamp": "2020-03-03T09:36:04Z"
      },
      {
        "ns": 0,
        "title": "長靴猫シリーズ",
        "pageid": 2623926,
        "size": 26739,
        "wordcount": 3221,
        "snippet": "『長靴<span class=\"searchmatch\">猫</span>シリーズ』（ながぐつねこシリーズ）は、ペローの童話『長靴をはいた<span class=\"searchmatch\">猫</span>』を原作とする、東映動画（現：東映アニメーション）製作による劇場版長編アニメーション映画シリーズの通称。『長靴をはいた<span class=\"searchmatch\">猫</span>』（1969年）、『ながぐつ三銃士』（1972年）、『長靴をはいた<span class=\"searchmatch\">猫</span>",
        "timestamp": "2020-02-20T12:09:42Z"
      },
      {
        "ns": 0,
        "title": "三味線",
        "pageid": 19199,
        "size": 17244,
        "wordcount": 2591,
        "snippet": "三味線（しゃみせん）は、日本の有棹弦楽器。もっぱら弾(はじ)いて演奏される撥弦楽器である。四角状の扁平な木製の胴の両面に<span class=\"searchmatch\">猫</span>や犬の皮を張り、胴を貫通して伸びる棹に張られた弦を、通常、銀杏形の撥（ばち）で弾き演奏する。 成立は15世紀から16世紀にかけてとされ、戦国時代に琉球（現在の沖縄県）から伝来し",
        "timestamp": "2020-02-26T01:02:18Z"
      },
      {
        "ns": 0,
        "title": "グーグーだって猫である",
        "pageid": 1181328,
        "size": 17396,
        "wordcount": 1792,
        "snippet": "『グーグーだって<span class=\"searchmatch\">猫</span>である』（グーグーだってねこである）は大島弓子の漫画作品、およびそれを原作とした映画作品およびテレビドラマ作品。 タイトル・ロールとなっているアメリカンショートヘアの<span class=\"searchmatch\">猫</span>、「グーグー」を始めとする<span class=\"searchmatch\">猫</span>たちと作者との生活を綴ったエッセイ漫画。『ヤングロゼ』1996年11月号から1997",
        "timestamp": "2020-02-26T13:44:59Z"
      },
      {
        "ns": 0,
        "title": "迷い猫オーバーラン!",
        "pageid": 1727982,
        "size": 70796,
        "wordcount": 9877,
        "snippet": "PJ ライトノベル ポータル 文学 『迷い<span class=\"searchmatch\">猫</span>オーバーラン！』（まよいねこオーバーラン！）は、松智洋による日本のライトノベル。 集英社スーパーダッシュ文庫より、2008年10月から全12巻が刊行されている。イラストは9巻までぺこが担当していたが、10巻はヤス、11巻は氷川へきる、最終12巻はみつみ美",
        "timestamp": "2020-01-25T04:22:18Z"
      }
    ]
  }
}
```
</div></details>

[WikipediaのUIで検索した場合](https://ja.wikipedia.org/w/index.php?sort=relevance&search=%E7%8C%AB&title=%E7%89%B9%E5%88%A5:%E6%A4%9C%E7%B4%A2&profile=advanced&fulltext=1&advancedSearch-current=%7B%7D&ns0=1)と同じような結果が得られることがわかる.  
いい感じ.  
これでAPIの動作確認は完了.  

### GoでAPIをたたく
このままシェルスクリプト化しても便利だと思うんだけど,  
今回は`Go`の練習としてAPIクライアントっぽく作ってみる.  

<details><summary>`main.go`</summary><div>
```go
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"net/url"
	"os"
)

// JSONをパースするための構造体を定義
type WikipediaResponse struct {
	Query Query `json:"query"`
}

type Query struct {
	SearchInfo SearchInfo `json:"searchinfo"`
	Search     []Search   `json:"search"`
}

type SearchInfo struct {
	Totalhits int `json:"totalhits"`
}

type Search struct {
	Title  string `json:"title"`
	PageId int    `json:"pageid"`
	// Snippet string `json:"snippet"`
}

func main() {
	// 引数チェック
	var language = flag.String("lang", "ja", "検索するwikiの言語. default: ja")
	var srlimit = flag.Int("srlimit", 10, "検索件数. default: 10")
	flag.Parse()
	if flag.NArg() != 1 {
		fmt.Println("検索ワードを指定してください")
		os.Exit(1)
	}
	if flag.NFlag() > 2 {
		fmt.Println("言語以外のフラグは無効です")
		os.Exit(1)
	}
	arg := flag.Arg(0)

	// APIを叩くためのURL作成
	baseUrl := url.URL{}
	baseUrl.Scheme = "http"
	baseUrl.Host = fmt.Sprintf("%s.wikipedia.org", *language)
	baseUrl.Path = "w/api.php"
	query := baseUrl.Query()
	query.Set("action", "query")
	query.Set("list", "search")
	query.Set("srsearch", arg)
	query.Set("srlimit", fmt.Sprintf("%d", *srlimit))
	query.Set("format", "json")
	baseUrl.RawQuery = query.Encode()

	// 記事検索APIを叩く
	resp, err := http.Get(baseUrl.String())
	if err != nil {
		log.Fatal(err)
	}
	defer resp.Body.Close()

	// レスポンスをパースする
	body, err := ioutil.ReadAll(resp.Body)
	wikipediaResponse := new(WikipediaResponse)
	err = json.Unmarshal(body, wikipediaResponse)

	// ヒットした記事が0件の場合は終了
	if wikipediaResponse.Query.SearchInfo.Totalhits <= 0 {
		fmt.Println("記事が見つかりませんでした。")
		os.Exit(1)
	}

	// 記事タイトルとURLを表示
	for _, v := range wikipediaResponse.Query.Search {
		fmt.Println("---------------------------------------------------")
		fmt.Println(v.Title)
		fmt.Printf("https://%s.wikipedia.org/?curid=%d\n", *language, v.PageId)
	}
	fmt.Println("---------------------------------------------------")

}
```
</div></details>

やったこととしては  

- 引数でのクエリパラメータとオプションの受け付け
- リクエストURLの組み立てとAPIへのHTTPリクエストの実行
- レスポンス(JSON)のパース
- 必要な情報の表示

だけ.  
特に難しかったのはAPIを叩く部分とレスポンスのパース部分.  

APIを叩く部分については[`net/url`](https://golang.org/pkg/net/url/)を使ってURLを組み立てて,  
[`net/http`](https://golang.org/pkg/net/http/)を使ってリクエストを投げるようにした.  
基本的にはcurlで叩いたときと同じリクエストを送るようにした.  

公式パッケージ[`encoding/json`](https://golang.org/pkg/encoding/json/)を使ったJSONのパースは結構面倒で,  
事前にJSONの構造を構造体として定義してやる必要がある.  
APIのレスポンスはJSONが結構入れ子になっているので書くのが大変だった.  

また, 今回はレスポンスのJSONから記事タイトルとidだけ抜き出して,  
https://ja.wikipedia.org/?curid=1215264 の形式にすることでページへのリンクを作成するようにした.  

実際に使ってみるとこんな感じ.  

```bash
# ビルドする
$ ls
go.mod  main.go
$ go build -o $GOPATH/bin/wikipedia .

# GOPATHがPATHに入っていればwikipediaコマンドを呼び出せる
$ wikipedia '猫'
---------------------------------------------------
ネコ
https://ja.wikipedia.org/?curid=1215264
---------------------------------------------------
三毛猫ホームズシリーズ
https://ja.wikipedia.org/?curid=227387
---------------------------------------------------
1905年
https://ja.wikipedia.org/?curid=2506
---------------------------------------------------
吾輩は猫である
https://ja.wikipedia.org/?curid=13241
---------------------------------------------------
猫騙し
https://ja.wikipedia.org/?curid=229647
---------------------------------------------------
クイズRPG 魔法使いと黒猫のウィズ
https://ja.wikipedia.org/?curid=2969296
---------------------------------------------------
長靴猫シリーズ
https://ja.wikipedia.org/?curid=2623926
---------------------------------------------------
三味線
https://ja.wikipedia.org/?curid=19199
---------------------------------------------------
グーグーだって猫である
https://ja.wikipedia.org/?curid=1181328
---------------------------------------------------
迷い猫オーバーラン!
https://ja.wikipedia.org/?curid=1727982
---------------------------------------------------

# 実行時引数でオプションを変えられる
$ wikipedia -lang=en -srlimit=3 'イチロー'
---------------------------------------------------
Ichirō
https://en.wikipedia.org/?curid=1067866
---------------------------------------------------
Orix Buffaloes
https://en.wikipedia.org/?curid=1145207
---------------------------------------------------
Ichiro Suzuki
https://en.wikipedia.org/?curid=66417
---------------------------------------------------
```
なんかそれっぽいのが作れた.  
やったぜ.  

つくったやつはここ.  
https://github.com/uzimihsr/wikipedia-search  

APIクライアントはこんな感じで一度作ってしまえば他にもいろいろできそう.  
時間があれば他のAPIにも対応させてwikipedia用のコマンドラインツールみたいなのを作っても面白いかもしれない(需要はなさそう).  

## おまけ
のび〜をするねこ  
![ストレッチそとちゃん](/images/2020-03-09-sotochan.jpg)  
