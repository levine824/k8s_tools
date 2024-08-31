# Kubernetes新增/删除节点

## 新增节点

### 安装前准备

将``K8s安装.md``文档中``安装前准备``章节再新节点全部执行，此处不再赘述。

### 安装kubeadm、kubelet和kubectl

在新增节点执行以下命令安装：

```shell
yum -y install kubeadm-1.22.1 kubelet-1.22.1 kubectl-1.22.1
systemctl enable kubelet && systemctl daemon-reload
```

### 节点加入集群

获取加入集群的命令，以下指令在master节点执行:

```shell
[root@master-1 ~]# kubeadm token create --print-join-command
W0927 16:16:58.224174    8022 configset.go:348] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
kubeadm join 10.2.6.66:16443 --token bemli4.ixsk0n7wypt5twnl     --discovery-token-ca-cert-hash sha256:8e8317507ae17fa6cfbeb8acc9dc60851d297184d5ed5fe400ae4100add64365 
```

在新增节点执行上述显示的命令：

```shell
kubeadm join 10.2.6.66:16443 --token bemli4.ixsk0n7wypt5twnl     --discovery-token-ca-cert-hash sha256:8e8317507ae17fa6cfbeb8acc9dc60851d297184d5ed5fe400ae4100add64365
```

## 删除节点

```shell
# 驱逐节点
kubectl drain node03.linux.com --delete-local-data --force --ignore-daemonsets
# 删除节点
kubectl delete node node03.linux.com
```

