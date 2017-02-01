# 西側にDockerホストを作成して、クラスタにジョインさせる。

東側のホストからアクセスするときのレイテンシーの高い西側の環境にDockerホストを構築し、
グローバル・ネットワーク経由でDocker SwarmクラスタおよびGaleraのクラスタにホスト及びGaleraノードを追加する。
とりあえず暗号化はしない状態で検証する。

## やりたいこと

目的としては東西に分けることで耐障害性があることを確認するため。
Galeraのクラスタの台数を増やして西側にコンテナが起動した上でデータが複製されることを確認したい。
西側からWebを閲覧して問題なく見れることを確認する。

## 構成
| hostname    | region | detail       |
|:------------|:------:|:-------------|
|dockerhost001|east    |構築済み      |
|dockerhost002|east    |構築済み      |
|dockerhost003|west    |今回構築      |

## 準備

### クラスタに参加するためのtokenの入手(worker)

既存ホストにて以下を実行。
```
dockerhost001$ docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-xxxxxxxxxxxxx \
    x.x.x.x:2377
```

### FWの開放。東西のFWにDocker swarmのポート開放が必要。
https://docs.docker.com/engine/swarm/swarm-tutorial/
```

Open ports between the hosts¶

The following ports must be available. On some systems, these ports are open by default.

    TCP port 2377 for cluster management communications
    TCP and UDP port 7946 for communication among nodes
    TCP and UDP port 4789 for overlay network traffic
```

### VM（CentOS)を作成して、dockerをインストール

```
dockerhost003$ curl -fsSL https://get.docker.com/ | sudo sh
sudo systemctl enable docker && sudo systemctl start docker  
```

## クラスタにノードをjoinさせる作業

### dockerhost003をDocker swarmクラスタにjoin

```
dockerhost003$ sudo docker swarm join \
  --token SWMTKN-xxxxxxxxxxxxx \
  x.x.x.x:2377
```

ネットワークが足らない。なぜ？
```
dockerhost003$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

dockerhost003$$ sudo docker network ls                                                                                                                  
NETWORK ID          NAME                DRIVER              SCOPE
1c28cf971952        bridge              bridge              local
ca7d0598f4aa        docker_gwbridge     bridge              local
7fae388d72f2        host                host                local
4u3z0r6c7j5s        ingress             overlay             swarm
d19281ec83d1        none                null                local
```

この状態でもちゃんと外部からアクセスすればWebアプリが表示されたので問題なさそう。
よくわからない。


## galera node追加

```
$ docker service ls
ID            NAME                REPLICAS  IMAGE                 COMMAND
68kz2imoicck  sv_galera           3/3       erkules/galera:basic  --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://sv_galera_initnode
99mu0ck0j4c1  sv_galera_initnode  1/1       erkules/galera:basic  --wsrep-cluster-name=local-test --wsrep-cluster-address=gcomm://
9ey1sthbpz7d  sv_redmine_custom   1/1       redmine:custom        
biqucmtn774w  sv_redmine          1/1       redmine      
```

sv_galeraをスケールアウトする。
```
docker service scale 68kz2imoicck=8
```

何やらエラーが出る。
```
Feb  1 15:51:20 localhost dockerd: time="2017-02-01T15:51:20.913863815+09:00" level=error msg="fatal task error" error="mkdir /var/lib/docker/overlay/6323971999a7176cb9c0e9b50e0fd1cfa682d488e6a37e7a28e6e478904a3a10-init/merged/dev/shm: invalid argument" module="node/agent/taskmanager" task.id=5ksyptp5fobotfzis56b6xdwe
```

以下の現象と同じだと思われる。
https://github.com/docker/docker/issues/10294

```
so, update kernel from 3.10.0 to 3.18.0 + fixed the issue.
```

まじか。
```
$ uname -a                                                                                                                                
Linux dockerhost001 3.10.0-229.el7.x86_64 #1 SMP Fri Mar 6 11:36:42 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

困った。

```
$ sudo yum install kernel
====================================================================================================================================================================
 Package                                 Arch                            Version                                             Repository                        Size
====================================================================================================================================================================
Installing:
 kernel-debug                            x86_64                          3.10.0-514.6.1.el7                                  updates                           39 M
Updating:
 kmod                                    x86_64                          20-9.el7                                            base                             115 k
 xfsprogs                                x86_64                          4.5.0-9.el7_3                                       updates                          895 k
Updating for dependencies:
 linux-firmware                          noarch                          20160830-49.git7534e19.el7                          base                              31 M

Transaction Summary
====================================================================================================================================================================
Install  1 Package
Upgrade  2 Packages (+1 Dependent package)
```

`xfsprogs` というのが入っているので、もしかしたらbugfixがバックポートされているかも。
とおもってインストールしてみた。

```
sudo reboot
```

再起動したのがよかったのか `xfsprogs` が入ったことが良かったのかはわからないけど直った。めでたし。
普通に考えると再起動が良かったのだと思う。

その後もう一度 `docker service scale 68kz2imoicck=8` を7や6にして増減させた結果 `dockerhost003` にコンテナが構築された。データも挿入されているので全く問題なさそう。
