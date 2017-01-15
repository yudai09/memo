# tips: docker network createで作成したネットワークの中にいるコンテナのIPを調べる

## 概要/目的

swarm mode(docker engine 1.12以降)を使うとserviceという単位でコンテナが管理できる。
これは一つ一つのコンテナを意識せずに群として扱うことができる便利な機能だ。
しかしいろんなケースで個々のIP(というかVIP)を知りたくなることがある。

ここではそのVIPを調べる方法を説明する。他にも良い方法があるかもしれない。

## 2つの方法

* 内部DNSに聞く
* docker network inspectを使う。

### 内部DNSに聞く場合

コンテナ同士のバランシングは内部DNSとingress load balancerを使っている。
内部DNSは以下のようにサービスを表すVIPが登録されていて、このVIPにアクセスすることでコンテナ間のバランシングが行われている。

```
user@docker-host$ docker network create --driver overlay nw_mongo

user@docker-host$ docker network ls -f 'driver=overlay'
NETWORK ID          NAME                DRIVER              SCOPE
dj75i6yn1vrj        ingress             overlay             swarm
nnbuc2pj3jot        nw_mongo            overlay             swarm

# docker service createの過程は割愛

user@docker-host:~$ docker service ps sv_mongo -f 'desired-state=running'
ID            NAME        IMAGE         NODE    DESIRED STATE  CURRENT STATE              ERROR  PORTS
hzdeqjmn41h3  sv_mongo.1  mongo:3.2.10  node-1  Running        Running about an hour ago
zq0cs5l4y4l2  sv_mongo.2  mongo:3.2.10  node-1  Running        Running about an hour ago
yi28a45w1s91  sv_mongo.3  mongo:3.2.10  node-1  Running        Running 22 hours ago

@どこかのコンテナの中（digが入っていること前提）
user@container:~$ dig sv_mongo +short
10.0.0.2
```

sv_mongoというサービスは合計で3つのコンテナが動いていることがわかるが、
sv_mongoという名前を引くと10.0.0.1というIPがひとつだけ返ってくる。
このIPにアクセスすると3つのコンテナにバランシングされる。

ではこのIPに対して、バランシング対象のIPは何なのかを調べる方法は以下の通り。

```
user@container:~$ dig tasks.sv_mongo +short
10.0.0.5
10.0.0.6
10.0.0.7
```

ドキュメントには今のところ（2017/1）記載されていないが、いろんなところで使われているようだ。

### docker network inspectを使う場合

dockerのホスト上で以下を実行する。

```
user@docker-host:~$ docker network inspect nw_mongo -f "{{range .Containers}} {{.IPv4Address}} {{end}}"
 10.0.0.5/24  10.0.0.6/24  10.0.0.7/24
```

少し詳細を説明すると、`docker network inspect nw_mongo` をオプションを付けず実行した場合に、
結果はネストしたものになる。そこからIPを取り出すために `-f` で指定したgo templateを使っている。
帰ってくる構造が変わると役に立たなくなる点に注意が必要。

# よく考えるとsv_mongo, nw_mongoを混同している。
  nw_mongoの中にいるホストは `docker network inspect` で容易に入手可能だが、
  コンテナ内部で入手する方法は見つからない。

  一方でsv_mongoの中にいるホストは `dig tasks.*` で入手可能。
  dockerコマンドでsv_mongoの中にいるホストは入手できない。（どのコマンドを使ってもうまく行く方法が見つからなかった。

  service - network の関係性を外部から入手する綺麗な方法がないのは困るなぁ。
  そのうち改善されるのかね。
