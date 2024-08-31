# Kubernetes证书续期

## 方法一：手动更新证书

由 kubeadm 生成的客户端证书默认只有一年有效期，我们可以通过 `check-expiration` 命令来检查证书是否过期：

```shell
kubeadm certs check-expiration
```

该命令显示 `/etc/kubernetes/pki` 文件夹中的客户端证书以及 kubeadm 使用的 `KUBECONFIG` 文件中嵌入的客户端证书的到期时间/剩余时间。

**注意：**`kubeadm` 不能管理由外部 CA 签名的证书，如果是外部的证书，需要自己手动去管理证书的更新。

另外需要说明的是上面的列表中没有包含 `kubelet.conf`，因为 kubeadm 将 kubelet 配置为自动更新证书。

另外 kubeadm 会在控制面板升级的时候自动更新所有证书，所以使用 kubeadm 搭建的集群最佳的做法是经常升级集群，这样可以确保你的集群保持最新状态并保持合理的安全性。但是对于实际的生产环境我们可能并不会去频繁的升级集群，所以这个时候我们就需要去手动更新证书。

要手动更新证书也非常方便，我们只需要通过 `kubeadm certs renew` 命令即可更新你的证书，这个命令用 CA（或者 front-proxy-CA ）证书和存储在 `/etc/kubernetes/pki` 中的密钥执行更新。

**注意：**如果你运行了一个高可用的集群，这个命令需要在所有控制面板节点上执行。

接下来我们来更新我们的集群证书，下面的操作都是在 master 节点上进行，首先备份原有证书：

```shell
cp -r /etc/kubernetes/ /etc/kubernetes_0922bak
```

然后备份 etcd 数据目录：

```shell
cp -r /var/lib/etcd /var/lib/etcd_0922bak
```

接下来执行更新证书的命令：

```shell
kubeadm certs renew all --config=kubeadm.yaml
```

通过上面的命令证书就一键更新完成了，这个时候查看上面的证书可以看到过期时间已经是一年后的时间了.

kubeadm.yaml可通过如下命令获得：

```shell
kubectl get cm -o yaml -n kube-system kubeadm-config
```

然后记得更新下 kubeconfig 文件：

```yaml
kubeadm init phase kubeconfig all --config kubeadm.yaml
```

将新生成的 admin 配置文件覆盖掉原本的 admin 文件:

