# K8S服务端口规范

# 1   服务端口使用规范

## 1.1  服务端口使用规范

### 1.1.1 规范目的

明确k8s服务端口使用规范

### 1.1.2 规范

K8s服务本身配合其他开源软件需要部署一系列的基础服务，包括kube-api、kube-controller-manager等，以现有项目经验判断，相关基础k8s服务默认的端口不容易与其他开源软件端口冲突，不建议进行自行修改

 

## 1.2  服务端口规范

### 1.2.1 规范目的

明确各基础k8s服务端口定义值

### 1.2.2 规范

| **k8s**                 | **端口**                                                     |
| ----------------------- | ------------------------------------------------------------ |
| kube-api                | 6443                                                         |
| kube-controller-manager | 10252                                                        |
| kube-scheduler          | 10251                                                        |
| kubelet                 | 10250                                                        |
| kube-proxy              | 10256                                                        |
| etcd                    | 2379与client端交互  2380与etcd内部交互                       |
| node-exporter           | 9100                                                         |
| kube-rbac-proxy         | 9100                                                         |
| kube-state-metrics      | 8080  telemetry-port=8081  监控这些资源，如：deployments, nodes and pods. |
| grafana                 | 3000                                                         |
| prometheus              | 9090                                                         |
| alertmanager            | 9093   6783                                                  |