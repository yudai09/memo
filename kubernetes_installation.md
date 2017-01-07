# kubernetesをインストール

参照: http://knowledge.sakura.ad.jp/tech/3681/

## master node

### firewalldの停止

あとで警告が出るので止めておく。
```
# systemctl disable firewalld && systemctl stop firewalld
```

### selinuxの停止

```
# sed -i 's|^SELINUX=.*$|SELINUX=permissive|' /etc/sysconfig/selinux
# reboot
```

### /etc/hosts

```
cat >> /etc/hosts << EOF
172.28.128.11 host1
172.28.128.12 host2
172.28.128.13 host3
EOF
```

### etcd, kubernetes, flannel install

```
# yum install etcd kubernetes flannel -y
```
### etcd

```
# sed -i.orig 's|ETCD_LISTEN_CLIENT_URLS=.*|ETCD_LISTEN_CLIENT_URLS="http://host1:2379,http://localhost:2379"|' /etc/etcd/etcd.conf
# systemctl start etcd
```

### flanneld

```
virtualboxのhost-only-networksはeth1なので、そちらから通信させるようにする。
# sed -i.orig 's|FLANNEL_OPTIONS=.*|FLANNEL_OPTIONS="--iface=eth1"|' /etc/sysconfig/flanneld

# grep FLANNEL_ETCD_PREFIX /etc/sysconfig/flanneld
FLANNEL_ETCD_PREFIX="/atomic.io/network"

# etcdctl mk /atomic.io/network/config '{"Network":"10.244.0.0/16"}'
# systemctl start flanneld
```

### 鍵作成

```
# openssl genrsa -out /etc/kubernetes/serviceaccount.key 2048
```

### kubernetes install

```
# sed -i.orig 's|KUBE_API_ARGS=.*|KUBE_API_ARGS="--service_account_key_file=/etc/kubernetes/serviceaccount.key"|' /etc/kubernetes/apiserver
# sed -i 's|KUBE_API_ADDRESS=.*|KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"|' /etc/kubernetes/apiserver
# sed -i.orig 's|KUBE_CONTROLLER_MANAGER_ARGS=.*|KUBE_CONTROLLER_MANAGER_ARGS="--service_account_private_key_file=/etc/kubernetes/serviceaccount.key"|' /etc/kubernetes/controller-manager

```

```
# systemctl start kube-apiserver
# systemctl start kube-controller-manager
# systemctl start kube-scheduler
# systemctl start kube-proxy
```

### kubectl setup

rootではない他のユーザで実行
```
$ kubectl config set-credentials myself --username=admin --password=amdin
$ kubectl config set-cluster local-server --server=http://host1:8080
$ kubectl config set-context default-context --cluster=local-server --user=myself
$ kubectl config use-context default-context
$ kubectl config set contexts.default-context.namespace default
```

## slave node

### firewalldの停止

あとで警告が出るので止めておく。
```
# systemctl disable firewalld && systemctl stop firewalld
```

### selinuxの停止

```
# sed -i 's|^SELINUX=.*$|SELINUX=permissive|' /etc/sysconfig/selinux
# reboot
```

### /etc/hosts

```
# cat >> /etc/hosts << EOF
172.28.128.11 host1
172.28.128.12 host2
172.28.128.13 host3
EOF
```

## docker, kubernetes, flannel install

```
# yum install docker kubernetes flannel -y
```

### flannel setup

```
# sed -i.orig 's|FLANNEL_ETCD_ENDPOINTS=.*|FLANNEL_ETCD_ENDPOINTS=http://host1:2379|' /etc/sysconfig/flanneld
# sed -i 's|FLANNEL_OPTIONS=.*|FLANNEL_OPTIONS="--iface=eth1"|' /etc/sysconfig/flanneld
# systemctl start flanneld
```

overlay networkが作成されたことを確認する。

```
# ip a l

4: flannel0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1472 qdisc pfifo_fast state UNKNOWN qlen 500
    link/none
    inet 10.244.52.0/16 scope global flannel0
       valid_lft forever preferred_lft forever

```

### docker start

```
# systemctl start docker
```

### kubernetesの設定

```
# sed -i.orig 's|KUBE_MASTER=.*|KUBE_MASTER="--master=http://host1:8080"|' /etc/kubernetes/config

# sed -i.orig 's|KUBELET_ADDRESS=.*|KUBELET_ADDRESS="--address=172.28.128.12"|' /etc/kubernetes/kubelet
# sed -i 's|KUBELET_HOSTNAME=.*|KUBELET_HOSTNAME="--hostname_override="|' /etc/kubernetes/kubelet
# sed -i 's|KUBELET_API_SERVER=.*|KUBELET_API_SERVER="--api-servers=http://host1:8080"|' /etc/kubernetes/kubelet

# systemctl start kube-proxy
# systemctl start kubelet
```

### kubectrl

rootではない他のユーザで実行
```
$ kubectl config set-credentials myself --username=admin --password=amdin
$ kubectl config set-cluster local-server --server=http://host1:8080
$ kubectl config set-context default-context --cluster=local-server --user=myself
$ kubectl config use-context default-context
$ kubectl config set contexts.default-context.namespace default
```

### nodeが追加されたか確認

