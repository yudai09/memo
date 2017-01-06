# virtualboxでCentOS7をぽこぽこ立ち上げるためのメモ

Virtualboxはどっかで適当に入れていたようではいっていた。
設定がコツがあったのでメモる。

## SSHでＶＭに入れるようにする。
まずはvirtualbox上のVMにSSHでアクセスするためにHost-Only-AdapterをVirtualboxの設定から作成して、ＶＭの設定からそのアダプタを２個目のネットワークとして追加する。（1個めは自動的にNAT用のものが追加されている状態になっている。）
VMをクローニングする場合は設定でMACアドレスを変更しないとネットワーク的に不都合が生じるので注意。VirtualboxのVMの設定から変更可能。

## インターネットにアクセスできるようにする。
なぜかNATだとインターネットにアクセスできなかった。

```
# nmtui

-> activate a connection
  enp0s3をactivateすれば通信できるようになる。
```

# vagrantでやりたいな
[参考Qiita](http://qiita.com/ozawan/items/160728f7c6b10c73b97e)
こっちのほうが綺麗にポンポン立ち上げられて良いと思う。

上でやったようなこと(MACの変更、ethの有効化)とホスト名付与等の初期構築まで自動的にやりたい。VM内の変更も可能っぽい。[参考Qiita](http://qiita.com/murachi1208/items/00c3c2fe51763a6535f8)
ホスト名はランダムにしちゃってもいいと思う。いちいち書き換えるのがめんどくさいので。

# vagrantでやってみる。

Vagrantの公式サイトから最新のdebファイルを持ってきてインストール。
そして以下を実行するだけでCentOS7のboxを持ってきて起動できる。

```
vagrant init centos/7; vagrant up --provider virtualbox
```

boxはそれなりのデータサイズなのでダウンロードに時間がかかる。
とりあえず1つのＶＭが起動することが確認できた。
設定不足なので一旦捨てるために `vagrant destroy` しておく。

続いて`vagrant init` で生成されるVagrantfileをカスタマイズする。

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", type: "dhcp"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
  end
  config.vm.define "server1" do |server1|
    server1.vm.hostname = "server1"
  end

  config.vm.define "server2" do |server2|
    server2.vm.hostname = "server2"
  end
end
```

これで2台のＶＭが生成される。

```
$ vagrant up --provider virtualbox
```

ログインするには以下のようにすると、それぞれのサーバにログインできる。

```
$ vagrant ssh server1
$ vagrant ssh server2
```
