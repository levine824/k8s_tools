# K8s安装

## 架构图

![img](../images/1805318-20200802132335968-769331513.png)

本文采用三台主机作为master节点，通过haproxy和keepalived实现apiserver的负载均衡，达到高可用的目的。

## 主机规划

|     ip     |  主机名  |              角色              |
| :--------: | :------: | :----------------------------: |
| 10.1.19.50 | master-1 | master节点、keepalived,haproxy |
| 10.1.19.51 | master-2 | master节点、keepalived,haproxy |
| 10.1.19.52 | master-3 | master节点、keepalived,haproxy |
| 10.1.19.53 |  node-1  |            node节点            |
| 10.1.19.54 |  node-2  |            node节点            |
| 10.1.19.55 |  node-3  |            node节点            |
| 10.1.19.35 |  ceph-1  |            ceph节点            |
| 10.1.19.36 |  ceph-2  |            ceph节点            |
| 10.1.19.37 |  ceph-3  |            ceph节点            |
| 10.1.19.38 |  ceph-4  |              待定              |

## 安装前准备

### 关闭防火墙以及SELINUX

```shell
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/^SELINUX=disabled/SELINUX=disabled/g' /etc/selinux/config
```

### 关闭Swap

```shell
swapoff -a && sysctl -w vm.swappiness=0
sed -i 's/.*swap.*/#&/' /etc/fstab
```

### 修改内核参数

```shell
# 加载br_netfilter 模块
modprobe br_netfilter
#将桥接的IPv4流量传递到iptables的链
cat << EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl -p  /etc/sysctl.d/k8s.conf
# 启动ip 转发
echo 1 > /proc/sys/net/ipv4/ip_forward
```

### 设置主机名

```shell
#主机名根据实际情况修改
hostnamectl set-hostname master-1
```

### 配置SSH免密登陆

修改`/etc/hosts`文件，添加主机名：

```shell
10.1.19.50 master-1
10.1.19.51 master-2
10.1.19.52 master-3
10.1.19.53 node-1
10.1.19.54 node-2
10.1.19.55 node-3
```

制作SSH证书，并拷贝到各个节点：

```shell
#根据实际情况拷贝
ssh-keygen -t rsa
ssh-copy-id root@master-1
```

### 安装必要工具

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl  ebtables bash-completion
source /usr/share/bash-completion/bash_completion
```

### 时钟同步

```shell
systemctl enable ntpdate.service
echo '*/30 * * * * /usr/sbin/ntpdate time7.aliyun.com >/dev/null 2>&1' > /tmp/crontab2.tmp
crontab /tmp/crontab2.tmp
systemctl start ntpdate.service

ntpdate time7.aliyun.com
hwclock -w
```

### 修改内核参数

```shell
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf
```

### 在节点上安装docker

```shell
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install docker-ce -y
systemctl start docker && systemctl enable docker
#配置docker加速
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io && systemctl restart docker
```

### 添加Kubernetes源

```shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装ipset服务

```shell
yum -y install ipvsadm ipset sysstat conntrack libseccomp
```

### 开启ipvs模块

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
     #!/bin/sh
     modprobe -- ip_vs
     modprobe -- ip_vs_rr
     modprobe -- ip_vs_wrr
     modprobe -- ip_vs_sh
     modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

### 重启主机

```shell
reboot
```

## 安装Haproxy

###  yum安装Haproxy

```shell
yum install -y haproxy
```

### 配置Haproxy

```shell
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the masters
#---------------------------------------------------------------------
frontend apiserver #根据实际情况命名
    bind *:16443 #根据实际情况配置端口
    mode tcp
    option tcplog
    default_backend apiserver #和下方backend名字对应

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver #根据实际情况命名
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server master-1 10.1.19.50:6443 check  #根据实际情况配置
        server master-2 10.1.19.51:6443 check
        server master-3 10.1.19.52:6443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------        
listen stats
    bind                 *:1080
    mode http
    stats enable
    stats refresh 10s                  #统计页面自动刷新时间
    stats uri /haproxy-admin           #监控页面的访问地址
    stats realm Haproxy\ Statistics    #统计页面密码框上提示文本
    stats auth admin:123456             #统计页面用户名和密码设置
    stats hide-version                 #隐藏统计页面上HAProxy的版本信息
EOF
```