```shell
mv $HOME/.kube/config $HOME/.kube/config.old
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

完成后重启 kube-apiserver、kube-controller、kube-scheduler、etcd 这4个容器即可，我们可以查看 apiserver 的证书的有效期来验证是否更新成功：

```shell
echo | openssl s_client -showcerts -connect 127.0.0.1:6443 -servername api 2>/dev/null | openssl x509 -noout -enddate
```

可以看到现在的有效期是一年过后的，证明已经更新成功了。

此方法可将证书更新，但有效期只有一年，如需将证书一次性更新为十年有效期，请查看方法二。

## 方法二：脚本更新证书

**该脚本用于处理已过期或者即将过期的 kubernetes 集群证书**

**该脚本适用于所有 k8s 版本集群证书更新(使用 kubeadm 初始化的集群)**

kubeadm 生成的证书有效期为 1 年，该脚本可将 kubeadm 生成的证书有效期更新为 10 年

该脚本只处理 master 节点上的证书，node 节点的 kubelet 证书默认自动轮换更新，无需关心过期问题，只需关心 master 节点上的证书即可

### 1. 使用说明

**该脚本仅需要在 master 节点执行，无需在 node 节点执行**

- 若没有 etcd 相关证书，只需要更新 master 证书即可，[见这里](https://github.com/yuyicai/update-kube-cert/blob/master/other.md#1-只更新-master-证书)（小于等于 `v1.9` 版本，etcd 默认不使用 TLS 连接）
- 默认情况按照下面步骤进行证书更新

#### 1.1. 拉取脚本

```
git clone https://github.com/yuyicai/update-kube-cert.git
cd update-kubeadm-cert
chmod 755 update-kubeadm-cert.sh
```

执行时请使用 `./update-kubeadm-cert.sh all` 或者 `bash update-kubeadm-cert.sh all` ，不要使用 `sh update-kubeadm-cert.sh all`，因为某些 Linux 发行版 sh 并不是链接到 bash，可能会不兼容

#### 1.2. 更新证书

如果有多个 master 节点，在每个 master 节点都执行一次

执行命令：

```
./update-kubeadm-cert.sh all
```

输出类似信息

```
CERTIFICATE                                       EXPIRES
/etc/kubernetes/controller-manager.config         Sep 12 08:38:56 2022 GMT
/etc/kubernetes/scheduler.config                  Sep 12 08:38:56 2022 GMT
/etc/kubernetes/admin.config                      Sep 12 08:38:56 2022 GMT
/etc/kubernetes/pki/ca.crt                        Sep 11 08:38:53 2031 GMT
/etc/kubernetes/pki/apiserver.crt                 Sep 12 08:38:54 2022 GMT
/etc/kubernetes/pki/apiserver-kubelet-client.crt  Sep 12 08:38:54 2022 GMT
/etc/kubernetes/pki/front-proxy-ca.crt            Sep 11 08:38:54 2031 GMT
/etc/kubernetes/pki/front-proxy-client.crt        Sep 12 08:38:54 2022 GMT
/etc/kubernetes/pki/etcd/ca.crt                   Sep 11 08:38:55 2031 GMT
/etc/kubernetes/pki/etcd/server.crt               Sep 12 08:38:55 2022 GMT
/etc/kubernetes/pki/etcd/peer.crt                 Sep 12 08:38:55 2022 GMT
/etc/kubernetes/pki/etcd/healthcheck-client.crt   Sep 12 08:38:55 2022 GMT
/etc/kubernetes/pki/apiserver-etcd-client.crt     Sep 12 08:38:56 2022 GMT
[2021-09-12T16:41:25.93+0800][INFO] backup /etc/kubernetes to /etc/kubernetes.old-20210912
[2021-09-12T16:41:25.93+0800][INFO] updating...
[2021-09-12T16:41:25.99+0800][INFO] updated /etc/kubernetes/pki/etcd/server.conf
[2021-09-12T16:41:26.04+0800][INFO] updated /etc/kubernetes/pki/etcd/peer.conf
[2021-09-12T16:41:26.07+0800][INFO] updated /etc/kubernetes/pki/etcd/healthcheck-client.conf
[2021-09-12T16:41:26.11+0800][INFO] updated /etc/kubernetes/pki/apiserver-etcd-client.conf
[2021-09-12T16:41:26.54+0800][INFO] restarted etcd
[2021-09-12T16:41:26.60+0800][INFO] updated /etc/kubernetes/pki/apiserver.crt
[2021-09-12T16:41:26.64+0800][INFO] updated /etc/kubernetes/pki/apiserver-kubelet-client.crt
[2021-09-12T16:41:26.69+0800][INFO] updated /etc/kubernetes/controller-manager.conf
[2021-09-12T16:41:26.74+0800][INFO] updated /etc/kubernetes/scheduler.conf
[2021-09-12T16:41:26.79+0800][INFO] updated /etc/kubernetes/admin.conf
[2021-09-12T16:41:26.79+0800][INFO] backup /root/.kube/config to /root/.kube/config.old-20210912
[2021-09-12T16:41:26.80+0800][INFO] copy the admin.conf to /root/.kube/config
[2021-09-12T16:41:26.85+0800][INFO] updated /etc/kubernetes/kubelet.conf
[2021-09-12T16:41:26.88+0800][INFO] updated /etc/kubernetes/pki/front-proxy-client.crt
[2021-09-12T16:41:28.70+0800][INFO] restarted apiserver
[2021-09-12T16:41:29.17+0800][INFO] restarted controller-manager
[2021-09-12T16:41:30.07+0800][INFO] restarted scheduler
[2021-09-12T16:41:30.13+0800][INFO] restarted kubelet
[2021-09-12T16:41:30.14+0800][INFO] done!!!
CERTIFICATE                                       EXPIRES
/etc/kubernetes/controller-manager.config         Sep 11 08:41:26 2031 GMT
/etc/kubernetes/scheduler.config                  Sep 11 08:41:26 2031 GMT
/etc/kubernetes/admin.config                      Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/ca.crt                        Sep 11 08:38:53 2031 GMT
/etc/kubernetes/pki/apiserver.crt                 Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/apiserver-kubelet-client.crt  Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/front-proxy-ca.crt            Sep 11 08:38:54 2031 GMT
/etc/kubernetes/pki/front-proxy-client.crt        Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/etcd/ca.crt                   Sep 11 08:38:55 2031 GMT
/etc/kubernetes/pki/etcd/server.crt               Sep 11 08:41:25 2031 GMT
/etc/kubernetes/pki/etcd/peer.crt                 Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/etcd/healthcheck-client.crt   Sep 11 08:41:26 2031 GMT
/etc/kubernetes/pki/apiserver-etcd-client.crt     Sep 11 08:41:26 2031 GMT
```

将更新以下证书和 kubeconfig 配置文件

```
/etc/kubernetes
├── admin.conf
├── controller-manager.conf
├── scheduler.conf
├── kubelet.conf
└── pki
    ├── apiserver.crt
    ├── apiserver-etcd-client.crt
    ├── apiserver-kubelet-client.crt
    ├── front-proxy-client.crt
    └── etcd
        ├── healthcheck-client.crt
        ├── peer.crt
        └── server.crt
```

[更多](https://github.com/yuyicai/update-kube-cert/blob/master/other.md)

### 2. 证书更新失败回滚

脚本会自动备份 `/etc/kubernetes` 目录到 `/etc/kubernetes.old-$(date +%Y%m%d)` 目录（备份目录命名示例：`kubernetes.old-20200325`）

若更新证书失败需要回滚，手动将备份 `/etc/kubernetes.old-$(date +%Y%m%d)`目录覆盖 `/etc/kubernetes` 目录



参考文章：

https://github.com/kubernetes/kubernetes/blob/master/pkg/features/kube_features.go

https://cloud.tencent.com/developer/article/1689059

https://github.com/yuyicai/update-kube-cert

http://docs.kubernetes.org.cn/829.html
