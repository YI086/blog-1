---
title: "awkコマンドを使って文字列の先頭と末尾を削除する"
date: 2020-06-29T21:19:05+09:00
draft: false
tags: ["メモ", "Shell", "awk"]
---

## awkすき
Shellで文字列を扱うときにちょっとだけ困ったのでメモ.  

<!--more-->
---

## まとめ
`substr()`と`length()`を使う.  
{{< highlight bash >}}
# 先頭から1文字削除
$ echo 'abcdefghijkl' | awk '{print substr($0, 2)}'
bcdefghijkl

# 末尾から1文字削除
$ echo 'abcdefghijkl' | awk '{print substr($0, 1, length($0)-1)}'
abcdefghijk

# 先頭と末尾から1文字ずつ削除
$ echo 'abcdefghijkl' | awk '{print substr($0, 2, length($0)-2)}'
bcdefghijk
{{< /highlight >}}

## 環境
- macOS Mojave 10.14
- awk version 20070501

## やりかた
`substr(string, start, length)`は文字列(`string`)の頭n文字目(`start`)からm文字(`length`)を抜き出して表示する.  
引数の`length`は省略可能で, 省略した場合は文末まで表示される.  

{{< highlight bash >}}
# qwerty の2文字目(w)から末尾まで抽出
$ awk 'BEGIN {print substr("qwerty", 2)}'
werty

# qwerty の2文字目(w)から4文字ぶんを抽出
$ awk 'BEGIN {print substr("qwerty", 2, 4)}'
wert
{{< /highlight >}}

先頭から文字を抜き出せたが, 末尾から文字を抜き出したい場合もある.  
`length(string)`は文字列(`string`)の文字数を数えてくれるので, これと`substr()`を組み合わせてみる.  

{{< highlight bash >}}
# qwertyの文字数は6
$ awk 'BEGIN {print length("qwerty")}'
6

# qwerty の1文字目(q)から(length-1=5)文字ぶんを抽出
$ awk 'BEGIN {print substr("qwerty", 1, length("qwerty")-1)}'
qwert
{{< /highlight >}}

あとはパイプでつないでいい感じにすればいい.  

{{< highlight bash >}}
# ダブルクォート("")で囲まれた文字列("qwerty")から文字列(qwerty)のみを抽出
$ echo '"qwerty"' | awk '{print substr($0, 2, length($0)-2)}'
qwerty
{{< /highlight >}}

やったぜ.  

## おわり
やっぱり`awk`は神.  
もっと使いこなしていきたい.  

## おまけ
テレワーク中のしもべの真似をするねこ  
![そとちゃん](/images/2020-06-29/sotochan.jpg)  

## 参考
- https://www.gnu.org/software/gawk/manual/html_node/String-Functions.html  
