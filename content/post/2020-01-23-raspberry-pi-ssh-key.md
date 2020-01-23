---
title: "ラズパイに公開鍵でSSHするようにした"
date: 2020-01-23T22:49:13+09:00
draft: false
tags: ["作業ログ", "Raspberry Pi"]
---

## パスワード打つのめんどい
家にあるラズパイにSSHできるようにしたはいいものの, 毎回パスワード入れるのも面倒だし真面目に考えたらセキュリティ的にもよろしくないので公開鍵認証で入るようにした.  

<!--more-->
---

## やったことのまとめ

- MacBookの公開鍵をラズパイ側に登録
- ラズパイ側でパスワードログインを無効化

ラズパイのIPは[固定済み](https://uzimihsr.github.io/post/2020-01-07-raspberry-pi-static-id/#%E3%83%A9%E3%82%BA%E3%83%91%E3%82%A4%E3%81%AEip%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9%E5%9B%BA%E5%AE%9A)  

## つかうもの

- Raspberry Pi 3 Model B+
    - SSHサーバー(入られる側)
    - OSはRaspbian(10.0)
- MacBook
    - SSHクライアント(入る側)
    - opensshがあれば正直なんでもいい

## やったこと

### 公開鍵をラズパイ側に登録

まずはクライアント(MacBook)側で秘密鍵と公開鍵のペアを作成する.  
すでに`~/.ssh/id_rsa`(秘密鍵)と`~/.ssh/id_rsa.pub`(公開鍵)がある場合は飛ばしていい.  

```bash
# MacBookで実行

# 秘密鍵と公開鍵のペアを作成する
$ ssh-keygen
$ ls ~/.ssh
id_rsa         id_rsa.pub     known_hosts
```

次にMacBookの公開鍵をラズパイ側に登録する.  

```bash
# 全てMacBookで実行

# ラズパイに公開鍵を保存するためのファイル(~/.ssh/authorized_keys)を作成
# ここではまだパスワード認証
$ ssh pi@192.168.3.200 "touch ~/.ssh/authorized_keys; ls ~/.ssh"
authorized_keys # このファイルにMacBookの公開鍵を書き込む
id_rsa          # これはラズパイの秘密鍵なので関係ない
id_rsa.pub      # 同じく関係なし
known_hosts     # 同じく関係なし

# ラズパイのauthorized_keysにMacBookの公開鍵を書き込んで権限を変更
# まだパスワード認証
$ cat ~/.ssh/id_rsa.pub | ssh pi@192.168.3.200 "cat >> ~/.ssh/authorized_keys; chmod 700 ~/.ssh/authorized_keys"

# 念の為鍵が登録されたか確認
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAB3...wlj uzimihsr@macbook.local
# MacBookの公開鍵が登録されているのでパスワードを聞かれなくなっている
$ ssh pi@192.168.3.200 "cat ~/.ssh/authorized_keys"
ssh-rsa AAAB3...wlj uzimihsr@macbook.local
```

やったぜ.  
これでパスワードなしでもSSHできるようになった.  

### パスワード認証を無効化

このままでもいいんだけど, セキュリティ的にパスワード認証はあまりよろしくない(総当りされたらクソ雑魚ナメクジ)ので,  
パスワード認証を無効化する.  
ラズパイの`/etc/ssh/sshd_config`を編集してパスワード認証の可否を変更する.  

```bash
# 全てラズパイで実行

# SSHサーバー側の設定ファイルを編集
$ sudo vim /etc/ssh/sshd_config
...
# To disable tunneled clear text passwords, change to no here!
PasswordAuthentication no # ここをyesからnoに変更
#PermitEmptyPasswords no
...

# 念の為sshd.serviceを再起動
$ sudo systemctl restart ssh.service
```

ちゃんとパスワード認証が無効化されているか確認する.  
公開鍵認証ではクライアント(MacBook)の秘密鍵とサーバー(ラズパイ)に登録された公開鍵が対応していないと認証が通らないので,  
MacBook側の秘密鍵を使えなくするとログインできなくなる.  

```bash
# 全てMacBookで実行

# 秘密鍵の名前を変えて読めない状態にする
$ mv ~/.ssh/id_rsa ~/.ssh/id_rsa.bkup

# ラズパイにSSHを試みる
# MacBookの秘密鍵が使えず, パスワード認証が禁止されているので弾かれる
$ ssh pi@192.168.3.200
pi@192.168.3.200: Permission denied (publickey).

# 秘密鍵の名前をもとに戻して使えるようにする
$ mv ~/.ssh/id_rsa.bkup ~/.ssh/id_rsa

# 再びラズパイにSSHを試みる
# 今度は秘密鍵が使えるので問題なくログインできる
$ ssh pi@192.168.3.200
...
Last login: Thu Jan 23 23:24:24 2020 from ...
```

やったぜ.  
これで鍵認証でしかログインできなくなった.  

## おわり
ラズパイでの作業がだいぶ楽になった.  

## おまけ
パソコンばっかりさわってるしもべに愛想をつかすそとちゃん  
![そとちゃん](/images/2020-01-23-sotochan.jpg)  
