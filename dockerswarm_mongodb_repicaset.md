# docker swarmでmongodbのrepica setを試す。

## やりたいこと

docker swarmはクラスタを容易につくることができる。ただしそれはステートレスなアプリケションの場合だけだ。
ステートフルなアプリケーションはいろいろ考慮した上で作らなければならない。

ステートフルなアプリケーションとしてDBを扱ってみる。
なるべく綺麗な作りにしたいので、クラスタ環境に向いたDBを使って検証してみる。

mongodbの場合は多分クラスタ環境に適していると思う。
https://docs.mongodb.com/manual/replication/
クラスタリング(replica set)を組めるようになっているからきっとDocker Swarmとも相性がいいはずだ。

## mongodbの勉強

Mongodbのクラスタの扱いかたがわからないだけではなく、そもそもMongodbがどんなDBなのかも全然知らないので勉強する。Mongodbの薄い本というものが無料で読めるので、これで勉強しはじめる。

http://www.cuspy.org/diary/2012-04-17/the-little_mongodb-book-ja.pdf
ざっくり読んで理解した気がするレベルになった。

レプリケーションについては以下でイメージを掴んだ。

http://gihyo.jp/dev/serial/01/mongodb/0004

* 書き込みができるのはPrimaryノードだけらしい。
  * pymongoのサンプルを見るとPrimaryが死んだ時に他のノードから情報を採取して新しいPrimaryのIPを知ることができる。
   * https://api.mongodb.com/python/current/examples/high_availability.html
  * アプリケーションがSecondaryを含むすべてのノードのIPに対して直にアクセスできないとマスターがフェイルオーバした時に復旧できないかも
* Oplogを共有ストレージ領域においておいて、スケールアウトを高速化しないとオートスケールは実用的にならない。

ローカルの環境でmongodbを3台たててreplica setを作ってみた。
そしてpymongoで
```
# hostname is kashiwagi

$ mkdir /tmp/db1 /tmp/db2 /tmp/db3
$ mongod --port 27017 --dbpath /tmp/db1 --replSet testrs/kashiwagi:27018,kashiwagi:27019 --rest &
$ mongod --port 27018 --dbpath /tmp/db2 --replSet testrs/kashiwagi:27017 --rest &
$ mongod --port 27019 --dbpath /tmp/db3 --replSet testrs/kashiwagi:27017 --rest &

# kashiwagi:27017にアクセスして以下を実行。
$ mongo
> rs.initiate()
> rs.status()
-> initiate()したあとは勝手にmasterを決めてくれる。

# pymongoでアクセスしてみる。
$ python
>>> from pymongo import MongoClient
>>> c = MongoClient(replicaset='testrs')
>>> c.nodes
frozenset([(u'kashiwagi', 27018), (u'kashiwagi', 27017), (u'kashiwagi', 27019)])

-> 1台に尋ねれば他のnodeの情報が取得できる。
```

## docker swarmでクラスタを作る方法を検討する。

ググるとdockerでmongoクラスターを作る方法は以下のようなものが見つかった。

https://medium.com/@kalahari/running-a_mongodb-replica-set-on-docker-1-12-swarm-mode-step-by-step-a5f3ba07d06e#.5jadtb9yn
```
$ docker service create --replicas 1 --network mongo --mount type=volume,source=mongodata1,target=/data/db --mount type=volume,source=mongoconfig1,target=/data/configdb --constraint 'node.labels.mongo.replica == 1' --name mongo1 mongo:3.2 mongod --replSet example
b29ftmx77l75owmhmkswntcmz
docker@manager1:~$ docker service create --replicas 1 --network mongo --mount type=volume,source=mongodata2,target=/data/db --mount type=volume,source=mongoconfig2,target=/data/configdb --constraint 'node.labels.mongo.replica == 2' --name mongo2 mongo:3.2 mongod --replSet example
1o08g3f0b6ub60et2u7uu9bc5
docker@manager1:~$ docker service create --replicas 1 --network mongo --mount type=volume,source=mongodata3,target=/data/db --mount type=volume,source=mongoconfig3,target=/data/configdb --constraint 'node.labels.mongo.replica == 3' --name mongo3 mongo:3.2 mongod --replSet example
715k8yuv3uodxn2cyai43uxy2
```

