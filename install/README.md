## ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました。(ceph-deployはdeprecated)  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、コンテナを使わない1node:1daemonのオーソドックスなスタイルとします。(コンテナデプロイのスタイルは別の機会に書きます)

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcs/blob/master/install/image/cluster.png)


### 前提

- 全ノードのSSH構成  
- 名前解決  
- 時刻同期
- ファイアウォールのポート開放
  - MON : aaa
  - OSD : bbb
- RHCSサブスクリプションをアタッチ  
- yumリポジトリの有効化  
  - MGMT : rhel-7-server-rpms, rhel-7-server-rhscon-2-installer-rpms  
  - MON : rhel-7-server-rpms, rhel-7-server-rhceph-2-mon-rpms  
  - OSD : rhel-7-server-rpms, rhel-7-server-rhceph-2-osd-rpms  
- 時刻同期(ntp)  