```
# kubectl get nodes
```


### 宿題

serviceを作ったけど、他のノードからどうやってアクセスすればよいのかわからない。
あと `kubectl exec POD` が使えない理由も不明。 `kube-dns` が必要なんだろうか。


# 以下ボツ

kubeadmでインストール
http://kubernetes.io/docs/getting-started-guides/kubeadm/

## masterの構築

### (1/4) Installing kubelet and kubeadm on your hosts

説明の通り実行
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet
```

### (2/4) Initializing your master

flannelを使う場合は`--pod-network-cider`を付与せよとのことだったので、付与した。

```
[root@host1 ~]# kubeadm init --pod-network-cidr=10.244.0.0/16
[kubeadm] WARNING: kubeadm is in alpha, please do not use it for production clusters.
[preflight] Running pre-flight checks
[preflight] Starting the kubelet service
[init] Using Kubernetes version: v1.5.1
[tokens] Generated token: "8c6032.9c346c45b50e519c"
[certificates] Generated Certificate Authority key and certificate.
[certificates] Generated API Server key and certificate
[certificates] Generated Service Account signing keys
[certificates] Created keys and certificates in "/etc/kubernetes/pki"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/admin.conf"
[apiclient] Created API client, waiting for the control plane to become ready
[apiclient] All control plane components are healthy after 20.531908 seconds
[apiclient] Waiting for at least one node to register and become ready
[apiclient] First node is ready after 2.501989 seconds
[apiclient] Creating a test deployment
[apiclient] Test deployment succeeded
[token-discovery] Created the kube-discovery deployment, waiting for it to become ready
[token-discovery] kube-discovery is ready after 2.502074 seconds
[addons] Created essential addon: kube-proxy
[addons] Created essential addon: kube-dns

Your Kubernetes master has initialized successfully!

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
    http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node:

kubeadm join --token=8c6032.9c346c45b50e519c 10.0.2.15
```
kubeadmはαリリースらしいので、本番では使うなということらしい。
ノードを追加する場合は一番下のトークンを使うので、メモっておけとのこと。
中で何をやったのかはkubeadmなしで構築しないとわからないな。

セキュリティ上の理由でmasterにはpodは配置しないらしい。

### (3/4) Installing a pod network

以下の選択肢があるが、とりあえずFlannelがスタンダートっぽいのでいれる。
```
Calico is a secure L3 networking and network policy provider.
Canal unites Flannel and Calico, providing networking and network policy.
Flannel is an overlay network provider that can be used with Kubernetes.
Romana is a Layer 3 networking solution for pod networks that also supports the NetworkPolicy API. Kubeadm add-on installation details available here.
Weave Net provides networking and network policy, will carry on working on both sides of a network partition, and does not require an external database
```

```
[root@host1 ~]# curl -L https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml > /tmp/kube-flannel.yml
[root@host1 ~]# kubectl apply -f /tmp/kube-flannel.yml
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
[root@host1 ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE
kube-system   dummy-2088944543-mlvxr            1/1       Running   0          14m
kube-system   etcd-host1                        1/1       Running   0          14m
kube-system   kube-apiserver-host1              1/1       Running   0          14m
kube-system   kube-controller-manager-host1     1/1       Running   0          14m
kube-system   kube-discovery-1769846148-ss6w3   1/1       Running   0          14m
kube-system   kube-dns-2924299975-sg6hf         0/4       Pending   0          14m
kube-system   kube-flannel-ds-xk6jv             2/2       Running   0          1m
kube-system   kube-proxy-pgftd                  1/1       Running   0          14m
kube-system   kube-scheduler-host1              1/1       Running   0          14m
```
なんかダメだ。flannelが入っていないからかな。kube-dnsが起動しない。
多分 flannel -> kube-flannelというつながりになるからあらかじめflannelを入れておく必要がある。

install方法はサクラのナレッジを参考にした。なんとyumで入る。
http://knowledge.sakura.ad.jp/tech/3681/
flannelはetcdと一緒に動くのでetcdもインストールしておく。

```
[root@host1 ~]# yum install flannel etcd
[root@host1 ~]# systemctl start flanneld && systemctl enable flanneld
```

なんか止まった。そしてタイムアウトで起動しなかった。
以下のようなエラーが出ている。
```
Failed to retrieve network config: 100: Key not found (/atomic.io) [2503]
```

```
[root@host1 ~]# cat /etc/sysconfig/flanneld
# Flanneld configuration options

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="http://127.0.0.1:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/atomic.io/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""

[root@host1 ~]# etcdctl mk /atomic.io/network/config '{"Network":"10.244.0.0/16"}'
```

みんな/coreos.com/network/configに設定しているが、なんか違った。atomic.io

あらためてflannelを起動すると問題なく起動した。

だがそれでもうまく行かない。
とりあえずホストを再起動したらもっとひどい感じになったので、また明日頑張る。

# 2016-01-06

Vagrantでホスト再作成を自動化しておきリトライ

よくわからんけど、Kubeadmはetcdをコンテナとして作成するからつなげ方が難しいとかなのか？？
有無よくわからんけどkubeadmはいろいろ隠蔽してしまってよくわからないので他の方法でやることにする。
