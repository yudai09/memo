
# setup

ネットワークを作る
```
$ docker network create nw_mongodb --driver overlay
```

mongodbと1:1になるようにサービスをつくる。
```
$ for i in $(seq 3); do \
docker service create --name sv_mongodb$i \
--reserve-memory 100m \
--network nw_mongodb \
--replicas 1 \
mongo:3.2.10 mongod --replSet "rs/sv_mongodb1,sv_mongodb2,sv_mongodb3"; \
done
```

# repica setの初期化

各コンテナでmongodbのrepica setを初期化する
各コンテナが収容されているdockerホストを見つけるのが面倒なのでsv_mongo_utilというコンテナを全サーバに配置して使う。

```
$ docker service create --name sv_mongo_util \
--reserve-memory 100m \
--mode global \
--network nw_mongodb \
mongo:3.2.10 tail -f /dev/null

$ docker exec -it $(docker ps -q --filter label=com.docker.swarm.service.name=sv_mongo_util) sh
# mongo sv_mongodb1:27017
rs.initiate()
rs.status()
```

# centosのコンテナを作りmongodbにアクセスしてみる。

```
$ docker service create --name sv_pytest \
--reserve-memory 100m \
--replicas 3 \
--network nw_mongodb \
-p 80:80 \
centos:7 tail -f /dev/null

# docker exec -it ${id_of_pytest} bash
$ docker@node-1:~$ docker exec -it 17cb8d3c7ec0 bash
[root@17cb8d3c7ec0 /]# yum install python-setuptools -y
[root@17cb8d3c7ec0 /]# easy_install pymongo
[root@17cb8d3c7ec0 /]# python
>>> from pymongo import MongoClient
>>> c = MongoClient('sv_mongodb1',replicaset='rs')
>>> print c
MongoClient(host=['sv_mongodb1:27017'], document_class=dict, tz_aware=False, connect=True, replicaset='rs')
>>> c.nodes
frozenset([(u'sv_mongodb3', 27017), (u'043f62218c07', 27017), (u'sv_mongodb2', 27017)])
```

043f62218c07の部分は改善の余地あり
