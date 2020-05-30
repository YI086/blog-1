---
title: "OpenSSLでデジタル証明書を試す"
date: 2020-05-30T19:58:15+09:00
draft: false
tags: ["メモ", "セキュリティ", "OpenSSL"]
---

## 身元を証明する
デジタル署名とデジタル証明書についても実際に触りながらまとめる.  

<!--more-->
---

## まとめ

**デジタル署名**
```bash
# メッセージ(message)のハッシュ値を秘密鍵(private-key.pem)で暗号化して署名(signature)を作成
$ openssl dgst -sha256 -sign private-key.pem -out signature message

# 署名(signature)を公開鍵(public-key.pem)で復号化してメッセージ(message)のハッシュ値と照合
$ openssl dgst -sha256 -verify public-key.pem -signature signature message
```

**デジタル証明書**
```bash
# 秘密鍵(private-key.pem)から証明書署名要求(req.pem)を作成
# 証明書署名要求には公開鍵と, 自身の身元を証明する情報が含まれている
$ openssl req -new -key private-key.pem -out req.pem

# 証明書署名要求(req.pem)の内容確認
$ openssl req -in req.pem -text -verify -noout

# 証明書署名要求(req.pem)に認証局の証明書(ca.pem)と秘密鍵(private-key-CA.pem)で署名して証明書を作成(cert.pem)
# 証明書には証明書署名要求の公開鍵の内容と認証局の署名が含まれている
$ openssl ca -policy policy_anything -in req.pem -out cert.pem -keyfile private-key-CA.pem -cert ca.pem

# 証明書(cert.pem)の内容確認
$ openssl x509 -in cert.pem -noout -text

# 証明書(cert.pem)を信頼できる認証局の証明書(ca.pem)で検証する
$ openssl verify -CAfile ca.pem cert.pem

# 証明書(cert.pem)に含まれる公開鍵の確認
$ openssl x509 -in cert.pem -pubkey -noout
```

## 環境
- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
- openssl
    - OpenSSL 1.1.1c  28 May 2019
    - ラズパイの初期装備

## もくじ