### 启动Haproxy

```shell
systemctl enable haproxy.service;
systemctl start haproxy.service;
```

## 安装Keepalived

### yum安装Keepalived

```shell
yum install -y keepalived
```

### 配置Keepalived

##### master节点

```shell
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens32  #根据实际网卡名修改
    virtual_router_id 51 #所有节点router_id的配置保持一致
    priority 100
    authentication {
        auth_type PASS
        auth_pass asb#1234
    }
    virtual_ipaddress {
        10.1.19.59/24 #根据实际vip配置
    }
    track_script {
        check_apiserver
    }
}
EOF
```

##### backup节点

```shell
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens32
    virtual_router_id 51
    priority 80
    authentication {
        auth_type PASS
        auth_pass asb#1234
    }
    virtual_ipaddress {
        10.1.19.59/24
    }
    track_script {
        check_apiserver
    }
}
EOF
```

### 创建check_apiserver.sh文件

```shell
cat > /etc/keepalived/check_apiserver.sh << EOF
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:16443/ -o /dev/null || errorExit "Error GET https://localhost:16443/"
if ip addr | grep -q 10.1.19.59; then
    curl --silent --max-time 2 --insecure https://10.1.19.59:16443/ -o /dev/null || errorExit "Error GET https://10.1.19.59:16443/"
fi
EOF
```

### 启动Keepalived

```shell
systemctl enable keepalived.service;
systemctl start keepalived.service;
```

## 安装kubeadm、kubelet和kubectl

在所有节点执行以下命令安装：

```shell
yum -y install kubeadm-1.22.1 kubelet-1.22.1 kubectl-1.22.1
systemctl enable kubelet && systemctl daemon-reload
```

## 获取配置文件

在其中一台master节点执行：

```shell
kubeadm config print init-defaults > kubeadm-config.yaml
```

## 修改配置文件

```shell
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.1.19.50     # 本机IP
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master-1        # 本主机名
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.1.19.59:16443"    # 虚拟IP和haproxy端口
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers    # 镜像仓库源
kind: ClusterConfiguration
kubernetesVersion: v1.22.1     # k8s版本
networking:
  dnsDomain: cluster.local
  podSubnet: "10.244.0.0/16"
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

## 下载相关镜像

```shell
kubeadm config images pull --config kubeadm-config.yaml
```

## 初始化集群

```shell
kubeadm init --config kubeadm-config.yaml
```

## 创建目录

在其他两个master节点创建以下目录：

```shell
mkdir -p /etc/kubernetes/pki/etcd
```

在所有node节点创建以下目录：

```shell
mkdir -p /etc/kubernetes
```

## 拷贝证书文件

```shell
scp /etc/kubernetes/pki/ca.* root@master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* root@master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* root@master-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* root@master-2:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf root@master-2:/etc/kubernetes/
scp /etc/kubernetes/pki/ca.* root@master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* root@master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* root@master-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* root@master-3:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf root@master-3:/etc/kubernetes/

scp /etc/kubernetes/admin.conf root@node-1:/etc/kubernetes/
scp /etc/kubernetes/admin.conf root@node-2:/etc/kubernetes/
scp /etc/kubernetes/admin.conf root@node-3:/etc/kubernetes/
```

## 加入集群认证

将初始化时显示的加入集群命令分别在master节点和node节点执行：

```shell
kubeadm join 10.1.19.59:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:dc523896010f290a38391c9fb60892b3a59447825250c2f4f255a3cac8e30b97 \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.1.19.59:16443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:dc523896010f290a38391c9fb60892b3a59447825250c2f4f255a3cac8e30b97 
```

备注：集群初始化失败重置集群，重新初始化：

```shell
kubeadm reset
```

### 拷贝kube-config文件

根据初始化节点后提示，执行命令。

## 查看所有节点状态

```shell
kubectl get nodes

kubectl get pods --all-namespaces
```

## 安装Flannel

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

