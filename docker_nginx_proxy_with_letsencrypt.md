# 無料のSSL証明書を使う

# やりたいこと

* SSL証明書をつかってRedmineに対するアクセスを暗号化する

# 背景

普通のSSL証明書を買うと結構高い。
SSL証明書は年間数万の費用がかかる。特に利益を生むことのない内部用の管理画面を運用する中でSSLを使いたいときに費用がネックになることがある。

実は無料の証明書というものが世の中には存在している。
以下のまとめサイトにあるが、証明書のレベルは一番低いドメインに認証（ドメイン認証＜企業実在認証＜EV）にあたるらしい。

https://jp.globalsign.com/blog/2016/freessltls_letsencrypt.html

したがって顧客に利用されるサービスにおいて使うのは微妙かもしれないが、内部システムにつかうのには十分だろう。

[letsencrypt](https://letsencrypt.jp/)で今回は証明書を取得してみたいと思う。

使用方法はこちらを参考にする。

https://letsencrypt.jp/usage/

# ざっくりシステム概要

* `certbot` 実行用のコンテナを使って証明書を作成
* ニフティクラウドAPI実行用のコンテナを作成し、API経由で証明書のアップロード

# certbot

`certbot` をLinux上に導入して証明書を発行することができるらしい。
dockerコンテナとしてcertbotを実行できるようなので、この方法が自分の環境においては利便性が高い。

http://qiita.com/setouchi/items/b3a78cb396ae6e58b84b

[dockerfile](https://github.com/certbot/certbot/blob/master/Dockerfile)

# certbotやってみる

```
dockerhost001 $ sudo mkdir -p /usr/local/share/letsencrypt/etc/
dockerhost001 $ sudo mkdir -p /usr/local/share/letsencrypt/var/

dockerhost001 $ docker run -it --rm -p 443:443 --name letsencrypt -v "/usr/local/share/letsencrypt/etc/:/etc/letsencrypt" -v "/usr/local/share/letsencrypt/var/:/var/lib/letsencrypt" quay.io/letsencrypt/letsencrypt:latest certonly
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Failed to find apache2ctl in PATH: /opt/certbot/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

How would you like to authenticate with the ACME CA?
-------------------------------------------------------------------------------
1: Place files in webroot directory (webroot)
2: Spin up a temporary webserver (standalone)
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel):redmine.grandeur09.com
Obtaining a new certificate
Performing the following challenges:
tls-sni-01 challenge for redmine.grandeur09.com
Waiting for verification...
Cleaning up challenges
Generating key (2048 bits): /etc/letsencrypt/keys/0000_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0000_csr-certbot.pem

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/redmine.grandeur09.com/fullchain.pem. Your
   cert will expire on 2017-05-03. To obtain a new or tweaked version
   of this certificate in the future, simply run certbot again. To
   non-interactively renew *all* of your certificates, run "certbot
   renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

dockerhost001 $ sudo tree /usr/local/share/letsencrypt/
/usr/local/share/letsencrypt/
├── etc
│   ├── accounts
│   │   └── acme-v01.api.letsencrypt.org
│   │       └── directory
│   │           └── b079e1eae9586497b54acc12f8961203
│   │               ├── meta.json
│   │               ├── private_key.json
│   │               └── regr.json
│   ├── archive
│   │   └── redmine.grandeur09.com
│   │       ├── cert1.pem
│   │       ├── chain1.pem
│   │       ├── fullchain1.pem
│   │       └── privkey1.pem
│   ├── csr
│   │   └── 0000_csr-certbot.pem
│   ├── keys
│   │   └── 0000_key-certbot.pem
│   ├── live
│   │   └── redmine.grandeur09.com
│   │       ├── README
│   │       ├── cert.pem -> ../../archive/redmine.grandeur09.com/cert1.pem
│   │       ├── chain.pem -> ../../archive/redmine.grandeur09.com/chain1.pem
│   │       ├── fullchain.pem -> ../../archive/redmine.grandeur09.com/fullchain1.pem
│   │       └── privkey.pem -> ../../archive/redmine.grandeur09.com/privkey1.pem
│   └── renewal
│       └── redmine.grandeur09.com.conf
└── var
    └── backups
```
443ポートには何もまだバインドしていないし、内部用のシステムなのでポート番号はずらす予定で443はcertbotが認証するための専用にできる。

そのため、certbotをホストの443にバインドしてstandalone（その場だけのWebサーバを立てる）で認証した。

とりあえずファイルがいっぱいでてきた。

とりあえず `live/` の下のものを使えば良いっぽい。

# 証明書を搭載したnginxリバースプロキシーをつくる

nginxのreverse proxyを作り、そこでSSLを復号させることにする。

nginxの設定はいかが参考になる。

http://webos-goodies.jp/archives/ssl_proxy_using_nginx.html

探していたところ、nginxとletsencryptを組み合わせたコンテナを発見した。

https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion

これ使えばいけるのだろうか。以下が気になる。なぜ `docker.sock` が必要なのか？
```
-v /var/run/docker.sock:/var/run/docker.sock:ro \
```
調べてみると `docker-gen` と連携してDockerに発生した変更を検知しているらしい。
そこまでいらないんだけどな。

以下にそれなりにletsencrypt + nginx + dockerについてまとまっているので、それを読む

https://hackerslog.net/post/my-tech/server/letsencrypt-on-docker-and-nginx/

自動更新か・・・ごくり。
Docker composeってSwarmで使えるんだっけ？

http://docs.docker.jp/compose/swarm.html
```
Docker Compose と Docker Swarm は完全な統合を目指しています。つまり、Compose アプリケーションを Swarm クラスタに適用したら、単一の Docker ホスト上で展開するのと同じように動作します。
```

あと `-v --volume` もSwarmで使えるのか？どのホストで起動するかわからないコンテナが参照するボリュームってNFSで全ホストから閲覧可能にしないと動かないのではないかと懸念。

http://qiita.com/atsaki/items/4f1e5bd2df1dbe86a517
```
他のコンテナのボリュームをマウント

他のコンテナのボリュームをマウントすることもできます。
```

できるらしい。本当か？

https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion/issues/145
やはりボリュームの関係で動かないようだな。

証明書をCOPYで持っていくのが現実的な気がする。

# ここから検証

```
$ pwd
/home/sci01436/nginx_redmine_proxy
$ cat Dockerfile                                                                                                        
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf

$ cat nginx.conf                                                                                                        
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    server {
        listen 8080;
        server_name redmine.grandeur09.com;
        location / {
            proxy_pass http://sv_redmine_custom:3000;
        }
    }
}
$ docker build -f ./Dockerfile -t proxy_redmine --no-cache=true .

$ docker service create --name sv_proxy_redmine \
--reserve-memory 100m \
--mode global \
--network nw_galera \
-p 8080:8080 \
proxy_redmine
```

いけた。
さて証明書を載せてSSLにする。

```
証明書をDockerfileと同じディレクトリに固めて置く
$  sudo tar zcvf redmine.grandeur09.com.crt.tar.gz -h -C  /usr/local/share/letsencrypt/etc/live/ redmine.grandeur09.com/

$ tree
.
├── Dockerfile
├── nginx.conf
└── redmine.grandeur09.com.crt.tar.gz

$ cat Dockerfile                                                                                                        
FROM nginx
COPY nginx.conf /etc/nginx/nginx.conf
ADD redmine.grandeur09.com.crt.tar.gz /usr/local/nginx/conf/cert/

$ cat nginx.conf
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    server {
        listen 443 default ssl;
        ssl on;
        ssl_certificate /usr/local/nginx/conf/cert/redmine.grandeur09.com/cert.pem;
        ssl_certificate_key /usr/local/nginx/conf/cert/redmine.grandeur09.com/privkey.pem;
        location / {
            proxy_pass http://sv_redmine_custom:3000;
        }
    }
}

リビルド
$ docker build -f ./Dockerfile -t proxy_redmine --no-cache=true .

serviceを消してから作り直し

$ docker service rm sv_proxy_redmine
$ docker service create --name sv_proxy_redmine --reserve-memory 100m --mode global --network nw_galera -p 8443:443 -p 8080:8080 proxy_redmine
```

これで無事作ることができた。

残課題として、証明書の更新を自動化する必要がある。
