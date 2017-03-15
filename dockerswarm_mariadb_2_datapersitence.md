# mariadbのデータ永続化について

## 概要

Galera clusterの残課題としてデータの永続化がある。

特に何も考慮していないコンテナを作った場合は、コンテナが再起動するたびにローカルに保存されたデータが初期化されてしまう。
Galeraクラスタは初期化されたノードが他のノードから最新の情報を受け取れるのである程度時間が経過すれば最新の状態になる。
しかし、例えば数ギガバイトのデータが保存されている場合はそれなりに復元に時間がかかってしまうだろう。

コンテナはボリュームという機能があり、コンテナのスケジューリング（起動、終了等）に影響を受けないローカルのデータの永続化が可能だ。
今回はそれを試す。

## 環境

dockerホスト x2

## 作業詳細

まずは以下のようにswarm-modeのserviceを `mode --global` で作ってみる。
DBのコンテナにはホストOSの `/tmp/mysql` をマウントさせる。（ `--mode global` は各ホストに1台ずつコンテナを立ち上げるオプションなので、問題ないが、`--repicas 3` とかどのホストでレプリケーションされるかわからないモードにした場合には同じホストにいるレプリカが同じフォルダを参照することになり、mysqlは動作しないどころか下手するとデータを壊すおそれがある）

各ホストで次の作業を行う。
```
$ mkdir /tmp/mysql
$ docker service create --name sv_galera_initnode \
--reserve-memory 100m \
--mode global \
--mount type=bind,source=/tmp/mysql/,target=/var/lib/mysql/ \
--network nw_galera \
erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address='gcomm://'

$ ls /tmp/mysql/
3ac7d1414524.pid  galera.cache  gvwstate.dat  ib_logfile1  mysql
auto.cnf          grastate.dat  ib_logfile0   ibdata1      performance_schema

```

2台ある内の一台目だけでdatabaseを作成して永続化されるか確認する。

```
$ sudo docker ps
CONTAINER ID        IMAGE                                                                                    COMMAND                  CREATED              STATUS              PORTS               NAMES
3ac7d1414524        erkules/galera@sha256:200631e44eacb15cab818bd84055a015bcff8430b588253c394d21834e826d85   "/entrypoint.sh --..."   About a minute ago   Up About a minute   3306/tcp            sv_galera_initnode.4snyinls8ei3oxa6u2rn43thh.iqs10oswswnwkp7i1n1durvod

$ docker exec -it 3ac7d1414524 mysql
mysql> create database testdb;
Query OK, 1 row affected (0.01 sec)

$ sudo docker service rm sv_galera_initnode

$ sudo docker service create --name sv_galera_initnode \
--reserve-memory 100m \
--mode global \
--mount type=bind,source=/tmp/mysql/,target=/var/lib/mysql/ \
--network nw_galera \
erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address='gcomm://'
```

経過は記述を割愛するが、databaseを見るとちゃんと永続化されている。問題なさそうだ。

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| testdb             |
+--------------------+
4 rows in set (0.00 sec)
```

2台でクラスタを組んで片方が落ちて復帰したときの動作を確認する。


どちらのgaleraで操作しても良いのでもう片方のIPを調べておいて以下のようにする。
```
$ sudo docker service ps sv_galera_initnode                                                            
ID            NAME                                          IMAGE                 NODE           DESIRED STATE  CURRENT STATE          ERROR  PORTS
k2twzmy5k8hd  sv_galera_initnode.jyx0mmsy2rma9uk60fqj1fkw9  erkules/galera:basic  dockerhost005  Running        Running 7 minutes ago         
op17l6seta6i  sv_galera_initnode.4snyinls8ei3oxa6u2rn43thh  erkules/galera:basic  dockerhost004  Running        Running 7 minutes ago    

msyql> set global wsrep_cluster_address='gcomm://10.0.0.3';
```

なぜかここで落ちる。ログを見るとバグかもよ、と書いてあるのだが。
とりあえず別のレプリカを作るか。

```
mkdir /tmp/mysql2/  

docker service create --name sv_galera \
--reserve-memory 100m \
--mode global \
--mount type=bind,source=/tmp/mysql/,target=/var/lib/mysql2/ \
--network nw_galera \
erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address='gcomm://10.0.0.3'

mysql> show databases;                                                                                                           
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| testdb             |
+--------------------+
4 rows in set (0.01 sec)

一台目のdatabaseが反映されている。ちゃんと更新が伝わっているようだ。

消す
sudo docker rm -f c88265ca6bb5

消すと自動的にデータベースをswarm modeの機能で復元してしまうので、あまり良い検証にならなかった。
やるとしたらホストごと消すようにしないとならないな。
```
