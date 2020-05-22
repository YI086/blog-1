---
title: "OpenSSLで秘密鍵と公開鍵を作る"
date: 2020-05-20T20:18:54+09:00
draft: false
tags: ["メモ", "セキュリティ", "OpenSSL"]
---

## 暗号わかんね
公開鍵暗号方式についてよくわかってないままだったので実際に触りながら自分なりにまとめる.  

<!--more-->
---

## まとめ

**秘密鍵と公開鍵の作成**  
```bash
# 秘密鍵(private-key.pem)を作成
## パスフレーズなし
$ openssl genrsa -out private-key.pem
## パスフレーズあり(aes256で暗号化)
$ openssl genrsa -out private-key.pem -aes256

# 秘密鍵(private-key.pem)から公開鍵(public-key.pem)を作成
$ openssl rsa -in private-key.pem -pubout -out public-key.pem
```
**データの暗号化と復号化**
```bash
# 公開鍵(public-key.pem)を使ってデータ(data.txt)を暗号化(encrypted-data.txt)
$ openssl rsautl -encrypt -in data.txt -out encrypted-data.txt -inkey public-key.pem -pubin

# 秘密鍵(private-key.pem)を使って暗号データ(encrypted-data.txt)を復号化(decrypted-data.txt)
$ openssl rsautl -decrypt -in encrypted-data.txt -out decrypted-data.txt -inkey private-key.pem
```
**秘密鍵のパスフレーズ**
```bash
# 秘密鍵(private-key.pem)にパスフレーズをつける(private-key-with-pass-phrase.pem)
$ openssl rsa -in private-key.pem -out private-key-with-pass-phrase.pem -aes256

# 秘密鍵(private-key-with-pass-phrase.pem)のパスフレーズを外す(private-key-without-pass-phrase.pem)
$ openssl rsa -in private-key-with-pass-phrase.pem -out private-key-without-pass-phrase.pem
```


## 環境
- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
- openssl
    - OpenSSL 1.1.1c  28 May 2019
    - ラズパイの初期装備

## もくじ

