# docker swarmを試す

## 試したいこと

・Serviceを使って外部からの接続をルーティングしたい
・embbed DNSで内部のルーティングをしたい
・ingressを使ってみたい。

https://hd35468.wordpress.com/2016/06/27/docker-loadbalancer/
https://docs.docker.com/engine/swarm/

## 準備

まずは読み解く。

https://docs.docker.com/engine/swarm/
tutorialでは1台のmanagerノードと2台のワーカーノードで構成されるものを作っている。
不足している情報
* ディスカバリバックエンドとしてのetcdの関わりがわからないこと
 * etcdはどこに立てればいいのか？そもそも必須なのか？
* 内部の通信がどのようにバランシングされるかわからないこと
 *


##

* docker-machineをインストールする。
* discovery backendをたてる

## installation


### docker-machineを公式の案内する方法にしたがってインストール

https://docs.docker.com/machine/install-machine/
うちのクラウドのdriverがないので本番では使えないだろうけど。
```
yudai@kashiwagi:~$ curl -L https://github.com/docker/machine/releases/download/v0.9.0-rc2/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
>   chmod +x /tmp/docker-machine &&
>   sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   600    0   600    0     0    711      0 --:--:-- --:--:-- --:--:--   711
100 24.1M  100 24.1M    0     0  2890k      0  0:00:08  0:00:08 --:--:-- 4196k
yudai@kashiwagi:~$ docker-machine version
docker-machine version 0.9.0-rc2, build 7b19591
```

https://docs.docker.com/engine/reference/commandline/swarm_join_token/

### manager node x1, worker node x2 を立てる

$ docker-machine create -d virtualbox manager
$ docker-machine create -d virtualbox worker1
$ docker-machine create -d virtualbox worker2

これでvirtualbox上に3台のVMが構築される。
ちなみにOSはboot2docker.isoから作成される専用のLinuxが入っているようだ。

IPは自動的にふられたので固定できていない気がするが、以下のようになった。

manager: eth0->10.0.2.15, eth1->192.168.99.101
worker1: eth0->10.0.2.15, eth1->192.168.99.102
worker2: eth0->10.0.2.15, eth1->192.168.99.103

eth0はvirtualboxのNATでInternetにつながっている状態。
eth1は連番でそれぞれのnodeで異なるIPが付与されている。

### manager nodeでswarm init

```
$ docker-machine ssh manager
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.12.6, build HEAD : 5ab2289 - Wed Jan 11 03:20:40 UTC 2017
Docker version 1.12.6, build 78d1802

docker@manager:~$ docker swarm init --advertise-addr 192.168.99.101
Swarm initialized: current node (a81drkkvbwp58k3qr4bspj5ho) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-14q2mbc3yfh8ygbt8bm4jnvsagifbdppla3v829qp0rjmj1mj9-14b1q96d3lzpr6gxckuatjuav \
    192.168.99.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

### worker nodeでswarm init

```
$ docker-machine ssh worker1
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.12.6, build HEAD : 5ab2289 - Wed Jan 11 03:20:40 UTC 2017
Docker version 1.12.6, build 78d1802
docker@worker1:~$ docker swarm join \
>     --token SWMTKN-1-14q2mbc3yfh8ygbt8bm4jnvsagifbdppla3v829qp0rjmj1mj9-14b1q96d3lzpr6gxckuatjuav \
>     192.168.99.101:2377
This node joined a swarm as a worker.
```

manager node側でjoinしたことを確認する。

```
$ docker-machine ssh manager
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.12.6, build HEAD : 5ab2289 - Wed Jan 11 03:20:40 UTC 2017
Docker version 1.12.6, build 78d1802
docker@manager:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
1zv6ym0fymbe0b8gxwimf3lgv    worker1   Ready   Active        
a81drkkvbwp58k3qr4bspj5ho *  manager   Ready   Active        Leader
```

同じことをworker2でも実行しておく。

### serviceをデプロイする。

```
$ docker-machine ssh manager
                        ##         .
                  ## ## ##        ==
               ## ## ## ## ##    ===
           /"""""""""""""""""\___/ ===
      ~~~ {~~ ~~~~ ~~~ ~~~~ ~~~ ~ /  ===- ~~~
           \______ o           __/
             \    \         __/
              \____\_______/
 _                 _   ____     _            _
| |__   ___   ___ | |_|___ \ __| | ___   ___| | _____ _ __
| '_ \ / _ \ / _ \| __| __) / _` |/ _ \ / __| |/ / _ \ '__|
| |_) | (_) | (_) | |_ / __/ (_| | (_) | (__|   <  __/ |
|_.__/ \___/ \___/ \__|_____\__,_|\___/ \___|_|\_\___|_|
Boot2Docker version 1.12.6, build HEAD : 5ab2289 - Wed Jan 11 03:20:40 UTC 2017
Docker version 1.12.6, build 78d1802
docker@manager:~$ docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
1zv6ym0fymbe0b8gxwimf3lgv    worker1   Ready   Active        
5qeaxpqrybhxw3uoaots0s0fw    worker2   Ready   Active        
a81drkkvbwp58k3qr4bspj5ho *  manager   Ready   Active        Leader
docker@manager:~$ docker service create --replicas 1 --name helloworld alpine ping docker.com
9abu17y6p7af0y2camrol2rog
docker@manager:~$ docker service ls
ID            NAME        REPLICAS  IMAGE   COMMAND
9abu17y6p7af  helloworld  1/1       alpine  ping docker.com

docker@manager:~$ docker service inspect --pretty helloworld
ID:		9abu17y6p7af0y2camrol2rog
Name:		helloworld
Mode:		Replicated
 Replicas:	1
Placement:
UpdateConfig:
 Parallelism:	1
 On failure:	pause
ContainerSpec:
 Image:		alpine
 Args:		ping docker.com
Resources:

docker@manager:~$ docker service ps helloworld
ID                         NAME          IMAGE   NODE     DESIRED STATE  CURRENT STATE               ERROR
8h78i1h9piwdh6fbwebb46du2  helloworld.1  alpine  manager  Running        Running about a minute ago  
```

よくわからんけどデプロイされたようだ。
managerで稼働していることがわかる。
説明にはmanager nodeではデフォルではサービスはデプロイされないと書いてあるのだが・・・

```
docker@manager:~$ docker service scale helloworld=5
helloworld scaled to 5
docker@manager:~$ docker service ps helloworld
ID                         NAME          IMAGE   NODE     DESIRED STATE  CURRENT STATE                   ERROR
8h78i1h9piwdh6fbwebb46du2  helloworld.1  alpine  manager  Running        Running 3 minutes ago           
6qso27khicl0qotit8org6ul9  helloworld.2  alpine  manager  Running        Running 3 seconds ago           
e35kzwfzsq7amiwhig3nzuuuo  helloworld.3  alpine  worker2  Running        Running less than a second ago  
6nhek8pwr6bzmwlgzttdxbzve  helloworld.4  alpine  worker1  Running        Running less than a second ago  
20lzck9ggqr25auat1wxba1qw  helloworld.5  alpine  worker1  Running        Running less than a second ago
```

スケールさせるとworkerにも配置された。

ここまでやってきたが、ロードバランシングの話は出てこなかった。
