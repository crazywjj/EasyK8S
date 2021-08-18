[TOC]





# Node管理





# 1 Node的扩容与删除

在实际生产系统中经常遇到服务器容量不足的情况，这时候就需要购买新的服务器，对应用系统进行水平扩展以实现扩容。

在k8s中，对一个新的node的加入非常简单，只需要在node节点上安装docker、kubelet和kube-proxy服务，然后将kubelet和kube-proxy的启动参数中的master url指定为当前kubernetes集群master的地址，然后启动服务即可。基于kubelet的自动注册机制，新的node会自动加入现有的kubernetes集群中，如下图：

<img src="assets/node%E6%B3%A8%E5%86%8C.jpg" alt="node注册" style="zoom: 67%;" />

kubernetes master在接受了新node的注册之后，会自动将其纳入当前集群的调度范围内，在之后创建容器时，就可以向新的node进行调度了。



> ==增加node节点==，请参考**Kubeadm部署--单主集群  2.4 章节**。
>
> ==删除node节点==，请参考**Kubeadm部署--单主集群  2.6 章节**。



# 2 Node的隔离与恢复

在硬件升级、硬件维护等情况下，我们需要将某些Node隔离， 使其脱离Kubernetes集群的调度范围。Kubernetes提供了一种机制， 既可以将Node纳入调度范围，也可以将Node脱离调度范围。



## 2.1 通过配置文件实现

创建配置文件unschedule_node.yml，内容如下：

```
apiVersion: v1
kind: Node
metadata:
  name: k8s-node1
  labels:
    namne: k8s-node1
spec:
  unschedulable: true
```

然后执行该配置文件，即可将指定的node脱离调度范围：

```bash
kubectl replace -f unschedule_node.yml
```

查看Node的状态，可以观察到在Node的状态中增加了一项 SchedulingDisabled。这样，对于后续创建的Pod，系统将不会再向该Node进行调度。



## 2.2 通过命令行的方式实现

```bash
kubectl patch node k8s-node1 -p '{"spec":"{"unschedulable":"true"}"}'
```

同样，如果需要将某个Node重新纳入集群调度范围，则将 unschedulable 设置为false，再次执行kubectl replace或kubectl patch命令就能恢复系统对该Node的调度。 

另外，使用kubectl的子命令 cordon 和 uncordon 也可以实现将Node 进行隔离调度和恢复调度操作。 

==**注意：**==将某个Node脱离调度范围时，在其上运行的Pod 并不会自动停止，管理员需要手动停止在该Node上运行的Pod。

