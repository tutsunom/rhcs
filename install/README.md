# ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

## はじめに

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました。(ceph-deployはdeprecated)  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、コンテナを使わない1node:1daemonのオーソドックスなスタイルとします。(コンテナデプロイのスタイルは別の機会に書きます)

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcs/blob/master/install/image/cluster.png)


## 事前準備

クラスターをデプロイする前に各ノードで事前準備が必要になります。  
これも負荷がバカにならない作業なので、mgmtノードからansibleを使ってこれらの設定も自動で行います。



- 全ノードのSSH構成
- 名前解決
- 時刻同期
- ファイアウォールのポート開放
  - MON : tcp/6789,6800-7300
  - OSD : tcp/6800-7300
  - RGW : tcp/7480
- RHCSサブスクリプションをアタッチ
- yumリポジトリの有効化
  - Mgmt : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-tools-rpms
  - MON : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-mon-rpms
  - OSD : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-osd-rpms
  - RGW : rhel-7-server-rpms, rhel-7-server-extras-rpms, rhel-7-server-rhceph-3-tools-rpms
- 時刻同期(ntp)  



```
[root@mgmt]# nmcli connection modify eth1 ipv4.dns xxx.xxx.xxx.xxx
(ans-inst)# nmcli connection modify eth1 ipv4.dns xxx.xxx.xxx.xxx
(ans-inst)# systemctl restart NetworkManager.service

## [c]サブスクリプションのアタッチ
(ans-inst)# subscription-manager register --username=[USERNAME] --password=[PASSWORD]
(ans-inst)# subscription-manager list --available　　## RHELとRHCSのPool IDを調べてメモ
(ans-inst)# subscription-manager attach --pool=[POOL_ID]

## [d]yumリポジトリの有効化
(ans-inst)# subscription-manager repos --disable='*' \ --enable=rhel-7-server-rpms --enable=rhel-7-server-rhscon-2-installer-rpms

## [e]時刻同期 : ntpdのインストールと設定
(ans-inst)# yum -y install ntp
(ans-inst)# vi /etc/ntp.conf
+restrict xxx.xxx.xxx.0 mask 255.255.255.0 nomodify notrap　　## 時刻同期の有効範囲を設定
+server xxx.xxx.xxx.xxx iburst　　## NTPサーバを設定
(ans-inst)# systemctl restart ntpd.service; systemctl enable ntpd.service
(ans-inst)# ntpq -p　　## 同期処理


## ceph-ansibleのインストール
(ans-inst)# yum -y install ceph-ansible

```

