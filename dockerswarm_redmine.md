# Docker SwarmでRedmineの環境を構築

## 概要

Redmine + MySQLをDocker Swarm上で構築する。
`dockerswarm_galera.md` で構築した環境を引き続き使用する。

## 準備

事前にDBにredmineというユーザとdatabaseを作っておく。
DBサーバは `dockerswarm_galera.md` で作成したクラスタのどれか一台でOK。

```
mysql> create database redmine default character set utf8;
Query OK, 1 row affected (0.01 sec)

MySQL [mysql]> grant all on redmine.* TO redmine@'%' IDENTIFIED BY 'test123';
Query OK, 0 rows affected (0.00 sec)

MySQL [mysql]> flush privileges;
Query OK, 0 rows affected (0.01 sec)
```
`default character set` をUTF8にしておくのは重要。
あとからRedmineの言語設定を日本語に変えた時にエラーが出ないようにするために。

### Redmineコンテナを3台たてる

```
docker@galera-1:~$ docker service create --name sv_redmine \
--reserve-memory 100m \
--replicas 3 \
--network nw_galera \
-e 'REDMINE_DB_MYSQL=sv_galera' \
-e 'REDMINE_DB_USERNAME=redmine' \
-e 'REDMINE_DB_PASSWORD=test123' \
-e 'REDMINE_DB_DATABASE=redmine' \
-e 'REDMINE_DB_ENCODING=utf8mb4' \
-p 8080:3000 \
redmine
```

sv_galeraは `dockerswarm_galera.md` で構築した `galera cluster` のサービスである。
おなじ `nw_galera` というネットワーク内にいればホスト名として参照でき、3306/tcpに起動しているmysqlにアクセスできる。

環境変数は以下のgithubを確認せよ。

https://github.com/docker-library/redmine/blob/c137cd0e39758f56be851cbd56273cd486590149/3.1/docker-entrypoint.sh

とりあえず、これでRedmineが出来上がった。
添付ファイルをすべてのRedmineサーバで共有できていない問題はあるが。
これはs3fsで解決すれば良いと思われる。