先述の通りすべてのノードのIPをアプリが意識できるようにするためにノードごとに新しいサービスを作っている。
サービスは本来は複数のコンテナを束ねるもののはずだが、そのように扱えていない。
あまりかっこよくない。

## こっからサンプル作る。

イメージ。こんな感じにしたい。
```
$ docker network create nw_mongo --driver overlay
$ docker service create --name sv_mongo \
--reserve-memory 100m \
--network nw_mongo \
--replicas 3 \
mongo:3.2.10 mongod --replSet "rs/sv_mongo"
```

でも無理。sv_mongoというサービスはsv_mongoの中の誰か（一番近い自分になるのかな）1つにアクセスをバランシングさせてしまう。mongodbが他の全員を知る手段がない。
また、アプリケーションからも個々のmongoが見えないのでPrimaryだけにアクセスする方法がない。

### こうしたい

少なくともmongo同士は全員のIPがわかるようにしたい。はじめに人間が入れるのではなく、自動的に。
アプリケーションはmongoをクラスタとして使えるようにしたい。個別のIPを意識したくない。

### 調べた。

http://dba.stackexchange.com/questions/130321/can_mongodb-be-configured-to-sit-behind-a-load-balancer

これいい回答だな。
mongos使えば解決できそう。
まずはローカルで試してみよう。

いや、よく考えるとmongoとアプリケーションがセットなら問題ないのか。
アプリケーションとmongoを一緒にスケールさせなければならないのが問題だが。
規模を考えればそれほどお問題にならない

２つの場合の構成を検証してみよう。
まずはmongosから。

mongosを構成するためにはconfig serverが必要になるらしい。
結局config serverにすべてのmongoのIPアドレスの情報を把握させる必要がある。
そうなるとmongodの追加削除をconfig serverが把握する必要がある。
そして、問題が再燃することになる。

やっぱり一番良いのはregistrator的なものがあって、nodeの参加があるたびにそれを特定のサーバに通知してくれるのが良い。

etcd/consulを使うべきかな。
そういえば、Docker Flow Proxyはどうやっているのだろうか。

#### interlock

https://forums.docker.com/t/swarm-mode-and-service-discovery/16563/6
ここで外部からswarm modeのDNSにアクセスする方法がないのかという話をしている。

2016/11にinterlockというプロジェクトでHAProxyのための何かをやっていると書いていて、ソース見ろと言っている。

https://github.com/ehazlett/interlock

コンテナの生成/削除を検知してHAProxyを設定するものらしい。
interlock(内部結合？)のためのもの。registratorと同じ役割だな。
このプロジェクトでoverlay networkの対応をしているから、それと似せればmongoのconfigにも適用可能だろう。ただかなり大変だし、Dockerが仕組みを作ってくれるのが一番良い。

#### dockerプロジェクトでmongoについてのissueが上がっていないか確認する。

https://github.com/docker/docker/issues/28358
servciceにした時にmongoの内部でホスト一覧を生成するときにコンテナのホスト名で登録されてしまい、うまく名前解決できなくなる。たしかにそうだろう。ホスト名ではなく、IP:Portをconfigを明示的に渡してやる必要があるなぁ。

### docker swarm-mode / mongodb config server

```
busbyjonJon Busby
Jul '16

I've had some success using the "interlock" project (although you'll need to use their swarm mode branch at the moment) to act as a HAProxy and to handle vhosts on each node..
```

registratorにswarm-modeに対応したものがある？
https://github.com/gliderlabs/registrator/pull/476


### mongos

mongos + repica set
ref: http://gihyo.jp/dev/serial/01/mongodb/0006


## 所感

devops本にもあったけど、アプリケーションに適したロードバランサは必要だと思う。
