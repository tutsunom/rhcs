# ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

## はじめに

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました(ceph-deployはdeprecated)。  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、コンテナを使わない1node:1daemonのオーソドックスなスタイルとします。(コンテナデプロイのスタイルは別の機会に書きます)

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcs/blob/master/install/image/cluster.png)

---

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
[mgrs]
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

failed無しでplaybookが実行されたら準備完了です。

---

## ceph-ansibleの変数用YAMLファイルの作成

ceph-ansibleで必要となる変数はたくさんあります。これらはYAMLファイルに記述され、これがcephクラスターの基本構成を決めます。  
ここではクラスターを構成する必要最小限の項目だけ取り上げて作成しますが、他の項目も興味がある方は後述される各YAMLのサンプルファイルに説明がありますのでご覧下さい。

### 1. keyring用のディレクトリを作成
```
[root@mgmt]# mkdir ~/ceph-ansible-keys
[root@mgmt]# ln -s /usr/share/ceph-ansible/group_vars /etc/ansible/group_vars
```

### 2. all.ymlの作成
all.ymlはクラスター全体に共通する項目が格納されるYAMLファイルです。  
サンプルの`/usr/share/ceph-ansible/group_vars/all.yml.sample`をコピーしてall.ymlを作ります。適宜コメントマーカーを外して編集します。
```
[root@mgmt]# cd /usr/share/ceph-ansible/
[root@mgmt]# cp group_vars/all.yml.sample group_vars/all.yml
[root@mgmt]# vi all.yml		#下記のような項目を設定する
[root@mgmt]# grep -v -e '^\s*#' -e '^\s*$' all.yml
---
dummy:
fetch_directory: ~/ceph-ansible-keys
ceph_origin: repository
ceph_repository: rhcs
ceph_repository_type: cdn
ceph_rhcs_version: 3
monitor_interface: eth0
public_network: "10.0.10.0/16"
cluster_network: "192.168.1.0/24"
radosgw_interface: eth0
```

### 3. osds.ymlの作成
`osds.yml`はOSDの項目が格納されるYAMLファイルです。OSDはdataとjournalの配置など設計によっていろいろなパターンの構成を取ることができますが、今回はシンプルにuserdataとjournalを全OSDに分散するパターンで作ります。  
`/usr/share/ceph-ansible/group_vars/osds.yml.sample`をコピーして`osds.yml`を作って編集します。
```
[root@mgmt]# cd /usr/share/ceph-ansible/
[root@mgmt]# cp group_vars/osds.yml.sample group_vars/osds.yml
[root@mgmt]# vi osds.yml	#下記のような項目を設定する
[root@mgmt]# grep -v -e '^\s*#' -e '^\s*$' osds.yml
---
dummy:
osd_scenario: collocated
devices:
  - /dev/sdb
  - /dev/sdc
  - /dev/sdd
```

### 4. playbookの実行
ceph-ansibleのplaybookは`/usr/share/ceph-ansible/site.yml.sample`をコピーして`site.yml`として作ります。  
`site.yml`自身は編集する必要がなく、そのまま`ansible-playbook`コマンドで指定してplaybookを実行します。  
```
[root@mgmt]# cd /usr/share/ceph-ansible/
[root@mgmt]# cp site.yml.sample site.yml
[root@mgmt]# echo "retry_files_save_path = ~/" >> /etc/ansible/ansible.cfg	#retryファイルのパスをホームディレクトリに指定
[root@mgmt]# ansible-playbook site.yml
...
...
PLAY RECAP ***********************************************************************************************************
mon01		: ok=97   changed=22   unreachable=0    failed=0   
mon02		: ok=84   changed=18   unreachable=0    failed=0   
mon03		: ok=89   changed=19   unreachable=0    failed=0   
osd01		: ok=64   changed=14   unreachable=0    failed=0   
osd02		: ok=59   changed=12   unreachable=0    failed=0   
osd03		: ok=59   changed=12   unreachable=0    failed=0   
rgw		: ok=45   changed=14   unreachable=0    failed=0   
```

