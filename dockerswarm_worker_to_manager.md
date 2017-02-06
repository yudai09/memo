# Docker Swarmのworkerをmanagerにしてみた

3台構成のクラスタをmanager x1, worker x2で構築したが、
manager x3にしてみた。

方法は単純に
`docker swarm leave` してから `docker swarm join` しただけだった。

だが、問題がおきた。
コンテナを増やそうとして、 `docker service scale sv_redmine_custom=3` としたところ、
以下のようなエラーが出るようになった。

```
$ docker service ps sv_redmine_custom
ID                         NAME                     IMAGE           NODE           DESIRED STATE  CURRENT STATE                     ERROR
4c268uz9xlqehwp4a1gi34tea  sv_redmine_custom.1      redmine:custom  dockerhost002  Ready          Preparing less than a second ago  
0onhw0vlpc02dgwt07qi2ybd5   \_ sv_redmine_custom.1  redmine:custom  dockerhost002  Shutdown       Rejected 1 seconds ago            "No such image: redmine:custom"
7fgu8u7aw3wbu9mexwicdtn13   \_ sv_redmine_custom.1  redmine:custom  dockerhost002  Shutdown       Rejected 6 seconds ago            "No such image: redmine:custom"
bch3oozc7qnm3excf8261bdoq   \_ sv_redmine_custom.1  redmine:custom  dockerhost002  Shutdown       Rejected 11 seconds ago           "No such image: redmine:custom"
cam1f6kuper5ktmn2lcfp755j   \_ sv_redmine_custom.1  redmine:custom  dockerhost002  Shutdown       Rejected 16 seconds ago   
```

もともとイメージをビルドしたのが `dockerhost001` だったのだがそれが `dockerhost002` にはないのだ。
いやというかいままできづいていなかっただけか・・・
うーむ
workerをmanagerに昇格させたのが原因ではない笑
