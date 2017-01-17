# galera-with-docker

galera clusterをdockerのswarm mode上に作る。

## 断り
クラスタを作成するときに参加する先のノードのIPを決める必要があるのがめんどくさい。
できればそれをなくしたい。
tasks.sv_gareraをdigした結果のIPを並べたものを起動オプションに加えてmysqldを起動するようにdocker imageをbuildすればよいのだろう。

## 参照先

http://galeracluster.com/2015/05/getting-started-galera-with-docker-part-1/
http://galeracluster.com/documentation-webpages/docker.html
作り方は簡単そうだなぁ。

### Docker Machineで3台のノードを構築して検証環境を作る

3台のDocker Machineをたてる。
```
yudai@kashiwagi:~$ for i in 1 2 3; do   docker-machine create -d virtualbox galera-$i; done
```
毎回クリーンな環境で検証が始められるのが素晴らしい。

さっそうとswarm modeへ移行する。

```
docker@galera-1:~$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (yt8y52dkiv1t7ev28bkom0x7y) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-5lvf3blvqwxxxb77thdhoesfylcmzee3qec05gwr54ds0t2ewu-3dbjndx6jke16z0j75lnm9m2z \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

docker@galera-2:~$ docker swarm join \
> --token SWMTKN-1-5lvf3blvqwxxxb77thdhoesfylcmzee3qec05gwr54ds0t2ewu-3dbjndx6jke16z0j75lnm9m2z \
> 192.168.99.100:2377
This node joined a swarm as a worker.

docker@galera-3:~$ docker swarm join \
> --token SWMTKN-1-5lvf3blvqwxxxb77thdhoesfylcmzee3qec05gwr54ds0t2ewu-3dbjndx6jke16z0j75lnm9m2z \
> 192.168.99.100:2377
This node joined a swarm as a worker.
```

### ネットワーク構築

```
docker@galera-1:~$ docker network create nw_galera --driver overlay

kawhbt1bn4566ey4m8vm677xc

### galeraの最初のノードと残りのノードx3をたてる

docker@galera-1:~$ docker service create --name sv_galera_initnode \
--reserve-memory 100m \
--replicas 1 \
--network nw_galera \
erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address='gcomm://'

docker@galera-1:~$ docker service create --name sv_galera \
--reserve-memory 100m \
--replicas 3 \
--network nw_galera \
erkules/galera:basic --wsrep-cluster-name=local-test --wsrep-cluster-address='gcomm://sv_galera_initnode'

7ve0xe155zttxvc0w22oxnz4w

docker@galera-1:~$ docker service ps sv_galera_initnode
ID            NAME                  IMAGE                 NODE      DESIRED STATE  CURRENT STATE           ERROR  PORTS
gwwlvdxow0dl  sv_galera_initnode.1  erkules/galera:basic  galera-3  Running        Running 37 seconds ago

docker@galera-1:~$ docker service ps sv_galera
ID            NAME         IMAGE                 NODE      DESIRED STATE  CURRENT STATE           ERROR  PORTS
rm9bc4pabvp9  sv_galera.1  erkules/galera:basic  galera-2  Running        Running 27 seconds ago
7ojxw5ubbqh4  sv_galera.2  erkules/galera:basic  galera-1  Running        Running 27 seconds ago
z2sm7g5731it  sv_galera.3  erkules/galera:basic  galera-3  Running        Running 27 seconds ago
```

### sv_galera_initnodeが他のノードとつながったか確認する

