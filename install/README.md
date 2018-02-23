## ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました。(ceph-deployはdeprecated)  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、1ノードの1デーモンをデプロイするオーソドックスなスタイルとします。*(コンテナデプロイは別機会に書きます)*

次のようなcephクラスターをイメージしてデプロイする手順を記載します。


![クラスターイメージ](https://github.com/tutsunom/rhcs/blob/master/install/image/cluster.png)


### 事前準備
hoge
