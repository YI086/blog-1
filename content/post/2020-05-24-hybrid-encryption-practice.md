---
title: "OpenSSLで共通鍵暗号方式とハイブリッド暗号方式を試す"
date: 2020-05-24T17:09:44+09:00
draft: false
tags: ["メモ", "セキュリティ", "OpenSSL"]
---

## 欠点を補いあう
共通鍵暗号方式とハイブリッド暗号方式についても実際に触りながらまとめる.  

<!--more-->
---

## まとめ

**共通鍵でのデータの暗号化と復号化**
```bash
# 32bytesの乱数パスワードファイル(password)の作成
$ openssl rand -base64 -out password 32

# 平文データ(data)を公開鍵(password)を使ってaes256で暗号化(encrypted-data)
$ openssl enc -aes256 -in data -out encrypted-data -pass file:password -base64

# 暗号データ(encrypted-data)を公開鍵(password)を使ってaes256で復号化(decrypted-data)
$ openssl enc -d -aes256 -in encrypted-data -out decrypted-data -pass file:password -base64
```

**ハイブリッド暗号方式**
```bash
# 秘密鍵(private-key.pem)と公開鍵(public-key.pem)の作成
# 作成した公開鍵はもう片方に渡す
$ openssl genrsa -out private-key.pem
$ openssl rsa -in private-key.pem -pubout -out public-key.pem

# 共通鍵(password)の作成 (秘密鍵を持たない方が実行)
$ openssl rand -base64 -out password 32

# 公開鍵(public-key.pem)を使って共通鍵(password)を暗号化(encrypted-password) (秘密鍵を持たない方が実行)
# 暗号化した共通鍵をもう片方に渡す
$ openssl rsautl -encrypt -in password -out encrypted-password -inkey public-key.pem -pubin

# 秘密鍵(private-key.pem)を使って暗号化された共通鍵(encrypted-password)を復号化(decrypted-password) (秘密鍵を持つ方が実行)
$ openssl rsautl -decrypt -in encrypted-password -out decrypted-password -inkey private-key.pem

# 以降は共通鍵で暗号化したデータをやりとりする
```


## 環境
- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
- openssl
    - OpenSSL 1.1.1c  28 May 2019
    - ラズパイの初期装備

## もくじ