```
docker@galera-3:~$ docker ps
CONTAINER ID        IMAGE                                                                                    COMMAND                  CREATED              STATUS              PORTS               NAMES
9e4752d1952c        erkules/galera@sha256:200631e44eacb15cab818bd84055a015bcff8430b588253c394d21834e826d85   "/entrypoint.sh --..."   About a minute ago   Up About a minute   3306/tcp            sv_galera.3.z2sm7g5731ittxyjntaqgr2wn
45978d81dc71        erkules/galera@sha256:200631e44eacb15cab818bd84055a015bcff8430b588253c394d21834e826d85   "/entrypoint.sh --..."   About a minute ago   Up About a minute   3306/tcp            sv_galera_initnode.1.gwwlvdxow0dl6c1ciktjpdp13
4b2e3065feba        centos@sha256:c577af3197aacedf79c5a204cd7f493c8e07ffbce7f88f7600bf19c688c38799           "tail -f /dev/null"      37 minutes ago       Up 37 minutes                           sv_util_galera.f4j9f44uv2jqf83rn1cq6koff.1yifo58j850ezo4qhrryay3uh
docker@galera-3:~$ docker exec -it 45978d81dc71 mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 4
Server version: 5.6.28-1trusty (Ubuntu), wsrep_25.13

Copyright (c) 2000, 2015, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show status like 'wsrep%';
+------------------------------+-----------------------------------------------------------+
| Variable_name                | Value                                                     |
+------------------------------+-----------------------------------------------------------+
| wsrep_local_state_uuid       | ac6735e9-dc5b-11e6-8ffc-2b74aeb72d8e                      |
| wsrep_protocol_version       | 7                                                         |
| wsrep_last_committed         | 0                                                         |
| wsrep_replicated             | 0                                                         |
| wsrep_replicated_bytes       | 0                                                         |
| wsrep_repl_keys              | 0                                                         |
| wsrep_repl_keys_bytes        | 0                                                         |
| wsrep_repl_data_bytes        | 0                                                         |
| wsrep_repl_other_bytes       | 0                                                         |
| wsrep_received               | 9                                                         |
| wsrep_received_bytes         | 619                                                       |
| wsrep_local_commits          | 0                                                         |
| wsrep_local_cert_failures    | 0                                                         |
| wsrep_local_replays          | 0                                                         |
| wsrep_local_send_queue       | 0                                                         |
| wsrep_local_send_queue_max   | 1                                                         |
| wsrep_local_send_queue_min   | 0                                                         |
| wsrep_local_send_queue_avg   | 0.000000                                                  |
| wsrep_local_recv_queue       | 0                                                         |
| wsrep_local_recv_queue_max   | 1                                                         |
| wsrep_local_recv_queue_min   | 0                                                         |
| wsrep_local_recv_queue_avg   | 0.000000                                                  |
| wsrep_local_cached_downto    | 18446744073709551615                                      |
| wsrep_flow_control_paused_ns | 0                                                         |
| wsrep_flow_control_paused    | 0.000000                                                  |
| wsrep_flow_control_sent      | 0                                                         |
| wsrep_flow_control_recv      | 0                                                         |
| wsrep_cert_deps_distance     | 0.000000                                                  |
| wsrep_apply_oooe             | 0.000000                                                  |
| wsrep_apply_oool             | 0.000000                                                  |
| wsrep_apply_window           | 0.000000                                                  |
| wsrep_commit_oooe            | 0.000000                                                  |
| wsrep_commit_oool            | 0.000000                                                  |
| wsrep_commit_window          | 0.000000                                                  |
| wsrep_local_state            | 4                                                         |
| wsrep_local_state_comment    | Synced                                                    |
| wsrep_cert_index_size        | 0                                                         |
| wsrep_causal_reads           | 0                                                         |
| wsrep_cert_interval          | 0.000000                                                  |
| wsrep_incoming_addresses     | 10.0.0.3:3306,10.0.0.5:3306,10.0.0.10:3306,10.0.0.11:3306 |
| wsrep_evs_delayed            |                                                           |
| wsrep_evs_evict_list         |                                                           |
| wsrep_evs_repl_latency       | 0/0/0/0/0                                                 |
| wsrep_evs_state              | OPERATIONAL                                               |
| wsrep_gcomm_uuid             | ac66f6de-dc5b-11e6-a50d-0e30da5de74f                      |
| wsrep_cluster_conf_id        | 2                                                         |
| wsrep_cluster_size           | 4                                                         |
| wsrep_cluster_state_uuid     | ac6735e9-dc5b-11e6-8ffc-2b74aeb72d8e                      |
| wsrep_cluster_status         | Primary                                                   |
| wsrep_connected              | ON                                                        |
| wsrep_local_bf_aborts        | 0                                                         |
| wsrep_local_index            | 0                                                         |
| wsrep_provider_name          | Galera                                                    |
| wsrep_provider_vendor        | Codership Oy <info@codership.com>                         |
| wsrep_provider_version       | 3.14(r53b88eb)                                            |
| wsrep_ready                  | ON                                                        |
+------------------------------+-----------------------------------------------------------+
56 rows in set (0.01 sec)

```

### 試しにdatabaseを増設してそれが他のノードにも反映される確認する

sv_garera_initnode上で以下を実行

```
mysql> create database hoge;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| hoge               |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)
```

他のノードで以下を実行

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| hoge               |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.00 sec)
```
