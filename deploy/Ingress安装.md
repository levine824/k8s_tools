# Ingress安装

本文介绍Ingress高可用安装。

为了配置kubernetes中的ingress的高可用，对于kubernetes集群以外只暴露一个访问入口，需要使用keepalived排除单点问题。需要使用daemonset方式将ingress-controller部署在边缘节点上。

所谓的边缘节点即集群内部用来向集群外暴露服务能力的节点，集群外部的服务通过该节点来调用集群内部的服务，边缘节点是集群内外交流的一个Endpoint。

**边缘节点要考虑两个问题**

- 边缘节点的高可用，不能有单点故障，否则整个kubernetes集群将不可用
- 对外的一致暴露端口，即只能有一个外网访问IP和端口

为了满足边缘节点的以上需求，我们使用[keepalived](http://www.keepalived.org/)来实现。

在Kubernetes中添加了ingress后，在DNS中添加A记录，域名为你的ingress中host的内容，IP为你的keepalived的VIP，这样集群外部就可以通过域名来访问你的服务，也解决了单点故障。

## 获取Ingress的Yaml

```shell
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/tree/main/deploy/static/provider/baremetal/deploy.yaml
```

## 增加节点标签

```shell
kubectl label node master-1 edgenode=true
kubectl label node master-2 edgenode=true
kubectl label node master-3 edgenode=true
kubectl get nodes --show-labels
```

## 修改Yaml文件

1.修改Deployment为DaemonSet，并注释掉副本数。

```yaml
kind: DaemonSet
  #replicas: 1
```

2、启用hostNetwork网络，并指定运行节点

hostNetwork暴露ingress-nginx controller的相关业务端口到主机，这样node节点主机所在网络的其他主机，都可以通过该端口访问到此应用程序。

dnsPolicy: ClusterFirstWithHostNet ，使用hostNetwork后容器会使用物理机网络包括DNS，会无法解析内部service，使用此参数让容器使用K8S集群内部的DNS

nodeSelector指定之前添加edgenode=true标签的node。

```yaml
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      nodeSelector:
        edgenode: 'true'
```

3.删除controller-service相关Yaml

## 安装Ingress

```shell
kubectl apply -f ingress.yaml

[root@master-1 ingress]# kubectl get pods -n ingress-nginx             
NAME                                      READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create--1-sjwlq   0/1     Completed   0          3m52s
ingress-nginx-admission-patch--1-tc622    0/1     Completed   1          3m52s
ingress-nginx-controller-565rj            1/1     Running     0          3m52s
ingress-nginx-controller-7wl6b            1/1     Running     0          3m52s
ingress-nginx-controller-qzdgr            1/1     Running     0          3m52s
```

## 安装Keepalived

此处不再赘述。





参考文章：

https://www.cnblogs.com/keep-live/p/11882829.html

https://github.com/kubernetes/ingress-nginx