- [共通鍵暗号方式](#共通鍵暗号方式)
- [公開鍵暗号方式と共通鍵暗号方式の比較](#公開鍵暗号方式と共通鍵暗号方式の比較)
- [ハイブリッド暗号方式](#ハイブリッド暗号方式)

### 共通鍵暗号方式

共通鍵暗号方式と言っても,  
(誤解を恐れずに言えば)パスワードでデータを暗号化/復号化する方式のこと.  

共通鍵暗号方式の通信の簡単な流れとしては

1. 送信側と受信側で事前に同じ共通鍵(パスワード)を持っておく
1. 送信側は平文データを共通鍵で暗号化する
1. 送信側は受信側に暗号データを渡す
1. 受信側は暗号データを共通鍵で復号化する

たったこれだけ.  
暗号化/復号化に同じ鍵を使うことが大きな特徴.  

実際にやってみる.  

まずはじめに共通鍵(パスワード)を作成して送信側と受信側で共有する.  

```bash
# ディレクトリを送信側(dirA)と受信側(dirB)に見立てて作業
$ mkdir dirA dirB
$ cd dirA

# 共通鍵(password)を作成
# 本当はこんな適当な値ではなく乱数が望ましい
$ echo 1234567 > password
$ cat password
1234567

# 共通鍵(password)を受信側(dirB)に共有
$ cp password ../dirB/
$ ls
password
$ ls ../dirB
password
```

次に平文データを共通鍵で暗号化する.  
これにより, 共通鍵がわからない場合はデータの内容を読むことができなくなる.  

```bash
# 送信側(dirA)で作業
$ pwd
/path/to/dirA

# 平文データ(data.txt)を作成
$ echo abcdefg > data.txt

# 暗号化アルゴリズム(aes256)を使って平文データ(data.txt)を暗号化(encrypted-data.txt)
# アルゴリズムが非推奨のものなので警告が出ているがとりあえずはOK
$ openssl enc -aes256 -in data.txt -out encrypted-data.txt -pass file:password
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
$ ls
data.txt  encrypted-data.txt  password

# 暗号化されているので読めない
$ cat encrypted-data.txt
Salted__aGo�r�ʵ�3�Z��ċ�D
```

暗号データを受信側に渡し,  
受信側で共通鍵を使って復号化する.  

```bash
# 暗号データ(encrypted-data.txt)を受信側(dirB)に渡す
$ pwd
/path/to/dirA
$ cp encrypted-data.txt ../dirB/

# 受信側(dirB)で作業
$ cd ../dirB
$ ls
encrypted-data.txt  password

# 暗号データ(encrypted-data.txt)を共通鍵(password)で復号化(decrypted-data.txt)
$ openssl enc -d -aes256 -in encrypted-data.txt -out decrypted-data.txt -pass file:password
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
$ cat decrypted-data.txt
abcdefg
```

こんな感じでデータの暗号化と復号化ができる.  

重要なのは, 送信側と受信側の間で暗号データの他に共通鍵の受け渡しをしていること.  
共通鍵暗号方式では同じ鍵で暗号化と復号化が行われるため,  
通信が傍受され共通鍵が流出した場合は簡単に平文データが取り出されてしまう.  
(いわゆる鍵配送問題)  

### 公開鍵暗号方式と共通鍵暗号方式の比較

[公開鍵暗号方式](https://uzimihsr.github.io/post/2020-05-20-public-key-practice/)と共通鍵暗号方式の大きな違いとしては  
公開鍵暗号方式が暗号化/復号化を **別々の鍵** で行うのに対し,  
共通鍵暗号方式では暗号化/復号化を **同じ鍵** で行うこと.  

これらの特徴を比較すると次のようになる.  

- 共通鍵暗号方式
    - 公開鍵暗号方式よりも暗号化/復号化が高速
        - アルゴリズムが簡単なため
    - 通信を傍受されると無力
- 公開鍵暗号方式
    - 暗号化/復号化が遅い
    - 通信を傍受されても問題ない

これらの特徴をうまく組み合わせて高速で安全な通信を行うのがハイブリッド暗号方式.  

### ハイブリッド暗号方式

ハイブリッド暗号方式の大きな特徴は,  
**共通鍵そのものを公開鍵暗号方式で渡すこと**.  

主な流れはこんな感じ.  

1. 送信側で共通鍵を作成
1. 受信側で秘密鍵と公開鍵を作成, 送信側に公開鍵を渡す
1. 送信側は受け取った公開鍵で共通鍵を暗号化して受信側に渡す
1. 受信側は暗号化された共通鍵を秘密鍵で復号化
1. 以降は2者間で共通鍵で暗号化したデータをやり取りする

最初の1回だけ公開鍵暗号方式で共通鍵を渡した後は高速な共通鍵暗号方式を使うことができるので,  
高速で安全にデータをやり取りするためのSSL/TLS通信では公開鍵暗号方式単体ではなくこちらが使用されている.  

実際にやってみる.  

まずは送信側で共通鍵を作成する.  

```bash
# ディレクトリを送信側(dir1)と受信側(dir2)に見立てて作業
$ mkdir dir1 dir2

# 送信側(dir1)で共通鍵(password)を作成
# 今回はちゃんと32bytesの乱数で作成する
$ cd dir1
$ openssl rand -base64 -out password 32
$ cat password
VAJhXVUt7aRxfFs5ba7SNkjWyIqOaI8E10t0tnmF7as=
```

次に受信側で秘密鍵と公開鍵を作成して公開鍵を送信側に渡す.  

```bash
# 受信側(dir2)で秘密鍵(private-key.pem)と公開鍵(public-key.pem)を作成, 送信側(dir1)に公開鍵を渡す
$ cd ../dir2
$ openssl genrsa -out private-key.pem
$ openssl rsa -in private-key.pem -pubout -out public-key.pem
$ cp ./public-key.pem ../dir1/ # 公開鍵を渡す
```

送信側は受け取った公開鍵で共通鍵を暗号化して受信側に渡す.  

```bash
# 送信側(dir1)は受け取った公開鍵(public-key.pem)で共通鍵(password)を暗号化(encrypted-password)して受信側に渡す
$ cd ../dir1
$ openssl rsautl -encrypt -in password -out encrypted-password -inkey public-key.pem -pubin
$ cat encrypted-password
��ʞ��...
$ cp ./encrypted-password ../dir2/ # 暗号化された共通鍵を渡す
```

最後に受信側は暗号化された共通鍵を秘密鍵で復号化する.  

```bash
# 受信側(dir2)は暗号化された共通鍵(encrypted-password)を秘密鍵(private-key.pem)で復号化(decrypted-password)
$ cd ../dir2
$ openssl rsautl -decrypt -in encrypted-password -out decrypted-password -inkey private-key.pem
$ cat decrypted-password
VAJhXVUt7aRxfFs5ba7SNkjWyIqOaI8E10t0tnmF7as=
```

以上の手順で送信側, 受信側双方に共通鍵が存在する状態になったので,  
以降は2者間で共通鍵暗号方式で通信する.  

```bash
# 以降は2者間で共通鍵で暗号化したデータをやり取りする
$ echo hello > file1
$ openssl enc -aes256 -in file1 -out encrypted-file1 -pass file:decrypted-password -base64
$ cp ./encrypted-file1 ../dir1/ # 暗号データを渡す
$ cd ../dir1
$ openssl enc -d -aes256 -in encrypted-file1 -out decrypted-file1 -pass file:password -base64
$ cat decrypted-file1
hello
$ echo world > file2
$ openssl enc -aes256 -in file2 -out encrypted-file2 -pass file:password -base64
$ cp ./encrypted-file2 ../dir2/ # 暗号データを渡す
$ cd ../dir2
$ openssl enc -d -aes256 -in encrypted-file2 -out decrypted-file2 -pass file:decrypted-password -base64
$ cat decrypted-file2
world
```

重要なのは暗号データの他に2者間でやり取りされるデータが **公開鍵, 公開鍵で暗号化された共通鍵のみ** であること.  
[公開鍵暗号方式](https://uzimihsr.github.io/post/2020-05-20-public-key-practice/)の性質として公開鍵では暗号化された共通鍵の復号化ができず,  
暗号化された状態の共通鍵では暗号データを復号化できないため,  
仮に悪意のある第三者がこれらの通信を傍受してもやりとりされる平文データの内容を読むことはできない(秘密が守られる).  

## おわり
[公開鍵暗号方式](https://uzimihsr.github.io/post/2020-05-20-public-key-practice/)に続いて共通鍵暗号方式とハイブリッド暗号方式を実際に試してみた.  
これでも十分そうに見えるけど, まだ公開鍵の信頼性にまつわる問題があるので次はデジタル証明書について触ってみたい.  

## おまけ
あくびねこ  
![そとちゃん](/images/2020-05-24/sotochan.jpg)

## 参考

- [共通鍵暗号方式](#共通鍵暗号方式)
    - https://www.openssl.org/docs/man1.1.0/man1/enc.html
    - https://www.openssl.org/docs/man1.1.0/man1/openssl.html
- [公開鍵暗号方式と共通鍵暗号方式の比較](#公開鍵暗号方式と共通鍵暗号方式の比較)
    - [アルゴリズム図鑑 絵で見てわかる26のアルゴリズム](https://www.shoeisha.co.jp/book/detail/9784798149776)
        - ほんとにわかりやすい
- [ハイブリッド暗号方式](#ハイブリッド暗号方式)
    - https://www.openssl.org/docs/man1.1.0/man1/rand.html