---

## クラスターの動作確認

### ステータスをチェックする
無事にplaybookがエラー無く実行完了したら動作確認をしてみます。  
いずれかのmonノードにログインしてクラスターのステータスを確認します。
```
[root@mon01]# ceph status
  cluster:
    id:     5765942e-be0e-4a70-8542-871bf4ced6c4
    health: HEALTH_WARN
            too few PGs per OSD (16 < min 30)
 
  services:
    mon: 3 daemons, quorum mon02,mon03,mon01
    mgr: mon02(active), standbys: mon03, mon01
    osd: 9 osds: 9 up, 9 in
    rgw: 1 daemon active
 
  data:
    pools:   6 pools, 48 pgs
    objects: 208 objects, 3359 bytes
    usage:   971 MB used, 45009 MB / 45980 MB avail
    pgs:     48 active+clean

```
`too few PGs per OSD`という内容のWARNINGが出ています。デフォルトで作られているプールに割り振られているPGが少ないようです。  
デフォルトのプールのPGを増やしてもいいのですが、せっかくなので新しいプールを作ってもう一度確認してみます。
```
[root@mon01]# ceph osd pool create mypool 256 256
pool ‘mypool’ created
[root@mon01]# ceph status
  cluster:
    id:     5765942e-be0e-4a70-8542-871bf4ced6c4
    health: HEALTH_OK
...
  data:
    pools:   7 pools, 304 pgs
    objects: 208 objects, 3359 bytes
    usage:   977 MB used, 45002 MB / 45980 MB avail
    pgs:     304 active+clean
```
クラスター全体でPGの数が増えたため、`HEALTH_OK`になりました。

### プールにオブジェクトをPUTする
次にMONノードからプールにオブジェクトをPUTしてみます。
```
[root@mon01]# echo "Hello world of ceph." > ~/test.txt

[root@mon01]# ls -l ~/test.txt
-rw-r--r--. 1 root root 21  2月 26 00:16 ~/test.txt
[root@mons-1 ~]# rados --pool mypool put helloworld ~/test.txt
[root@mons-1 ~]# rados --pool mypool ls
helloworld
[root@mons-1 ~]# rados --pool mypool df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD WR_OPS WR     
mypool      21       1      0      3                  0       0        0      0  0      1 1024 
...
```
`test.txt`というファイルを`hellowworld`というオブジェクト名で`mypool`にPUTした結果です。  
`USED`と`OBJECTS`にはそれぞれプールに置かれているオブジェクトのサイズと個数が表示されます。`WR_OPS`と`WR`にはそれぞれ書き込み回数と書き込まれた容量が表示されます。
`COPIES`が3であるのは`mypool`がデフォルトの3-way replicationで作られているためです。念の為確認してみましょう。
```
[root@mons-1 ~]# ceph osd pool get mypool size
size: 3
```

### プールからオブジェクトをGETする
別のノードにログインしてプールからオブジェクトをGETしてみます。
```
[root@mon01]# ssh mon02
[root@mon02]# rados --pool mypool ls
helloworld
[root@mons-2 ~]# rados --pool mypool get helloworld get.txt
[root@mons-2 ~]# ls -l get.txt
-rw-r--r--. 1 root root   21  2月 26 01:00 get.txt
[root@mons-2 ~]# cat get.txt 
Hello world of ceph.
[root@mons-2 ~]# rados --pool mypool df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD   WR_OPS WR   
mypool2     21       1      0      3                  0       0        0      1 1024      1 1024 
...
```
先程とは違うノードから`helloworld`オブジェクトをGETできました。`RD_OPS`と`RD`はそれぞれ読み込み回数と読み込み容量が表示されるところであり、GETしたことによりそれぞれ数値が変わったことが確認できます。

---

## Revision History
2017-02-26: 初版リリース
