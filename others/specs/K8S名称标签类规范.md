# K8S名称标签类规范

# 1   命名标签规范

## 1.1  K8S节点标签命名规范

### 1.1.1 规范目的

node标签设置后，pod可以根据要求让pod调度到想要的节点上运行，或者不在某节点运行。基于此，为了实现统一的打标方式，方便所有同事都能快速、顺利熟悉已有K8S环境特进行命名规范；希望规范指定完成后，大家能快速进行打标操作，而不通重新再思考打标规则；同时也让同事在执行查看节点标签的时候就知道环境是否是直真维护的环境，节点是否是直真维护的节点

### 1.1.2 规范

统一公司标记

zznode=true

其它标记根据实际情况调整，但是都需要遵循“zz-”开头，值为true的方式，如下是一些举例说明：

应用类节点 zz-app=true

服务类节点zz-svc=true

第三方组件部署节点zz-part=true

高性能节点zz-hipe=true

可访问特定网络节点zz-internet=true

可供外部访问节点zz-open=true

注:打标的目的主要是表明节点具有某种身份，所以我们只需要把节点具有的身份写上就可以了

## 1.2  服务名称规范

### 1.2.1 规范目的

规范化的名称方便更换实施人员快速知道服务的分类，只要熟悉了规范的人能尽快熟悉服务的分类

### 1.2.2 规范

服务名称以服务类型开头，中间是自定义的内容，可以是功能缩写等，后面接上svc

如：

app-customer-svc

web-management-svc

第一个例子：app表明这个service是一个应用工程，customer表示这个应用工程是提供客户信息服务的

第二个例子：web表示这是一个提供web访问的服务，management表示这是一个后台管理服务界面的

由于K8S对这类名称有长度限制，中间部分可以进行缩写

## 1.3  POD（Deployment）元数据规范

### 1.3.1 规范目的

规范化的元数据和标签方便在创建集群和服务的时候快速对POD的选择。Pod的元数据规范主要是对命名和标签进行规范，命名规范主要是让pod名称看上去分类清楚，让运维人员有熟悉的感觉；标签规范主要是方便service在选择pod的时候更加方便

### 1.3.2 规范

#### 1.3.2.1 命名规范

POD名称以类型开头，后跟自定义的内容，可以是功能缩写等

如：

app-customer

web-management

第一个例子：app表明这个POD是一个应用工程，customer表示这个应用工程是提供客户信息服务的

第二个例子：web表示这是一个提供web访问的服务，management表示这是一个后台管理服务界面的

由于K8S对这类名称有长度限制，中间部分可以进行缩写

 

#### 1.3.2.2 标签规范

固定两个标签，component和role

component用来指明多个pod是1个应用的多个实例，role用于需要区分的时候进行区分

如：

metadata:

 labels:

  component: authentication

  role: master

和

metadata:

 labels:

  component: authentication

  role: salve

当做集群的时候，服务可以通过component为authentication的方式将对应的pod加入到集群中，而master和slave则是对这两个pod的定位进行说明，不是特别重要，但最后好要有