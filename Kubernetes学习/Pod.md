# Pod

​		**Pod是Kubernetes最重要的基本概念，容器组Pod是最小部署单元，一个Pod有一个或多个容器组成， Pod中容器共享存储和网络，在同一台Docker主机上运行。**

![pod组成](C:%5CUsers%5Cw1818%5CDocuments%5CK8S%5CKubernetes%E5%AD%A6%E4%B9%A0%5Cassets%5Cpod%E7%BB%84%E6%88%90.jpg)

Kubernetes会设计出一个全新的Pod的概念并且有这样特殊的组成结构？ 

1、通过容器组的状态，去解决容器死亡率的问题，更好的判断业务的存活状态；

2、解决多个容器网络通信和文件共享问题；

​		Kubernetes为每个Pod都分配了唯一的IP地址，称之为Pod IP，一个Pod里的多个容器共享Pod IP地址。Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现，例如Flannel、OpenvSwitch等，因此我们需要牢记一点：在Kubernetes里，一个Pod里的容器与另外主机上的Pod容器能够直接通信。

Pod有两种类型：**普通的Pod**和**静态Pod**（Static Pod）。静态Pod比较特殊，它并没被存放在Kubernetes的etcd存储里，而是被存放在 某个具体的Node上的一个具体文件中，并且只在此Node上启动、运行。而普通的Pod一旦被创建，就会被放入etcd中存储，随后会被 Kubernetes Master调度到某个具体的Node上并进行绑定（Binding），随后该Pod被对应的Node上的kubelet进程实例化成一组相关的Docker容器并启动。在默认情况下，当Pod里的某个容器停止时，Kubernetes会自动检测到这个问题并且重新启动这个Pod（重启Pod里 的所有容器），如果Pod所在的Node宕机，就会将这个Node上的所有Pod重新调度到其他节点上。

Pod、容器与Node的关系如图所示：

![pod-容器-node的关系](C:%5CUsers%5Cw1818%5CDocuments%5CK8S%5CKubernetes%E5%AD%A6%E4%B9%A0%5Cassets%5Cpod-%E5%AE%B9%E5%99%A8-node%E7%9A%84%E5%85%B3%E7%B3%BB.jpg)

​		每个Pod都可以对其能使用的服务器上的计算资源设置限额，当前可以设置限额的计算资源有CPU与Memory两种，其中CPU的资源单位为CPU（Core）的数量，是一个绝对值而非相对值。

​		Kubernetes里通常以千分之一的CPU配额为最小单位，用m来表示。通常一个容器的CPU配额被定义为100～300m，即占用0.1～0.3个CPU。与CPU配额类似，Memory配额也是一个绝对值，它的单位是内存字节数。 

在Kubernetes里，一个计算资源进行配额限定时需要设定以下两个参数。

- Requests：该资源的最小申请量，系统必须满足要求。 
- Limits：该资源最大允许使用的量，不能被突破，当容器试图使用超过这个量的资源时，可能会被Kubernetes“杀掉”并重启。

示例：

表明MySQL容器申请最少0.25个CPU及64MiB内存，在运行过程中MySQL容器所能使用的资源配额为0.5个CPU及128MiB内存： 

```yml
spec:
  containers:
  - name: db
    image: mysql
    resource:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

查看pod资源和详细

```bash
$ kubectl get pod
$ kubectl describe pod pod-name
```









https://kubernetes.io/docs/concepts/workloads/pods/