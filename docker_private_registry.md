# private registryを作る

## 概要

* Privateなコンテナイメージを保管しておくregistryを構築する。
* バックエンド（格納先）はS3互換のストレージとする。

## 参照

* [Docker公式](https://docs.docker.com/registry/)
* [github: Distribution>registry](https://github.com/docker/distribution/tree/master/registry)
* [Qiita:開発版 Docker & Docker Registry の検証環境を作って試してみる](http://qiita.com/dtan4/items/0e34a09923579e7a6670)

docker-registryはDistributionという名前のパッケージに統合されたらしい。

## インストール

Qiitaの記事同様にDistributionをgit cloneで持ってきて、Dockerfileからビルドして使うことにする。

config.ymlの作り方は以下に詳細な記載がある。
https://github.com/docker/distribution/blob/8e065ad239542a91962f3777b798c84d4e017baa/docs/configuration.md
-> パラメタの書き換えは環境変数で指定可能らしいが、今回は設定を直接書き換えることにする。

まずはDockerfileを以下のように書き換える。

Dockerfile
```
-COPY cmd/registry/config-dev.yml /etc/docker/registry/config.yml
+COPY cmd/registry/config.yml /etc/docker/registry/config.yml
```
`-dev` のままだとかっこ悪いので。

config.ymlを以下のようにする
```
version: 0.1
log:
  level: info
  fields:
    service: registry
    environment: service
storage:
    delete:
      enabled: true
    s3:
      accesskey: ****
      secretkey: ****
      region: east-1
      regionendpoint: jp-east-2.os.cloud.nifty.com
      secure: true
      bucket: ****
      v4auth: no
    redirect:
      disable: yes
http:
    addr: :5000
    debug:
        addr: localhost:5001
```
* ニフティクラウドのオブジェクトストレージはv4authに非対応なので、noとしている。対応予定らしいけど。
* redirectは無効にしないと動作しなかった。
  redirectが有効だとblobのアップロードするときにDockerがregistryを介さずにオブジェクトストレージに直接アクセスするのだが、試してみると直接アクセスするために生成されるURLがでたらめなURLになっていた。

以下で起動する。
あとで述べるようにこの起動方法ではDockerからイメージをPUSHできない。
SSLを有効にするか、SSLを不要にするためにDockerを設定する必要がある（非推奨）

```
$ docker build -t registry .
$ docker run -d -p 5000:5000 --name registry registry
```

試しに以下のようにregistry自体をregistryに入れてみる。
```
$ docker tag registry localhost:5000/registry
$ docker push localhost:5000/registry
```
問題なくregistryに登録された。

最後にDNS名を与える。
`registry.grandeur09.com`

タグを付け直して再度push
```
$ docker tag localhost:5000/registry registry.grandeur09.com:5000/registry
$ docker push registry.grandeur09.com:5000/registry
```

しかしpushが上手くいかない。
```
$ docker push registry.grandeur09.com:5000/registry                                                  
The push refers to a repository [registry.grandeur09.com:5000/registry]
Get https://registry.grandeur09.com:5000/v1/_ping: http: server gave HTTP response to HTTPS client
```

HTTPSでの通信が必要なようだ。

## TLSを有効にする

registry.grandeur09.comの証明書を用意する。
letsencryptを使って作成したものを利用することにする。
letsencryptについては `docker_nginx_proxy_with_letsencrypt.md` に書いたのでここでは割愛する。

```
$ docker run -it --rm -p 443:443 --name letsencrypt \
-v "/usr/local/share/letsencrypt/etc/:/etc/letsencrypt" \
-v "/usr/local/share/letsencrypt/var/:/var/lib/letsencrypt" \
quay.io/letsencrypt/letsencrypt:latest certonly

$ docker run -d -p 5000:5000 --restart=always --name dockerregistry \
  -v "/usr/local/share/letsencrypt/etc/:/certs"\
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/live/registry.grandeur09.com/fullchain.pem \
  -e REGISTRY_HTTP_TLS_KEY=/certs/live/registry.grandeur09.com/privkey.pem \
  registry
```

これでうまくいくようになった。

TLSを回避する方法は以下に説明があるが、推奨されない（当たり前だけど）
https://docs.docker.com/registry/insecure/
