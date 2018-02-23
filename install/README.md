# ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

## はじめに

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました。(ceph-deployはdeprecated)  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、コンテナを使わない1node:1daemonのオーソドックスなスタイルとします。(コンテナデプロイのスタイルは別の機会に書きます)

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcs/blob/master/install/image/cluster.png)


## 事前セットアップ

クラスターをデプロイする前に各ノードで事前のセットアップ作業が必要になります。  
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

### 1. mgmtノードにceph-ansibleをインストール
```
[root@mgmt]# subscription-manager register --username=$CDN_USERNAME --password=$CDN_PASSWORD
[root@mgmt]# subscription-manager reflesh
[root@mgmt]# subscription-manager list --available --all --matches="*Ceph*"
[root@mgmt]# subscription-manager attach --pool=$POOL_ID

[root@mgmt]# subscription-manager repos --disable='*' --enable=rhel-7-server-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rhceph-3-tools-rpms
[root@mgmt]# yum update

[root@mgmt]# yum -y install ceph-ansible
```
### 2. ansible hostsファイルを編集
```
[root@mgmt]# vi /etc/ansible/hosts
[mons]
mon01
mon02
mon03
[osds]
osd01
osd02
osd03
[rgws]
rgw
```

### 3. Passwordless sshのためmgmtノードから各ノードにssh keyの配布
```
[root@mgmt]# ssh-keygen -f /root/.ssh/id_rsa -N ‘’
[root@mgmt]# ansible all -m authorized_key -a "user=root key='{{ lookup('file', '/root/.ssh/id_rsa.pub') }}'" --ask-pass
SSH password: $HOST_PASSWORD
mon01 | SUCCESS => {
	"changed": true,
	"exclusive": false,
	"key": "ssh-rsa 
...
[root@mgmt]# ansible all -m ping    ## 確認
```

### 4. RGW/MON/OSDノードのセットアップのためのplaybookを作成して実行
```
[root@mgmt]# vi setup.yml
---
## 全ノード共通の設定
-  hosts: all
   user: root
   tasks:
     - name: register system to RHSM and attach subscription
       redhat_subscription:
         username: $CDN_USERNAME
         password: $CDN_PASSWORD
         pool_id: $POOL_ID
         state=present
     - name: disable all repository
       yum_repository:
         name: '*'
         state: absent
     - name: enable RHEL 7 base repository
       yum_repository:
         name: rhel-7-server-rpms
         description: Red Hat Enterprise Linux 7 Server (RPMs)
         baseurl: https://cdn.redhat.com/content/dist/rhel/server/7/$releasever/$basearch/os
         state: present
     - name: enable RHEL 7 extras repository
       yum_repository:
         name: rhel-7-server-extras-rpms
         description: Red Hat Enterprise Linux 7 Server - Extras (RPMs)
         baseurl: https://cdn.redhat.com/content/dist/rhel/server/7/7Server/$basearch/extras/os
         state: present

## MONの設定
-  hosts: mons
   user: root
   tasks:
     - name: enable RHCS 3 MON repository
       yum_repository:
         name: rhel-7-server-rhceph-3-mon-rpms
         description: Red Hat Ceph Storage MON 3 for Red Hat Enterprise Linux 7 Server (RPMs)
         baseurl: https://cdn.redhat.com/content/dist/rhel/server/7/7Server/$basearch/rhceph-mon/3/os
         state: present
     - name: accept 6789/tcp port on firewall
       firewalld:
         port: 6789/tcp
         zone: public
         permanent: true
         state: enabled
     - name: accept 6800-7300/tcp port on firewall
       firewalld:
         port: 6700-7300/tcp
         zone: public
         permanent: true
         state: enabled

## OSDの設定
-  hosts: osds
   user: root
   tasks:
     - name: enable RHCS 3 OSD repository
       yum_repository:
         name: rhel-7-server-rhceph-3-osd-rpms
         description: Red Hat Ceph Storage OSD 3 for Red Hat Enterprise Linux 7 Server (RPMs)
         baseurl: https://cdn.redhat.com/content/dist/rhel/server/7/7Server/$basearch/rhceph-osd/3/os
         state: present
     - name: accept 6800-7300/tcp port on firewall
       firewalld:
         port: 6700-7300/tcp
         zone: public
         permanent: true
         state: enabled

## RGWの設定
-  hosts: rgws
   user: root
   tasks:
     - name: enable RHCS 3 Tools repository
       yum_repository:
         name: rhel-7-server-rhceph-3-tools-rpms
         description: Red Hat Ceph Storage Tools 3 for Red Hat Enterprise Linux 7 Server (RPMs)
         baseurl: https://cdn.redhat.com/content/dist/rhel/server/7/7Server/$basearch/rhceph-tools/3/os
         state: present
     - name: accept 7480/tcp port on firewall
       firewalld:
         port: 7480/tcp
         zone: public
         permanent: true
         state: enabled

[root@mgmt]# ansible-playbook setup.yml
```



