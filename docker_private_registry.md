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

config.ymlの作り方は以下に詳細が書いてある
https://github.com/docker/distribution/blob/8e065ad239542a91962f3777b798c84d4e017baa/docs/configuration.md
-> パラメタの書き換えは環境変数で指定可能らしいが、今回は設定を直接書き換えることにする。

まずはDockerfileを以下のように書き換える。

Dockerfile
```
-COPY cmd/registry/config-dev.yml /etc/docker/registry/config.yml
+COPY cmd/registry/config.yml /etc/docker/registry/config.yml
```

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
http:
    addr: :5000
    debug:
        addr: localhost:5001
```
ニフティクラウドのオブジェクトストレージはv4authに非対応なので、noとしている。

```
docker build -t registry .
docker run -d -p 5000:5000 --name registry registry
```

試しに以下のようにregistry自体をregistryに入れてみる。
```
docker tag registry localhost:5000/registry
docker push localhost:5000/registry
```

それなりに時間はかかるが成功した。
