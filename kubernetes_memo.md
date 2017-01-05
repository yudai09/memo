# kubernetesメモ

学習
- [x] Kubernetesの公式チュートリアルをこなす
 - http://kubernetes.io/docs/tutorials/kubernetes-basics/
 Stateful Applicationsとか以外は読んだ。
- [ ] Kubernetesアドベントカレンダーを読み切る
 - http://qiita.com/advent-calendar/2014/kubernetes
- [ ] Kubernetesの関門たるネットワーク周りについて理解する
 - Kubernetes Networking
  - http://www.slideshare.net/CJCullen/kubernetes-networking-55835829
  - http://kubernetes.io/docs/admin/networking/#kubernetes-model
  - kubernetesのNetworkingで重視されているのはコンテナ超えをするときにNAT/SNATをしないこと。ポートフォワーディングをするとデメリットが多い。
  - kube-proxyによってserviceが実現されている。kube-proxyはiptablesで実装されている。
  - flannelを使えばnode越えができるようだ。
 - [ ] kube-proxyとflannelの関係を理解する
  - [ ] 構築を手元でやる
   - http://blog.shippable.com/multi-node-kubernetes-cluster
   - http://blog.shippable.com/docker-overlay-network-using-flannel
   - http://blog.shippable.com/kubernetes-cluster-with-flannel-overlay-network
  - https://devops.profitbricks.com/tutorials/deploy-a-multi-node-kubernetes-cluster-on-centos-7/
  - flannelはノード越えするときに必要。ノードが一つだけの場合は不要。
  ```
  Note: Flannel, or another network overlay service, is required to run on the minions when there is more than one minion host. This allows the containers which are typically on their own internal subnet to communicate across multiple hosts. As the Kubernetes master is not typically running containers, the Flannel service is not required to run on the master.
  ```
  - http://www.atmarkit.co.jp/ait/articles/1510/23/news016_2.html
   - flannelはdockerにdocker0に割当てるべきネットワーク帯をおしえてくれる。
   - どのホストにネットワークをルーティングするべきかおしえてくれる。
   - ![image](http://image.itmedia.co.jp/ait/articles/1510/23/docker_manage3_3.jpg)
 - [ ] skydnsについて理解する（導入は任意だが必要になりそうな感じ）
  - https://www.consul.io/intro/vs/skydns.html
 - [ ] external servicesの章がよくわからないので調べる。
  - ingressについて調べる
   - http://kubernetes.io/docs/user-guide/ingress/
   - 冒頭だけよんだ。外部ネットワークをkuberntesクラスタへ引き込むためのものっぽい。
- [ ] GCPを試す
https://cloud.google.com/container-engine/
- [ ] Docker実践ガイドを読みとく
 - Kubernetesが何を隠蔽しているのかを正確に理解するため。
- [ ] Docker Swarmを理解する。
 - https://docs.docker.com/engine/userguide/networking/get-started-overlay/

---
用語整理

nodeはminionと同じ

http://kubernetes.io/docs/admin/node/

```
What is a node?

A node is a worker machine in Kubernetes, previously known as a minion. A node may be a VM or physical machine, depending on the cluster.
```
