---
title: "Raspberry PiのIPを固定していろいろできるようにした"
date: 2020-01-07T11:35:15+09:00
draft: false
tags: ["作業ログ", "Raspberry Pi"]
---

## 部屋のラズパイにSSHとかしたい

適当に作ったアプリの動作確認とかでたまに使ってるラズパイが部屋に転がってるんだけど,  
使うたびに画面につないだりするのが面倒なので部屋のMacBookからサクッとSSHとかVNCで作業できるようにした.  

<!--more-->
---

## やったことのまとめ

- ラズパイにSSHする設定を行った
- ラズパイのIPを固定した
- ついでにVNCでGUIを触れるようにした

## つかうもの

- Raspberry Pi 3 Model B+
    - IP固定したいラズパイ
    - ルーターに無線接続している状態
    - OSはRaspbian(10.0)
- E-WMTA2.3
    - うちにあるルーター
    - SoftBank光にすると使わざるを得ないやつ
    - [スペック](https://www.softbank.jp/support/faq/view/24536)
- MacBook
    - ラズパイと同じルーターに無線接続してるコンピュータ
    - 動作確認くらいにしか使わないのでSSHが使えればなんでも良い

## やったこと

- [ラズパイのSSH設定](#ラズパイのssh設定)
- [ラズパイのIPアドレス固定](#ラズパイのipアドレス固定)
- [VNCの設定](#vncの設定)

### ラズパイのSSH設定

まずはラズパイにSSHできるように設定する.  

ラズパイのデスクトップ左上のメニューから,  
`Preferences`->`Raspberry Pi Configuration`->`Interfaces`と進み, `SSH`の`Enable`を選択して`OK`を押す.  
これでSSHの準備は完了.  

![SSH設定](/images/2020-01-07-raspberry-pi-sc01.png)  

試しにラズパイのIPが固定されていない状態でMacBookからSSHしてみる.  

まずはラズパイ側でIPアドレスを確認.  

```bash
# ラズパイで実行
# 無線接続しているのでwlan0のinetを見る
$ ifconfig
eth0: ...

lo: ...

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.3.11  netmask 255.255.255.0  broadcast 192.168.3.255
        ...
```

以上よりラズパイのIP(DHCPで割り振られたもの)は **192.168.3.11** であることがわかったので, MacBookからSSHしてみる.  

```bash
# MacBookで実行
$ hostname
macbook.local

# ラズパイのデフォルトユーザー(pi)で入る
# 先程確認したラズパイのIPを指定
$ ssh pi@192.168.3.11
pi@192.168.3.11's password: # piのパスワードを入力
Linux raspberrypi 4.19.50-v7+ #896 SMP Thu Jun 20 16:11:44 BST 2019 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  7 00:21:34 2020 from 192.168.3.2

[pi@192.168.3.11]$ hostname
raspberrypi
```

やったぜ.  
ラズパイにSSHできた.  

### ラズパイのIPアドレス固定

上記の手順だとラズパイの電源を入れ直したときなどDHCPで新しいIPが振られるたびに確認が必要になって手間になるので,  
次はラズパイのIPをDHCPではなく固定するようにする.  

まずは[こちらの手順](http://ybb.softbank.jp/support/connect/hikari/router/bbu23-menu.php)を参考にブラウザで以下のアドレスからルーター(E-WMTA2.3)の設定画面を開く.  
ラズパイのブラウザは見づらいので自分はMacBookのChromeを使った.  
http://172.16.255.254/  

`ルーター機能の設定`->`IPアドレス／DHCPサーバの設定`と進んで`割当IPアドレスの範囲`を確認する.  

![ルーター設定](/images/2020-01-07-sc01.png)  

この場合は **192.168.3.2** ~ **192.168.3.254** までがDHCPで使われるIPになっている.  
(他のルーターでも同じようにDHCPで使うIPの範囲を確認できるはず)  

おそらく[この手順](https://www.softbank.jp/support/faq/view/19680)を使えばラズパイ側で設定しなくてもラズパイのMACアドレスに対してIPを固定できるんだけど(正確にはDHCPで振られるIPが固定される?),  
今回はラズパイの設定をいじる形でやってみる.  

このままだと他のデバイスが増えた場合にDHCPで使われる範囲が固定したいIPとかぶるかもしれない(ほんまか?)ので, 念の為範囲を **192.168.3.2** ~ **192.168.3.128** に変更しておく.  

![ルーター設定](/images/2020-01-07-sc02.png)  

ルーターを再起動する.  

次にラズパイの設定ファイル`/etc/dhcpcd.conf`を編集し, 任意のIPで固定する.  
今回は **192.168.3.200** で固定する.  

```bash
# 以下は全てラズパイで実行

# デフォルトゲートウェイの確認
$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         192.168.3.1     0.0.0.0         UG    303    0        0 wlan0 # この行の情報を使う
192.168.3.0     0.0.0.0         255.255.255.0   U     303    0        0 wlan0

# 設定ファイルを編集
$ vim /etc/dhcpcd.conf
...
# 以下の記述を末尾に追加
interface wlan0 # routeで確認したIface
static ip_address=192.168.3.200/24 # 任意の固定IP
static routers=192.168.3.1 # Gateway
static domain_name_servers=192.168.3.1 # 特にDNSの設定をしていなければroutersと同じ値で問題ないはず
```

編集後, ラズパイを再起動する...  

```bash
# 再度ラズパイでIPを確認する
$ ifconfig
eth0: ...

lo: ...

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.3.200  netmask 255.255.255.0  broadcast 192.168.3.255
        ...
```

電源何回入れ直しても同じIPになる.  
どうやら指定したIPで固定されたっぽい.  

今度はMacBookから固定IPを指定してSSHする.  

```bash
# MacBookで実行
# 固定IPを指定
$ ssh pi@192.168.3.200
The authenticity of host '192.168.3.200 (192.168.3.200)' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.3.200' (ECDSA) to the list of known hosts.
pi@192.168.3.200's password:
Linux raspberrypi 4.19.50-v7+ #896 SMP Thu Jun 20 16:11:44 BST 2019 armv7l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  7 01:45:23 2020 from 192.168.3.5

[pi@192.168.3.200]$ hostname
raspberrypi
```

できた.  
やったぜ.  

### VNCの設定

SSHできるようになったので, ついでにVNCでGUIをいじれるようにする.  
ラズパイのGUIはあんまり触らなさそうだけど, 念の為.  

といってもRaspbianにはRealVNCが最初から入っているので, 必要な設定はほとんどない.  
SSHを有効にしたときと同様にやればいい.  

ラズパイのデスクトップ左上のメニューから,  
`Preferences`->`Raspberry Pi Configuration`->`Interfaces`と進み, `VNC`の`Enable`を選択して`OK`を押す.  
![VNC設定](/images/2020-01-07-raspberry-pi-sc01.png)  

ラズパイ側の準備は完了.  
MacBookに最初から入ってる画面共有.appで固定IPを指定して接続を試みると失敗する.  

![デフォルトの画面共有だとだめ](/images/2020-01-07-sc03.png)  

対処法がよくわかんないので普通に[VNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)をMacBookにインストールして起動.  
検索窓みたいなやつにラズパイのIPを打ち込んでEnterを押すとなんか色々聞かれるけど問題なし.  

![なんか聞かれる](/images/2020-01-07-sc05.png)  

ラズパイのユーザー名とパスワードを聞かれるので入力.  

![たのむ](/images/2020-01-07-sc06.png)  

ラズパイの画面が開く.  
やったぜ.  

![VNCできた画面](/images/2020-01-07-sc07.png)  

このままだと解像度が低くてストレスがマッハなので,  
ラズパイ側で再度設定を行う.  

ラズパイのデスクトップ左上のメニューから,  
`Preferences`->`Raspberry Pi Configuration`->`System`と進み, `Resolution`を`Default`から好みの解像度(自分の場合は`DMT Mode 82`)に変更する.  
ラズパイの再起動を求められるのでYesを押す.  

![解像度の設定](/images/2020-01-07-sc08.png)  

少し時間がかかるが, ラズパイが再起動すると自動的にVNC Viewerの画面も再接続され,  
解像度が上がった画面が表示される.  

![解像度が上がったラズパイ画面](/images/2020-01-07-sc09.png)  

やったぜ.  
これでGUIも使えるようになった(たぶんあんまり使わない).  

## おわり
以上の手順で家の中のパソコンからいつでもラズパイに接続できるようになった.  
ラズパイに毎回ディスプレイとキーボード繋ぐのめんどくさかったし, もっと早くやればよかった...  
ルーターでポート周りの設定をすれば家の外からも接続できるっぽいんだけど,  
自宅サーバーの運用とかよくわかってないしちょっとセキュリティ的にも怖いので当分の間はこれで遊ぶつもり.  

## おまけ
ねずみ年そとちゃん
![ねずみ年そとちゃん](/images/2020-01-07-sotochan.jpg)  
