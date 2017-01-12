# docker swarmを試す

## 第三回

以下の構成要素の関係性等について調べる。
etcd/ingress/embedded dns

よくわかっていないこと

* etcdがどこにもいないように見えるけど、一体どこにいるのか
* ingressとetcdは連携していると思っているんだけど、どう連携するのか設定はどうなっているのか。
* embedded dnsでサービス名を参照できると書いてあるけどどうやって確認するのか、また仕組みはどうなっているのか。これもetcdがバックエンドにいるのか？

以下を読み進めることでわかった。
https://leanpub.com/the-devops-2-1-toolkit

* etcdがどこにもいないように見えるけど、一体どこにいるのか
  -> 内部に同等の機能を抱えてしまったようだ。
* ingressとetcdは連携していると思っているんだけど、どう連携するのか設定はどうなっているのか。
  -> そのあたりは深堀しないと出てこないので今後の宿題にする。
* embedded dnsでサービス名を参照できると書いてあるけどどうやって確認するのか、また仕組みはどうなっているのか。これもetcdがバックエンドにいるのか？
  -> 内部の分散KVSがいる。仕組みはわからないけどコンテナから名前解決すると仮想IPが引ける。
  
### etc

##### manager, workerの数

As a general rule, we should have a least three Swarm managers. That way, if one of them fails, the
others will reschedule the failed containers and can be used as our access points to the system. As is
often the case with solutions that require a quorum, an odd number is usually the best. Hence, we
have three.

Swarm managerは最低で3台必要。奇数がベスト。

#### 外部のKVSを利用する。

etcd, consulなどを分散KVSとして普通に使うのもあり。