- [秘密鍵と公開鍵の性質](#秘密鍵と公開鍵の性質)
- [秘密鍵と公開鍵を作る](#秘密鍵と公開鍵を作る)
- [公開鍵で暗号化する](#公開鍵で暗号化する)
- [秘密鍵で復号化する](#秘密鍵で復号化する)
- [パスフレーズをつける](#パスフレーズをつける)

### 秘密鍵と公開鍵の性質

まず秘密鍵と公開鍵の性質について.  
数学が苦手なので詳細なアルゴリズムとか数式とかは抜きにして次の性質がある前提で進める.  

- 秘密鍵と公開鍵は鍵生成アルゴリズムによって作成される数値
- 公開鍵から秘密鍵を逆算することはできない
- 公開鍵を用いてデータを暗号化する
- 暗号化されたデータは秘密鍵によってのみ復号化できる

とはいえ, 言葉だとよくわかんないので実際に作ってみるのが手っ取り早い.  

### 秘密鍵と公開鍵を作る

`openssl`を使ってまずは秘密鍵と公開鍵を作ってみる.  

**秘密鍵と公開鍵は鍵生成アルゴリズムによって作成される** のだが,  
今回は鍵生成アルゴリズムとして`RSA`を使う.  
アルゴリズムの詳細は省くけど, 素数が関係しているくらいは覚えておいてもいいはず.  

`RSA`の秘密鍵の作成自体はかんたんで, コマンド1行でできる.  

```bash
# 適当なディレクトリを作って作業
$ mkdir dirA
$ cd dirA

# パスフレーズなしの秘密鍵(private-key.pem)を作成する
$ openssl genrsa -out private-key.pem
Generating RSA private key, 2048 bit long modulus (2 primes)
.........................................+++++
...............................+++++
e is 65537 (0x010001)

# 秘密鍵が生成される
$ ls
private-key.pem

# そのまま表示しても読めない
$ cat private-key.pem
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA58HW0I+9qZiHopidx7pUuxNzVuobgzgp7OswlvgSHn2/BbKL
...
fSy2dXrjU08OXro9YHz0d6XESG3feSinK0ND0sbsrKTD+4j7fetk
-----END RSA PRIVATE KEY-----
```

次に秘密鍵の内容を確認してみる.  

秘密鍵には公開鍵の情報と鍵を作成したときの情報とかが入ってる.  
(例: `modulus`が公開鍵の情報で, `prime1`とか`prime2`とあるのが鍵生成時に使用した素数の情報)  

```bash
# 秘密鍵(private-key.pem)の中身を確認する
$ openssl rsa -in private-key.pem -text -noout
RSA Private-Key: (2048 bit, 2 primes)
modulus:
    00:e7:c1:d6:d0:8f:bd:a9:98:87:a2:98:9d:c7:ba:
    54:bb:13:73:56:ea:1b:83:38:29:ec:eb:30:96:f8:
    12:1e:7d:bf:05:b2:8b:c5:ff:33:fb:b1:7d:be:a6:
    7b:ec:62:5c:b2:d3:dd:b7:34:ec:5f:b2:1a:56:63:
    27:d4:f4:e0:7d:10:2b:29:8b:16:6e:f1:ce:7b:73:
    31:67:39:ca:cf:df:d3:fb:14:15:94:85:80:f4:2a:
    ee:c6:93:79:ff:81:09:51:55:29:14:e7:d3:dd:87:
    d4:82:2f:c2:80:5c:e3:89:60:c6:a7:9a:43:b3:0d:
    33:98:d1:05:1c:20:38:a4:dd:19:39:b8:0b:d1:6e:
    84:8f:e9:55:df:fd:70:16:c5:f8:5d:66:6a:03:13:
    36:7b:ab:e1:7e:30:7e:76:6b:e0:50:2d:cb:ad:59:
    9c:db:66:ba:2e:9d:61:50:ad:0f:2f:fe:22:7f:bb:
    ef:91:f5:70:02:bd:71:2f:6a:93:2f:db:85:fd:3e:
    28:cf:39:d1:d4:39:32:e8:c7:a0:3b:10:98:36:44:
    69:d3:15:57:bf:59:53:0c:88:01:02:78:9f:69:f6:
    1b:c2:20:5a:e0:0e:b2:1f:ed:10:c1:bf:9d:d0:17:
    9d:0a:7a:25:8d:84:bc:39:f1:1b:74:11:47:e0:68:
    03:21
publicExponent: 65537 (0x10001)
privateExponent:
    79:5d:79:31:1f:15:23:8b:4c:fc:49:0f:d7:58:2c:
    ...
prime1:
    00:fc:4f:d2:2b:d8:79:fa:6d:ca:c4:3f:b1:fe:67:
    ...
prime2:
    00:eb:25:19:b7:ca:02:f0:9a:09:ea:7f:19:03:40:
    ...
exponent1:
    00:e8:69:5e:5f:a4:f8:37:06:0b:50:da:9b:4a:8c:
    ...
exponent2:
    22:f6:11:2c:d2:4c:3d:99:a9:7f:c4:05:e4:05:db:
    ...
coefficient:
    4a:9f:3d:f5:6f:b6:83:02:f0:57:47:03:c3:d5:8c:
    ...
```

次に秘密鍵のデータから公開鍵を取り出してみる.  
公開鍵の中身が秘密鍵の中身の`modulus`と同じになっていることが確認できる.  

```bash
# 秘密鍵から公開鍵(public-key.pem)を出力する
$ openssl rsa -in private-key.pem -pubout -out public-key.pem
writing RSA key

# 公開鍵が生成される
$ ls
private-key.pem  public-key.pem

# 公開鍵もそのままでは読めない
$ cat public-key.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA58HW0I+9qZiHopidx7pU
...
IQIDAQAB
-----END PUBLIC KEY-----

# 公開鍵の中身を確認する
$ openssl rsa -in public-key.pem -text -noout -pubin
RSA Public-Key: (2048 bit)
Modulus:
    00:e7:c1:d6:d0:8f:bd:a9:98:87:a2:98:9d:c7:ba:
    54:bb:13:73:56:ea:1b:83:38:29:ec:eb:30:96:f8:
    12:1e:7d:bf:05:b2:8b:c5:ff:33:fb:b1:7d:be:a6:
    7b:ec:62:5c:b2:d3:dd:b7:34:ec:5f:b2:1a:56:63:
    27:d4:f4:e0:7d:10:2b:29:8b:16:6e:f1:ce:7b:73:
    31:67:39:ca:cf:df:d3:fb:14:15:94:85:80:f4:2a:
    ee:c6:93:79:ff:81:09:51:55:29:14:e7:d3:dd:87:
    d4:82:2f:c2:80:5c:e3:89:60:c6:a7:9a:43:b3:0d:
    33:98:d1:05:1c:20:38:a4:dd:19:39:b8:0b:d1:6e:
    84:8f:e9:55:df:fd:70:16:c5:f8:5d:66:6a:03:13:
    36:7b:ab:e1:7e:30:7e:76:6b:e0:50:2d:cb:ad:59:
    9c:db:66:ba:2e:9d:61:50:ad:0f:2f:fe:22:7f:bb:
    ef:91:f5:70:02:bd:71:2f:6a:93:2f:db:85:fd:3e:
    28:cf:39:d1:d4:39:32:e8:c7:a0:3b:10:98:36:44:
    69:d3:15:57:bf:59:53:0c:88:01:02:78:9f:69:f6:
    1b:c2:20:5a:e0:0e:b2:1f:ed:10:c1:bf:9d:d0:17:
    9d:0a:7a:25:8d:84:bc:39:f1:1b:74:11:47:e0:68:
    03:21
Exponent: 65537 (0x10001)
```

このように秘密鍵からは公開鍵の中身が得られるが,  
**公開鍵から秘密鍵を逆算することはできない** ので,  
公開鍵は自由に公開しても問題ない.  
(正しくは逆算は完全に不可能ではないんだけど, やろうとすると非現実的な計算時間が必要になるので誰もやろうとしない)  

### 公開鍵で暗号化する

秘密鍵と公開鍵が作成できたので,  
まずは試しに **公開鍵を用いてデータを暗号化する**.  

ここからは2つのディレクトリ(**dirA**, **dirB**)をデータの受信側, 送信側に見立てて進める.  
共通鍵暗号方式でデータをやり取りするには, まず受信側(**dirA**)の公開鍵を送信側(**dirB**)に渡す.  

```bash
# 秘密鍵を作ったのとは別のディレクトリで作業
$ cd ..
$ mkdir dirB
$ ls
dirA  dirB

# 受信側(dirA)の公開鍵を送信側(dirB)に送る
$ cp dirA/public-key.pem dirB/
```

次に送信側(**dirB**)は受信側からもらった公開鍵でデータ(**data.txt**)を暗号化する.   
公開鍵と平文データで数値計算をごにゃごにゃやった結果の値が暗号データとなる.  

```bash
# 送信側で適当な平文データを作る
$ cd dirB
$ echo abcdefg > data.txt
$ cat data.txt
abcdefg
$ ls
data.txt  public-key.pem

# 公開鍵を使ってデータを暗号化する
$ openssl rsautl -encrypt -in data.txt -out encrypted-data.txt -inkey public-key.pem -pubin

# 暗号化されたデータが出力される
$ ls
data.txt  encrypted-data.txt  public-key.pem

# そのまま表示しようとしても意味不明な状態
$ cat encrypted-data.txt
�����...
```

**暗号化されたデータは秘密鍵によってのみ復号化できる** ので,  
この暗号データ(**encrypted-data.txt**)を公開鍵で復号化することはできない.  

```bash
# 無理やり復号化を試みる
$ openssl rsautl -decrypt -in encrypted-data.txt -out decrypted-data.txt -inkey public-key.pem -pubin
A private key is needed for this operation
# 秘密鍵じゃないとできないよって怒られる
```

### 秘密鍵で復号化する  

次に先程暗号化したデータ(**encrypted-data.txt**)を送信側(**dirB**)から受信側(**dirA**)に渡し,  
受信側の秘密鍵で復号化してみる.  

```bash
# 送信側(dirB)から暗号データ(encrypted-data.txt)を受信側(dirA)に送る
$ cd ..
$ cp dirB/encrypted-data.txt dirA/

# 秘密鍵で復号化する
$ cd dirA
$ openssl rsautl -decrypt -in encrypted-data.txt -out decrypted-data.txt -inkey private-key.pem
$ ls
decrypted-data.txt  encrypted-data.txt  private-key.pem  public-key.pem

# 元のデータが復号化されている
$ cat decrypted-data.txt
abcdefg
```

以上の手順で暗号化データを平文データに復号化できた.  

重要なのは  
**送信側と受信側の間でやりとりされるデータが受信側の公開鍵と暗号データだけになる** こと.  
**暗号化されたデータは秘密鍵によってのみ復号化できる** ので,  
仮に通信が傍受されても秘密鍵が盗まれない限り情報は守られる.  
これが公開鍵暗号の考え方.  

### パスフレーズをつける

これまで確認したように公開鍵による暗号化で情報は守られるが,  
秘密鍵が流出した場合はそうではなくなってしまう.  

このため, 万が一に備えて秘密鍵自体をさらに暗号化してしまうというのがパスフレーズの考え.  
まずは確認のため, 秘密鍵にパスフレーズを追加してみる.  

```bash
# 受信側で作業
$ cd dirA
$ ls
decrypted-data.txt  encrypted-data.txt  private-key.pem  public-key.pem

# 秘密鍵にパスフレーズをつける(暗号化方式はaes256を使用する)
$ openssl rsa -in private-key.pem -out private-key-with-pass-phrase.pem -aes256
writing RSA key
Enter PEM pass phrase:              # 任意のパスフレーズを入力
Verifying - Enter PEM pass phrase:  # パスフレーズを再入力
$ ls
decrypted-data.txt  encrypted-data.txt  private-key.pem  private-key-with-pass-phrase.pem  public-key.pem
```

パスフレーズつきの秘密鍵を使って何かしようとすると,  
必ずパスフレーズの入力を求められる.  
このため, 万が一第三者に秘密鍵が流出してもパスフレーズがわからない限りは暗号データを復号化できない.  

```bash
# 秘密鍵で暗号データを復号化しようとするとパスフレーズを要求される
$ openssl rsautl -decrypt -in encrypted-data.txt -out decrypted-data.txt -inkey private-key-with-pass-phrase.pem
Enter pass phrase for private-key-with-pass-phrase.pem: # 間違ったパスフレーズを入力
unable to load Private Key
# パスフレーズが違うので秘密鍵が使えない
```

ただパスフレーズがあると面倒なパターンもあるので,  
これを外すこともできる.  

```bash
# パスフレーズを外す
$ openssl rsa -in private-key-with-pass-phrase.pem -out private-key-without-pass-phrase.pem
Enter pass phrase for private-key-with-pass-phrase.pem: # 正しいパスフレーズを入力
writing RSA key

# パスフレーズが要求されなくなる
$ openssl rsautl -decrypt -in encrypted-data.txt -out decrypted-data.txt -inkey private-key-without-pass-phrase.pem
```

## おわり
実際に手を動かして試したおかげでちょっとは理解が深まった気がする.  
公開鍵暗号方式だけではまだセキュリティに問題があるので(公開鍵の信頼性など),  
時間があればハイブリッド暗号方式とかデジタル証明書についても実際に試してみたい.  

## おまけ
はらみせねこ  
![そとちゃん](/images/2020-05-20/sotochan.jpg)

## 参考

- [秘密鍵と公開鍵の性質](#秘密鍵と公開鍵の性質)
    - [アルゴリズム図鑑 絵で見てわかる26のアルゴリズム](https://www.shoeisha.co.jp/book/detail/9784798149776)
        - めっちゃわかりやすい
- [秘密鍵と公開鍵を作る](#秘密鍵と公開鍵を作る)
    - https://www.openssl.org/docs/man1.1.0/man1/genrsa.html
    - https://www.openssl.org/docs/man1.1.0/man1/rsa.html
    - https://wiki.openssl.org/index.php/Command_Line_Utilities#Generating_an_RSA_Private_Key
- [公開鍵で暗号化する](#公開鍵で暗号化する), [秘密鍵で復号化する](#秘密鍵で復号化する)
    - https://www.openssl.org/docs/man1.1.0/man1/rsautl.html
- [パスフレーズをつける](#パスフレーズをつける)
    - https://www.openssl.org/docs/man1.1.0/man1/rsa.html
