# K8S容器IP规范

# 1   容器IP规范

## 1.1  容器IP使用规范

### 1.1.1 规范目的

明确容器IP使用规范

### 1.1.2 规范

Pod IP是每个Pod的IP地址，他是Docker Engine根据docker网桥的IP地址段进行分配的，通常是一个虚拟的二层网络

l 同Service下的pod可以直接根据PodIP相互通信

l 不同Service下的pod在集群间pod通信要借助于 cluster ip

l pod和集群外通信，要借助于node ip

 

每一个容器的IP均为唯一值，不允许重复，否则造成冲突

一般情况下，容器的ip均为使用者按照个人喜好创建的独立私网地址段，但是按照项目经验，各大运营商或厂商默认使用的地址段为192.168.0.0/16、172.16.0.0/16、10.0.0.0/12等，个人建议k8s的容器IP尽量与大众化的私有IP地址段独立开，避免重复。

## 1.2  端口配置文件规范

### 1.2.1 规范目的

明确配置文件中容器IP配置格式

### 1.2.2 规范

![img](file:///C:/Users/yuqi/AppData/Local/Temp/msohtmlclip1/01/clip_image001.png)

端口配置模式采用type=NodePort方式

容器IP配置采用clusterIP:ip地址