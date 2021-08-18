



# Label标签

Label是用户可灵活定义的对象属性，对于正在运行的资源对象，随时可以通过kubectl label命令进行增加、修改、删除等操作。

# 1 Label含义

Label（标签）是Kubernetes系统中另外一个核心概念。**一个Label是一个key=value的键值对**，其中key与value由用户自己指定。

Label可以被附加到各种资源对象上，例如Node、Pod、Service、RC 等，一个资源对象可以定义任意数量的Label，同一个Label也可以被添加到任意数量的资源对象上。Label可以在创建对象时就附加到对象上，也可以在对象创建后通过API进行额外添加或修改。

**每一个对象可以拥有多个标签，但是，key值必须是唯一的。** 



# 2 Label Selector

给某个资源对象定义一个 Label，就相当于给它打了一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象，Label Selector可以被类比为SQL语句中的where查询条件，

当前有两种Label Selector表达式：基于等式的（Equality-based）和基于集合的（Set-based）。

**等式类表达式匹配标签：**

- name=redis-slave：匹配所有具有标签name=redis-slave的资源对象。
- env!=production：匹配所有不具有标签env=production的资源对象，比如env=test就是满足此条件的标签之一。 

**集合操作类表达式匹配标签：**

- name in（redis-master, redis-slave）：匹配所有具有标签name=redis-master或者name=redis-slave的资源对象。 
- name not in（php-frontend）：匹配所有不具有标签name=php-frontend的资源对象。 

通过多个Label Selector表达式的组合实现复杂的条件选择，多个表达式之间用“，”进行分隔即可，几个条件之间是“AND”的关系，即同时满足多个条件，比如下面的例子：

```
name=redis-slave,env!=production
name notin (php-frontend),env!=production
```

matchLabels用于定义一组Label，与直接写在Selector中的作用相同；matchExpressions用于定义一组基于集合的筛选条件，可用的条 件运算符包括In、NotIn、Exists和DoesNotExist。如果同时设置了matchLabels和matchExpressions，则两组条件为AND关系，即需要同时满足所有条件才能完成Selector的筛选。 

要查看每个Pod自动生成的标签，运行`kubectl get pods --show-labels`输出。



# 3 命名规则

label 必须以字母或数字开头，可以使用字母、数字、连字符、点和下划线，最长63个字符。



# 4 作用

通过对资源对象捆绑一个或多个不同的Label来实现多维度的资源分组管理功能，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

例如，部署不同版本的应用到不同的环境中；监控和分析应用（日志记录、监控、告警）等。一些常用的Label示例如下：

- 版本标签："release":"stable"、"release":"canary"。 
- 环境标签："environment":"dev"、"environment":"qa"、"environment":"production"。 
- 架构标签："tier":"frontend"、"tier":"backend"、"tier":"middleware"。 
- 分区标签："partition":"customerA"、"partition":"customerB"。 
- 质量管控标签："track":"daily"、"track":"weekly"。



# 5 日常操作

## 5.1 查看pod和node的标签

```bash
$ kubectl get pod --show-labels
NAME                      READY   STATUS    RESTARTS   AGE   LABELS
glusterfs-8d5pb           1/1     Running   1          89d   controller-revision-hash=6959b57b6,glusterfs-node=daemonset,pod-template-generation=1
glusterfs-dt7q6           1/1     Running   1          89d   controller-revision-hash=6959b57b6,glusterfs-node=daemonset,pod-template-generation=1
glusterfs-swss6           1/1     Running   1          89d   controller-revision-hash=6959b57b6,glusterfs-node=daemonset,pod-template-generation=1
heketi-7d8bd8cd86-5wpn9   1/1     Running   0          88d   glusterfs=heketi-pod,name=heketi,pod-template-hash=7d8bd8cd86

$ kubectl get node --show-labels
NAME           STATUS   ROLES    AGE    VERSION    LABELS
k8s-master40   Ready    master   133d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master40,kubernetes.io/os=linux,node-role.kubernetes.io/master=,ssd=true,storagenode=glusterfs,zone=foo
k8s-node41     Ready    <none>   132d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node41,kubernetes.io/os=linux,storagenode=glusterfs,zone=foo
k8s-node42     Ready    <none>   103d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node42,kubernetes.io/os=linux,ssd=true,storagenode=glusterfs,zone=bar

```



## 5.2 查看匹配标签key的pod

```bash
$ kubectl get pod -L app
NAME                                      READY   STATUS    RESTARTS   AGE   APP
nfs-client-provisioner-7c79ffd999-lscvc   1/1     Running   0          2d    nfs-client-provisioner
web-0                                     1/1     Running   0          2d    nginx
web-1                                     1/1     Running   0          47h   nginx
```

## 5.3 给pod添加标签

```bash
$ kubectl label pods web-0 env=test
pod/web-0 labeled
$ kubectl get pod -L env
NAME                                      READY   STATUS    RESTARTS   AGE   ENV
nfs-client-provisioner-7c79ffd999-lscvc   1/1     Running   0          2d
web-0                                     1/1     Running   0          2d    test
web-1                                     1/1     Running   0          47h
```



## 5.4 更新标签

```bash
$ kubectl label --overwrite pods web-0 env=prod
pod/web-0 labeled
$ kubectl get pod -L env
NAME                                      READY   STATUS    RESTARTS   AGE   ENV
nfs-client-provisioner-7c79ffd999-lscvc   1/1     Running   0          2d
web-0                                     1/1     Running   0          2d    prod
web-1                                     1/1     Running   0          47h
```



## 5.5 删除标签

```bash
# 删除env=prod的标签（使用“ - ”减号相连）
$ kubectl label pods web-0 env-
```

