## ceph-ansibleを使ったRed Hat Ceph Storage 3クラスターのデプロイ

<<<<<<< HEAD
RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました。(ceph-deployはdeprecated)
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、1ノードの1デーモンをデプロイするオーソドックスなスタイルとします。*(コンテナデプロイは別機会に書きます)*
=======
RHCS2からcephクラスターのデプロイはceph-ansibleを利用するようになりました (ceph-deployはdeprecated)。  
そこでceph-ansibleを使ったクラスターのデプロイ手順を記載します。  
なお今回はceph daemonをコンテナとしてデプロイするスタイルではなく、1ノードの1デーモンをデプロイするオーソドックスなスタイルとします。
>>>>>>> d4a53f606a13336f74bce0fda1616646d5942135


次のようなイメージのcephクラスターをデプロイする手順を記載します。
*簡単にテスト環境を作るのに参考になれば幸いです。

### 事前準備
hoge
