---
title: "Grafanaをログインなしで見られるようにする"
date: 2020-05-08T20:06:38+09:00
draft: false
tags: ["作業ログ", "Grafana"]
---

## ログインが面倒
Grafanaのダッシュボードを他の人に見せるときにログインしてもらうのが面倒だと思った.  

<!--more-->
---

## やったことのまとめ
- `Snapshot`を使ってダッシュボードを共有した
- ダッシュボードをログインなしで閲覧できるよう設定した
- ログインなしで見られるダッシュボードを`Organization`で分けた

## つかうもの

- Raspberry Pi 3 Model B+
    - OSはRaspbian(10.0)
    - 監視サーバとして使用
- Grafana
    - v6.5.2
    - [インストール済み](https://uzimihsr.github.io/post/2020-01-15-prometheus-grafana-raspberry-pi/)

## やったこと

- [Snapshotを使う](#snapshotを使う)
- [anonymous accessを有効にする](#anonymous-accessを有効にする)
- [Organizationを分ける](#organizationを分ける)

### Snapshotを使う

初期設定の`Grafana`を開くとIDとパスワードでのログインを求められる.  

![Grafana](/images/2020-05-08/sc01.png)  

自分が管理者であれば普通にログインするだけなんだけど,  
他の人にダッシュボードを共有するときにいちいち新規ユーザーを作ってパスワードを発行するのはめんどくさい.  

こんなときに`Snapshot`を使うとダッシュボードの状態を保存して共有できる.  

共有したいダッシュボード右上の`Share`ボタンから`Snapshot`を選択する.  

`Snapshot name`(共有する`Snapshot`の名前)と`Expire`(`Snapshot`の有効期限),  
`Timeout`(メトリクス取得のタイムアウト秒数)が設定できるけど,  
特に何も変えずに`Local Snapshot`を選択する.  

![Grafana](/images/2020-05-08/sc02.png)  

`Snapshot`のURLが生成される.  
ただしこの状態ではホストが **localhost** になっているので注意.  

![Grafana](/images/2020-05-08/sc03.png)  

`Snapshot`の一覧は画面左のメニューから`Dashboards`->`Snapshots`で確認できる.  
ここで表示されるURLを使うのがかんたん.  

![Grafana](/images/2020-05-08/sc04.png)  

ログインしてたときのキャッシュを使わないようにシークレットブラウザで`Snapshot`のURLを開いてみると,  
今度はログインなしでダッシュボードが開ける.  

![Grafana](/images/2020-05-08/sc05.png)  

ただし`Snapshot`は作成時に時間の範囲が固定されてしまうので,  
画像のように範囲を変えるとメトリクスを見ることができない...  

![Grafana](/images/2020-05-08/sc06.png)  

### anonymous accessを有効にする

`Snapshot`は簡単に共有できるので便利だけど,  
やっぱり普通のダッシュボードが見たいのでそもそものログイン設定をいじってみる.  

`Grafana`の設定ファイル(`grafana.ini`)を編集する.  

```bash
# Grafana設定ファイルを編集して再起動
$ sudo vim /etc/grafana/grafana.ini
$ sudo systemctl restart grafana-server
```

設定はこんな感じ.  
だいたい **290** 行目あたりに`Anonymous Auth`の設定があるので変更する.  
<details><summary>`grafana.ini`(抜粋)</summary><div>
```ini
#################################### Anonymous Auth ######################
[auth.anonymous]
# enable anonymous access
# ゲストユーザー(ログインなし)のアクセスを許可する
enabled = true

# specify organization name that should be used for unauthenticated users
# Organization名を指定する(Main Org.はデフォルトで作成されている)
org_name = Main Org.

# specify role for unauthenticated users
# ゲストユーザーのRole(役割)をViewer(閲覧のみ)にする
# Editor(編集者), Admin(管理者)も設定可能
org_role = Viewer
```
</div></details>

再起動した後, [Snapshotを使う](#snapshotを使う)と同様にシークレットブラウザで`Grafana`のURLを開くと
今度はログインなしでダッシュボードが開かれる.  

こちらは元のダッシュボードそのものなので時間の範囲を変更してもメトリクスは問題なく表示できるが,  
`Role`が`Viewer`なので編集ができなくなっていて, `Data Source`や`Query`も見えなくなっている.  

![Grafana](/images/2020-05-08/sc07.png)  

編集のためにログインしたい場合は左下の`Sign in`からログイン画面が開くので,  
そこからログインすれば今まで通り編集ができるようになる.  

![Grafana](/images/2020-05-08/sc08.png)  

![Grafana](/images/2020-05-08/sc09.png)  

これで閲覧のみの場合はログインなしで`Grafana`に入れるようになった.  
やったぜ.  

### Organizationを分ける

以上の手順で`anonymous access`を許可すると,  
指定した`Organization`のダッシュボードがすべてログインなしで閲覧可能になってしまう.  

そのため, 作成途中のダッシュボードやあまり他の人に知られたくないメトリクスの場合は`Organization`を分けて管理することにする.  

`Organization`の作成はUIから簡単にできる.  
`Admin`権限でログインした状態で画面左のメニューから`Server Admin`->`Orgs`->`New Org`と進み,  
`Org. name`に任意の`Organization`名を入力して`Create`するだけ.  

![Grafana](/images/2020-05-08/sc10.png)  

または, `Grafana`のAPIを使って作成することもできる.  
初期設定の場合はbasic認証がかけられているので`Admin`権限ユーザーのIDとパスワードで突破する.  

```bash
$ curl -X POST http://<GrafanaのURL>/api/orgs \
    -u <Admin権限のuser>:<password> \
    -H "Accept: application/json" \
    -H "Content-Type: application/json" \
    -d '{"name":"Org2"}'
{"message":"Organization created","orgId":3}
```

![Grafana](/images/2020-05-08/sc11.png)  

新しく作った`Organization`(**Org1**)でダッシュボードを作成して保存する.  

![Grafana](/images/2020-05-08/sc12.png)  

再度`Grafana`の設定を変更する.  

```bash
# Grafana設定ファイルを編集して再起動
$ sudo vim /etc/grafana/grafana.ini
$ sudo systemctl restart grafana-server
```

<details><summary>`grafana.ini`(抜粋)</summary><div>
```ini
#################################### Anonymous Auth ######################
[auth.anonymous]
# enable anonymous access
enabled = true

# specify organization name that should be used for unauthenticated users
# 新しく作ったOrganizationを指定
org_name = Org1

# specify role for unauthenticated users
org_role = Viewer
```
</div></details>

これまでと同様にシークレットブラウザで`Grafana`を開くと同様にログインなしで入れるが,  
今度は **Org1** のダッシュボードしか見られないようになっている.  

![Grafana](/images/2020-05-08/sc13.png)  

元々の`Organization`(**Main Org.**)を開きたい場合はログインした状態で  
左下のユーザーアイコンから`Switch Organization`->`Switch to`で切り替えられる.  

![Grafana](/images/2020-05-08/sc14.png)  

これでログインなしで見られるダッシュボードを管理することができた.  
やったぜ.  

## おわり
これで`Grafana`のダッシュボードをログインなしで見られるようになった.  
設定自体は3行だけでできてめっちゃ簡単だったし,  
ダッシュボードが完成した後はあまりいじることもないので基本はこの設定にしておくのが便利だと思った.  

## おまけ
アンモニャイト  
![そとちゃん](/images/2020-05-08/sotochan.jpg)  

## 参考

- [Snapshotを使う](#snapshotを使う)
    - https://grafana.com/docs/grafana/latest/reference/share_dashboard/
- [anonymous accessを有効にする](#anonymous-accessを有効にする)
    - https://grafana.com/docs/grafana/latest/auth/overview/#anonymous-authentication
- [Organizationを分ける](#organizationを分ける)
    - https://grafana.com/docs/grafana/latest/http_api/org/#create-organization
