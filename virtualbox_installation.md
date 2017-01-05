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
