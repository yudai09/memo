# redmine_phonecallの開発

https://github.com/yudai09/redmine_phonecall
上のプラグインのソースコードをレビューするために社内環境のProxy下で開発環境を構築した。

## vagrantでcentosのVMを作成

次の手順通り実施する

https://atlas.hashicorp.com/centos/boxes/7

環境

| ホストOS | ハイパーバイザー  | ゲストOS     |
|:---------|:------------------|:-------------|
| Windows7 | Virtualbox        | CentOS7      |


rsyncが必要だったのでCygwinを入れてそこからvagrantを実行した。

またプラグインのソースコードを編集しながら動作検証するためにホストとゲストでフォルダを共有する設定をおこなった。設定方法は次の記事を参照した。

http://qiita.com/HirofumiYashima/items/6044cfc64cfa3e84f97c

VirtualBoxの設定が完了するとゲストが以下の方法で共有フォルダをマウントできた。
```
mkdir /mnt/redmine
mount -t vboxsf docker_redmine /mnt/redmine
```

## dockerをインストール

次の手順の通り実施した。
https://docs.docker.com/engine/installation/linux/centos/

プロキシー配下で動かす場合はdockerの起動スクリプトに修正が必要だった。
```
vi /usr/lib/systemd/system/docker.service

# systemdの場合は以下のようにしてproxyを設定してあげる必要がある。
Environment='http_proxy=http://proxy.example.com:8080/'
# proxyのIPとdocker0のIP帯が競合するのでdocker0のIPをずらす
ExecStart=/usr/bin/dockerd --bip=192.168.42.1/24 --fixed-cidr=192.168.42.128/25
```

## redmine/mariadbの構築

pluginは所定のディレクトリにマウントしてあげることでredmineに取り込まれる。

```
sudo docker run --name db -d \
-e "MYSQL_ROOT_PASSWORD=********" \
-e "MYSQL_DATABASE=redmine" \
-e "MYSQL_USER=redadmin" \
-e "MYSQL_PASSWORD=redminepass" \
mariadb --character-set-server=utf8 --collation-server=utf8_unicode_ci

sudo docker run --name redmine -d \
-p 8888:3000 \
--link db:db \
-v '/mnt/redmine/redmine_phonecall/plugins:/usr/src/redmine/plugins' \
-e  "http_proxy=http://proxy.example.com:8080" \
-e  "HTTP_PROXY=http://proxy.example.com:8080" \
-e "MYSQL_ENV_MYSQL_ROOT_PASSWORD=********" \
-e  "REDMINE_DB_MYSQL=db" \
-e  "REDMINE_DB_DATABASE=redmine" \
-e  "REDMINE_DB_ENCODING=utf8" \
redmine:3.3
```

* `utf8mb4` にするとうまく起動しなので諦めて `utf8` とした。
* docker-composeで組んだほうがかっこよいのだが、linkのために作成されるbridgeが `--bip` で指定した範囲を無視してproxyのIPと競合してしまうためdocker-composeは使えなかった。

参考までにdocker-composeで作る場合のdocker-compose.ymlを記載する。

```
version: '2'

services:
  redmine:
    image: redmine
    ports:
      - 8888:3000
    environment:
      REDMINE_DB_MYSQL: db
      REDMINE_DB_PASSWORD: test123
      http_proxy: proxy.example.com:8080
    depends_on:
      - db
    volumes:
      - /mnt/redmine/plugins:/usr/src/redmine/plugins
    restart: always
  db:
    image: mariadb
    environment:
      MYSQL_DATABASE: redmine
      MYSQL_ROOT_PASSWORD: test123
      http_proxy: proxy.example.com:8080
```

## dbにテスト用のデータを挿入する

pluginのdb:migrateは自動で走らないようで次のように手動で実行しておく必要がある。

```
cd /usr/src/redmine/plugins/calls
rake redmine:plugins:migrate RAILS_ENV=production
```
参照： https://www.r-labs.org/projects/r-labs/wiki/%E3%83%97%E3%83%A9%E3%82%B0%E3%82%A4%E3%83%B3%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E6%96%B9%E6%B3%95
自動化するためにはDockerfileにdb:migrateを書いておくべき？

また現状はDBに入れておくべき値を設定ページなどから更新できないので、SQLを直接実行してデータを準備しておく必要があった。

```
■escalation_rules

MariaDB [redmine]> insert into escalation_rules (timeout, priority) values (2, 1);
Query OK, 1 row affected (0.00 sec)

■escalation_user

MariaDB [redmine]> insert into escalation_users (name, phone_number, priority) values ('kato.yudai', '*******', 1);
Query OK, 1 row affected (0.00 sec)

■ twilio_settings
insert into twilio_settings (twilio_phone_number, account_sid, auth_token, respons_url, wait_time) values ('*******', '*******', '*******', 'http://demo.twilio.com/docs/voice.xml', 10);
Query OK, 1 row affected (0.01 sec)
```
