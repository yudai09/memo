# kubernetesをインストール

kubeadmでインストール
http://kubernetes.io/docs/getting-started-guides/kubeadm/

## masterの構築

### (1/4) Installing kubelet and kubeadm on your hosts

説明の通り実行

### firewalldの停止

あとで警告が出るので止めておく。
```
[root@host1 ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
Removed symlink /etc/systemd/system/basic.target.wants/firewalld.service.
[root@host1 ~]# systemctl stop firewalld
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