- [デジタル署名](#デジタル署名)
- [デジタル証明書](#デジタル証明書)

### デジタル署名

デジタル証明書の前に, デジタル署名について確認する.  

デジタル署名とは次のような流れでデータの送信者が秘密鍵を持つ本人であることとそれが途中で改ざんされていないことを証明する仕組み.  

1. 送信側で秘密鍵と公開鍵を作成し, 受信側に公開鍵を渡す
1. 送信側でメッセージを作成し, それをハッシュ化する
1. 送信側で得られたハッシュ値を秘密鍵で暗号化(署名)する
1. 送信側はメッセージ, 署名, 公開鍵を受信側に渡す
1. 受信側は受け取った署名を公開鍵で復号化し, 受け取ったメッセージをハッシュ化したものと照合する

また, これを実現するためデジタル署名には次の特徴がある.  

- 秘密鍵でしか暗号化できない
- 公開鍵で復号化すると元のメッセージに戻る

実際にやってみる.  

まずは送信側で秘密鍵と公開鍵を作成し, メッセージを暗号化(署名)する.  

```bash
# 2つのディレクトリを送信側(dirX), 受信側(dirY)に見立てて作業
$ mkdir dirX dirY

# 送信側(dirX)で秘密鍵(private-key.pem)と公開鍵(public-key.pem)を作成して公開鍵を受信側(dirY)に渡す
$ cd dirX
$ openssl genrsa -out private-key.pem
$ openssl rsa -in private-key.pem -pubout -out public-key.pem
$ cp ./public-key.pem ../dirY
```

次にメッセージを作成し, ハッシュ化する.  

```bash
# メッセージ(message)を作成してSHA-256でハッシュ化(hashed-message)
$ echo qwerty > message
$ openssl sha256 -out hashed-message message
$ cat hashed-message
SHA256(message)= 9ceece10cf8b97d1f1924dae5d14c137fd144ce999ede85f48be6d7582e2dd23
```

ハッシュ値を秘密鍵で暗号化(署名)する.  

```bash
# ハッシュ値(hashed_message)を秘密鍵(private-key.pem)で暗号化して署名(signature)を作成
$ openssl rsautl -sign -in hashed-message -out signature -inkey private-key.pem
$ cat signature
����������������
```

メッセージ, 署名を受信側に渡す.  

```bash
# メッセージ(message), 署名(signature)を受信側(dirY)に渡す
$ cp ./message ../dirY
$ cp ./signature ../dirY/
$ cd ../dirY
$ ls
message  public-key.pem  signature
```

受信側で署名を復号化して, メッセージをハッシュ化したものと照合する.  
一致すればメッセージが改ざんされておらず, 送り主が秘密鍵の持ち主であることを証明できる.  

```bash
# 署名(signature)を公開鍵(public-key.pem)で復号化(signature-verify)
$ openssl rsautl -verify -in signature -out signature-verify -inkey public-key.pem -pubin

# メッセージ(message)をハッシュ化(hashed-message-verify)
$ openssl sha256 -out hashed-message-verify message

# 復号化した署名(signature-verify)とメッセージのハッシュ値(hashed-message-verify)を照合
$ cat signature-verify
SHA256(message)= 9ceece10cf8b97d1f1924dae5d14c137fd144ce999ede85f48be6d7582e2dd23
$ cat hashed-message-verify
SHA256(message)= 9ceece10cf8b97d1f1924dae5d14c137fd144ce999ede85f48be6d7582e2dd23
$ diff signature-verify hashed-message-verify
```

メッセージのハッシュと署名を復号化した値が一致する組み合わせを作れるのは公開鍵に対応する秘密鍵を持つ者だけなので,  
仮にメッセージが改ざんされたり, 同じメッセージが別の秘密鍵で署名された場合は  
受信側で照合したときに一致せず, 異常を検知することができる.  

以上の手順を踏むことで, 通信時のなりすましや改ざん, 事後否認を防ぐことができる.  

また, ハッシュ計算と署名を1つのコマンドで実行することもできる.  

```bash
# メッセージ(message)のハッシュ値を秘密鍵(private-key.pem)で暗号化して署名(signature)を作成
$ openssl dgst -sha256 -sign private-key.pem -out signature message

# 署名(signature)を公開鍵(public-key.pem)で復号化してメッセージ(message)のハッシュ値と照合
$ openssl dgst -sha256 -verify public-key.pem -signature signature message
```

### デジタル証明書

盗聴を防ぐための方法として用いる[公開鍵暗号方式](https://uzimihsr.github.io/post/2020-05-20-public-key-practice/)や[ハイブリッド暗号方式](https://uzimihsr.github.io/post/2020-05-24-hybrid-encryption-practice/),  
そしてなりすまし, 改ざん, 事後否認を防ぐために用いる[デジタル署名](#デジタル署名)だが,  
これらはすべて受信側が受け取った公開鍵を信頼できるという条件の上で成り立っている.  

仮に公開鍵を受け取る際にそれが悪意のある第三者によってすり替えられていた場合,  
これらの仕組みが全く意味を持たなくなってしまう(公開鍵の信頼性の問題).  

この問題を解決し, 公開鍵の正当性を証明するのがデジタル証明書.  
以下の流れで証明書の作成と検証を行う.  

1. 自分の身分を証明したい人(A)が秘密鍵と公開鍵を用意する.
1. Aは自分の公開鍵と自分の身元を証明できる情報を用意(証明書署名要求)して認証局(CA)に送る.
1. CAはAの証明書署名要求からAの身元を確認した後それらの情報にCA自身の秘密鍵で署名して証明書を作成し, Aに渡す.
1. Aは通信したい人(B)に自分の証明書を渡す.
1. BはCA自体の証明書(公開鍵)でAの証明書を検証する.

また, この仕組みを実現するために証明書には次の性質がある.  

- Aの証明書署名要求はA本人しか作れない
- 証明書には身元が証明される人の公開鍵情報とそれに署名した認証局の情報が含まれる
- 信頼している認証局が署名した証明書であれば信頼性が保証される
- 認証局自身の信頼性はさらに上位の認証局が発行する証明書で保証する

実際にやってみて確認する.  
まずはAとCAの秘密鍵, 公開鍵を用意する.  

```bash
# 3つのディレクトリを通信者(dirA, dirB), 認証局(dirCA)に見立てて作業する
$ mkdir dirA dirB dirCA

# CAとAはそれぞれ秘密鍵を持っている状態
$ cd dirCA
$ openssl genrsa -out private-key-CA.pem
$ openssl rsa -in private-key-CA.pem -pubout -out public-key-CA.pem
$ cd ../dirA
$ openssl genrsa -out private-key-A.pem
$ openssl rsa -in private-key-A.pem -pubout -out public-key-A.pem
```

次にAは自分の公開鍵と身元を証明する情報を合わせたデータ(証明書署名要求)を作成し,  
CAに渡す.  

```bash
# Aの秘密鍵(private-key-A.pem)から証明書署名要求(req.pem)を作成
# 実際は秘密鍵から公開鍵を取り出して使っている(秘密鍵そのものを渡すわけではない)
$ openssl req -new -key private-key-A.pem -out req.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
----- # 以下, 身元を証明するための情報を入力する
Country Name (2 letter code) [AU]:JP #国
State or Province Name (full name) [Some-State]:Tokyo # 都道府県
Locality Name (eg, city) []: # 市区町村
Organization Name (eg, company) [Internet Widgits Pty Ltd]: # 組織名
Organizational Unit Name (eg, section) []: # 部署名
Common Name (e.g. server FQDN or YOUR name) []:uzimihsr.example.com # 名前
Email Address []:example@mail.com # メールアドレス

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
$ cat req.pem
-----BEGIN CERTIFICATE REQUEST-----
MIICxjCCAa4CAQAwgYAxCzAJBgNVBAYTAkpQMQ4wDAYDVQQIDAVUb2t5bzEhMB8G
...
cH32EBVTnecz4nSPVYHwr9xcgKi+i0ol4ea0lMNWz5m0O0dYOc6H0Qqc
-----END CERTIFICATE REQUEST-----

# 証明書署名要求(req.pem)をCAに渡す
$ cp req.pem ../dirCA/
```

CAは受け取ったAの証明書署名要求の内容を確認する.  

```bash
# 証明書署名要求(req.pem)の内容を検証する
$ cd ../dirCA
$ openssl req -in req.pem -text -verify -noout
verify OK
Certificate Request:
    Data: # Aの公開鍵と身元の情報
        Version: 1 (0x0)
        Subject: C = JP, ST = Tokyo, O = Internet Widgits Pty Ltd, CN = uzimihsr.example.com, emailAddress = example@mail.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:b7:51:7c:41:9c:c9:f2:4d:c0:a8:43:bb:96:2a:
                    ...
                    0c:d7
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    Signature Algorithm: sha256WithRSAEncryption
         26:88:97:d7:c4:57:da:24:1b:d5:b4:e2:e8:82:23:b5:1c:e0:
         ...
         87:d1:0a:9c
```

証明書署名要求の内容に問題がなければ署名してAの証明書を作る...前に認証局もそれ自身の身元を証明する必要があるので,  
自己証明証明書(オレオレ証明書)を作ってCAを認証局として動かすための設定をする.  

```bash
# CA自身の身元を証明し鍵(private-key-CA.pem)の正当性を保証するためのオレオレ証明書(ca.pem)を作成
# こちらも内部で秘密鍵から公開鍵を取り出して使用している(証明書に秘密鍵を埋め込むわけではない)
$ openssl req -x509 -key private-key-CA.pem -out ca.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
----- # その場しのぎの証明書なので何も入れなくてもOK
Country Name (2 letter code) [AU]:
State or Province Name (full name) [Some-State]:
Locality Name (eg, city) []:
Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:
Email Address []:

# 証明書に署名するために必要な設定
$ mkdir -p demoCA/newcerts
$ touch demoCA/index.txt
$ echo 01 > demoCA/serial
```

CAはAの証明書署名要求に署名してAの証明書を作成し, Aに渡す.  
これによりAの身元と公開鍵の正当性をCAが保証することになる.  

```bash
# Aの証明書署名要求(req.pem)にCA自身の秘密鍵(private-key-CA.pem)と証明書(ca.pem)を使って署名する(cert-A.pem)
$ openssl ca -policy policy_anything -in req.pem -out cert-A.pem -keyfile private-key-CA.pem -cert ca.pem
Using configuration from /usr/lib/ssl/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: May 30 09:15:17 2020 GMT
            Not After : May 30 09:15:17 2021 GMT
        Subject:
            countryName               = JP
            stateOrProvinceName       = Tokyo
            organizationName          = Internet Widgits Pty Ltd
            commonName                = uzimihsr.example.com
            emailAddress              = example@mail.com
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Comment:
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier:
                F2:CC:D3:94:F9:57:AE:F3:4E:A9:12:1F:15:29:87:8C:58:4B:6C:56
            X509v3 Authority Key Identifier:
                keyid:57:A6:BC:47:C9:76:6C:E3:93:48:D7:09:9E:03:8A:86:3F:A5:08:80

...
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

# 証明書をAに渡す
$ cp cert-A.pem ../dirA/
```

Aは通信を行いたい相手(B)に自分の証明書を渡す.  

```bash
# Bに証明書を渡す
$ cd ../dirA
$ cp cert-A.pem ../dirB
```

BはAの証明書の内容を確認する.  
BはCAを信頼しているので,  
Aの証明書にCAが署名していることを確認できればこれを信頼し, 証明書から取り出したAの公開鍵を安心して利用することができる.  

```bash
# Aの証明書(cert-A.pem)の内容を確認する
$ cd ../dirB
$ openssl x509 -in cert-A.pem -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = AU, ST = Some-State, O = Internet Widgits Pty Ltd # 認証局(CA)の情報
        Validity
            Not Before: May 30 09:15:17 2020 GMT
            Not After : May 30 09:15:17 2021 GMT
        Subject: C = JP, ST = Tokyo, O = Internet Widgits Pty Ltd, CN = uzimihsr.example.com, emailAddress = example@mail.com # Aの身元情報
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus: # Aの公開鍵情報
                    00:b7:51:7c:41:9c:c9:f2:4d:c0:a8:43:bb:96:2a:
                    ...
                    0c:d7
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Comment:
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier:
                F2:CC:D3:94:F9:57:AE:F3:4E:A9:12:1F:15:29:87:8C:58:4B:6C:56
            X509v3 Authority Key Identifier:
                keyid:57:A6:BC:47:C9:76:6C:E3:93:48:D7:09:9E:03:8A:86:3F:A5:08:80

    Signature Algorithm: sha256WithRSAEncryption
         39:a3:7f:a3:dd:9c:ff:21:b7:b3:1b:44:07:c1:38:e0:ae:7a:
         ...
         bc:d0:36:70

# 信頼している認証局(CA)の証明書(ca.pem)を使ってAの証明書(cert-A.pem)が正しいか検証する
$ openssl verify -CAfile ../dirCA/ca.pem cert-A.pem
cert-A.pem: OK

# 検証が成功したので証明書からAの公開鍵を取り出す
$ openssl x509 -in cert-A.pem -pubkey -noout
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAt1F8QZzJ8k3AqEO7liqz
...
1wIDAQAB
-----END PUBLIC KEY-----

# 以降はAの公開鍵を信頼して通信する
```

Bが信頼できる証明書はBが信頼している認証局とA本人によってのみ作成できるので,  
悪意のある第三者がAになりすましてBに自身の公開鍵を渡すことはできない.  
したがって, 証明書を使うことで公開鍵の正当性を保証することができる.  

以上の手順で公開鍵の信頼性を保証することができた.  
やることが多くて混乱するけど,  
要は信頼できる認証局が相手の身元を保証していれば自分も相手を信頼する, という仕組み.  

今回は認証局(CA)の身元を自身で保証したけど(オレオレ証明書),  
実際は上位の認証局がその正当性を保証する.  
その上位の認証局の正当性はさらに上位の認証局が保証し... というように,  
認証局の保証は木構造になっている.  
最上位の認証局の正当性は自身の証明書では保証できないので,  
公的に信頼できる政府関連の機関などが最上位の認証局を努めている.  
以上が公開鍵基盤(PKI)の仕組み.  

## おわり
[公開鍵暗号方式](https://uzimihsr.github.io/post/2020-05-20-public-key-practice/), [ハイブリッド暗号方式](https://uzimihsr.github.io/post/2020-05-24-hybrid-encryption-practice/)に続いてデジタル証明書を実際に試してみた.  
証明書署名要求から証明書の作成までの手順も確認できたので,  
機会があればオレオレ証明書でSSL通信したり, クライアント認証とかを試してみたい.  

## おまけ
ちょっと怒ってるねこ  
![そとちゃん](/images/2020-05-30/sotochan.jpg)

## 参考
- [デジタル署名](#デジタル署名)
    - https://www.openssl.org/docs/man1.1.0/man1/dgst.html
    - https://www.openssl.org/docs/man1.1.0/man1/rsautl.html
- [デジタル証明書](#デジタル証明書)
    - https://www.openssl.org/docs/man1.1.0/man1/req.html
    - https://www.openssl.org/docs/man1.1.0/man1/ca.html
    - https://www.openssl.org/docs/man1.1.0/man1/x509.html
    - https://www.openssl.org/docs/man1.1.0/man1/verify.html
