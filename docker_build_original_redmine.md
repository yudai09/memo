# Redmineコンテナに添付ファイルをS3に突っ込むプラグインを入れる。

カスタマイズ前の素のRedmineは以下
https://github.com/docker-library/redmine

このDockerfileとentrypoint.shを変更して以下の添付ファイルをS3に保存するプラグインを入れる。
https://github.com/ka8725/redmine_s3

## カスタマイズする前に普通にビルドしてrunできるか確認する

```
$ git clone https://github.com/docker-library/redmine.git                                                                                 
Cloning into 'redmine'...
remote: Counting objects: 467, done.                                                                                                                                
remote: Total 467 (delta 0), reused 0 (delta 0), pack-reused 467                                                                                                    
Receiving objects: 100% (467/467), 65.23 KiB | 0 bytes/s, done.
Resolving deltas: 100% (250/250), done.
[sci01436@dockerhost001 ~]$ cd redmine/
.git/ 3.1/  3.2/  3.3/  
$ cd redmine/3.3/                                                                                                                         
$ ls                                                                                                                                    
Dockerfile  docker-entrypoint.sh  passenger
$ docker build -f ./Dockerfile -t redmine:custom --no-cache=true .
(途中結果割愛）
Removing intermediate container b6ce4f0e6bb6
Successfully built 14c2abc21e6e
```

buildできたのでrunしてみる

```
$ docker run -d --name redmine -it -p 8888:3000 redmine:custom
13c6f8572b383f12ffafd18601503170ee49b746e0bf6e03ae870405ec19b62f
$ docker ps                                                                                                                             
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
13c6f8572b38        redmine:custom         "/docker-entrypoint.s"   23 seconds ago      Up 21 seconds       0.0.0.0:8888->3000/tcp   redmine
$ docker rm -f 13c6f8572b38
```

runも正常に行われたので、build/runの手順が確立できた。

## カスタマイズ

ここから本題。
変更部分の再ビルドは `--no-cache=false` とすれば高速にできる。

インストール手順はプラグインのgithubに記載されたものを参照した。

```
$ vi s3.yml
production:
  access_key_id: *****
  secret_access_key: *****
  bucket: *****
  folder: files
  endpoint: jp-east-2.os.cloud.nifty.com
  secure: true
  private: true
  expires:
  proxy: false
  thumb_folder:
$ vi Dockerfile
以下を追加
COPY s3.yml config/s3.yml
$ vi docker-entrypoint.sh
末尾に以下を追加
                apt-get install git -y
                cd /usr/src/redmine
                git clone git://github.com/ka8725/redmine_s3.git plugins/redmine_s3
                bundle install --without development test
                rm -Rf plugins/redmine_s3/.git
$ docker build -f ./Dockerfile -t redmine:custom --no-cache=false .
$ docker run -d --name redmine -it -p 8888:3000 redmine:custom
```

動いた！

ついでに `dockerswarm_redmine.md` でやったDocker swarm版にも適用してみよう。

```
$ docker service create --name sv_redmine_custom \
--reserve-memory 100m \
--replicas 1 \
--network nw_galera \
-e 'REDMINE_DB_MYSQL=sv_galera' \
-e 'REDMINE_DB_USERNAME=redmine' \
-e 'REDMINE_DB_PASSWORD=test123' \
-e 'REDMINE_DB_DATABASE=redmine' \
-e 'REDMINE_DB_ENCODING=utf8mb4' \
-p 8888:3000 \
redmine:custom
```

問題なく動作することが確認できた。
添付ファイルをチケットに登録すると、S3ストレージにオブジェクトが作成される。すばらしい。
