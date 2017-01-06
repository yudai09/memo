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
