[TOC]





# Pod资源调度

API Server在接受客户端提交Pod对象创建请求后，然后是通过调度器（kube-schedule）从集群中选择一个可用的最佳节点来创建并运行Pod。而这一个创建Pod对象，在调度的过程当中有3个阶段：节点预选、节点优选、节点选定，从而筛选出最佳的节点。

如图：

![201903151107441647](assets/201903151107441647.png)

- 节点预选：基于一系列的预选规则对每个节点进行检查，将那些不符合条件的节点过滤，从而完成节点的预选
- 节点优选：对预选出的节点进行优先级排序，以便选出最合适运行Pod对象的节点
- 节点选定：从优先级排序结果中挑选出优先级最高的节点运行Pod，当这类节点多于1个时，则进行随机选择

当我们有需求要将某些Pod资源运行在特定的节点上时，我们可以通过组合节点标签，以及Pod标签或标签选择器来匹配特定的预选策略并完成调度，如`MatchInterPodAfinity、MatchNodeSelector、PodToleratesNodeTaints`等预选策略，这些策略常用于为用户提供自定义Pod亲和性或反亲和性、节点亲和性以及基于污点及容忍度的调度机制。



# 1 常用的预选策略

预选策略实际上就是节点过滤器，例如节点标签必须能够匹配到Pod资源的标签选择器（MatchNodeSelector实现的规则），以及Pod容器的资源请求量不能大于节点上剩余的可分配资源（PodFitsResource规则）等等。执行预选操作，调度器会逐一根据规则进行筛选，如果预选没能选定一个合适的节点，此时Pod会一直处于Pending状态，直到有一个可用节点完成调度。其常用的预选策略如下：

- CheckNodeCondition：检查是否可以在节点报告磁盘、网络不可用或未准备好的情况下将Pod对象调度其上。
- HostName：如果Pod对象拥有spec.hostname属性，则检查节点名称字符串是否和该属性值匹配。
- PodFitsHostPorts：如果Pod对象定义了ports.hostPort属性，则检查Pod指定的端口是否已经被节点上的其他容器或服务占用。
- MatchNodeSelector：如果Pod对象定义了spec.nodeSelector属性，则检查节点标签是否和该属性匹配。
- NoDiskConflict：检查Pod对象请求的存储卷在该节点上可用。
- PodFitsResources：检查节点上的资源（CPU、内存）可用性是否满足Pod对象的运行需求。
- PodToleratesNodeTaints：如果Pod对象中定义了spec.tolerations属性，则需要检查该属性值是否可以接纳节点定义的污点（taints）。
- PodToleratesNodeNoExecuteTaints：如果Pod对象定义了spec.tolerations属性，检查该属性是否接纳节点的NoExecute类型的污点。
- CheckNodeLabelPresence：仅检查节点上指定的所有标签的存在性，要检查的标签以及其可否存在取决于用户的定义。
- CheckServiceAffinity：根据当前Pod对象所属的Service已有其他Pod对象所运行的节点调度，目前是将相同的Service的Pod对象放在同一个或同一类节点上。
- MaxEBSVolumeCount：检查节点上是否已挂载EBS存储卷数量是否超过了设置的最大值，默认值：39
- MaxGCEPDVolumeCount：检查节点上已挂载的GCE PD存储卷是否超过了设置的最大值，默认值：16
- MaxAzureDiskVolumeCount：检查节点上已挂载的Azure Disk存储卷数量是否超过了设置的最大值，默认值：16
- CheckVolumeBinding：检查节点上已绑定和未绑定的PVC是否满足Pod对象的存储卷需求。
- NoVolumeZoneConflct：在给定了区域限制的前提下，检查在该节点上部署Pod对象是否存在存储卷冲突。
- CheckNodeMemoryPressure：在给定了节点已经上报了存在内存资源压力过大的状态，则需要检查该Pod是否可以调度到该节点上。
- CheckNodePIDPressure：如果给定的节点已经报告了存在PID资源压力过大的状态，则需要检查该Pod是否可以调度到该节点上。
- CheckNodeDiskPressure：如果给定的节点存在磁盘资源压力过大，则检查该Pod对象是否可以调度到该节点上。
- MatchInterPodAffinity：检查给定的节点能否可以满足Pod对象的亲和性和反亲和性条件，用来实现Pod亲和性调度或反亲和性调度。

在上面的这些预选策略里面，CheckNodeLabelPressure和CheckServiceAffinity可以在预选过程中结合用户自定义调度逻辑，这些策略叫做可配置策略。其他不接受参数进行自定义配置的称为静态策略。



# 2 优选函数

预选策略筛选出一个节点列表就会进入优选阶段，在这个过程调度器会向每个通过预选的节点传递一系列的优选函数来计算其优先级分值，优先级分值介于0-10之间，其中0表示不适用，10表示最适合托管该Pod对象。

另外，调度器还支持给每个优选函数指定一个简单的值，表示权重，进行节点优先级分值计算时，它首先将每个优选函数的计算得分乘以权重，然后再将所有优选函数的得分相加，从而得出节点的最终优先级分值。权重可以让管理员定义优选函数倾向性的能力，其计算优先级的得分公式如下：

```
finalScoreNode = (weight1 * priorityFunc1) + (weight2 * priorityFunc2) + ......
```

下图是关于优选函数的列表图：

![20190315110812120](assets/20190315110812120.png)





# 3 Node亲和性调度

NodeAffinity意为Node亲和性的调度策略，是用于替换NodeSelector的全新调度策略。

节点亲和性是用来确定Pod对象调度到哪一个节点的规则，这些规则基于节点上的自定义标签和Pod对象上指定的标签选择器进行定义。

例如，将Pod调度至有着特殊CPU的节点或一个可用区域内的节点之上 。

定义节点亲和性规则有两种：**硬亲和性（require）**和**软亲和性（preferred）**

- 硬亲和性：实现的是强制性规则，指Pod调度时必须满足的规则，如果不存在满足规则的节点时 ，Pod对象的状态会一直是Pending。表达式为`RequiredDuringSchedulingIgnoredDuringExecution`。
- 软亲和性：实现的是一种柔性调度限制，在Pod调度时可以尽量满足其规则，在无法满足规则时，可以调度到一个不匹配规则的节点之上。表达式为`PreferredDuringSchedulingIgnoredDuringExecution`。

表达式中`IgnoredDuringExecution`的意思是：在Pod资源基于节点亲和性规则调度至某节点之后，如果节点标签发生了改变而不再符合此节点亲和性规则时 ，调度器不会将Pod对象从此节点上移出，因为该规则仅对新建的Pod对象生效。

节点亲和性模型如图所示：

![20200421-01](assets/20200421-01.png)



## 3.1 Node硬亲和性

为`Pod`对象使用`nodeSelector`属性可以基于节点标签匹配的方式将`Pod`对象强制调度至某一类特定的节点之上 ，不过它仅能基于简单的等值关系定义标签选择器，而`nodeAffinity`中支持使用 `matchExpressions`属性构建更为复杂的标签选择机制。

例如，下面示例中定义的`Pod`对象，其使用节点硬亲和规则定义可将当前`Pod`对象调度至拥有`zone`标签且其值为`foo`的节点之上：

vim with-required-nodeaffinity.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - {key: zone, operator: In, values: ["foo"]}
  containers:
  - name: nginx
    image: nginx
```

将上面配置清单中定义的资源创建于集群之中，由其状态信息可知它处于`Pending`阶段，这是由于节点硬亲和限制，节点不存在能够满足匹配条件所致：

```bash
$ kubectl apply -f with-required-nodeaffinity.yaml
pod/with-required-nodeaffinity created
$ kubectl get pod with-required-nodeaffinity
NAME                         READY   STATUS    RESTARTS   AGE
with-required-nodeaffinity   0/1     Pending   0          21s
```

通过`describe`查看对应的`events`：

```bash
$ kubectl describe pod with-required-nodeaffinity
...
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  57s   default-scheduler  0/3 nodes are available: 3 node(s) didn't match node selector.
  Warning  FailedScheduling  57s   default-scheduler  0/3 nodes are available: 3 node(s) didn't match node selector.
```

规划为各节点设置节点标签 ，这也是设置节点亲和性的前提之一：

```bash
$ kubectl label node k8s-master40 zone=foo
node/k8s-master40 labeled
$ kubectl label node k8s-node41 zone=foo
node/k8s-node41 labeled
$ kubectl label node k8s-node42 zone=bar

# 查看所有节点标签
$ kubectl get nodes --show-labels
NAME           STATUS   ROLES    AGE   VERSION    LABELS
k8s-master40   Ready    master   48d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-master40,kubernetes.io/os=linux,node-role.kubernetes.io/master=,storagenode=glusterfs,zone=foo
k8s-node41     Ready    <none>   47d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node41,kubernetes.io/os=linux,storagenode=glusterfs,zone=foo
k8s-node42     Ready    <none>   18d   v1.18.18   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=k8s-node42,kubernetes.io/os=linux,storagenode=glusterfs,zone=bar

```

然后查看pod状态和调度结果：

```bash
$ kubectl get pod with-required-nodeaffinity
NAME                         READY   STATUS    RESTARTS   AGE
with-required-nodeaffinity   1/1     Running   0          28m

$ kubectl describe pod with-required-nodeaffinity
...
  Normal   Scheduled         6m6s   default-scheduler  Successfully assigned default/with-required-nodeaffinity to k8s-master40
  Normal   Pulling           6m5s   kubelet            Pulling image "nginx"
  Normal   Pulled            5m55s  kubelet            Successfully pulled image "nginx"
  Normal   Created           5m55s  kubelet            Created container nginx
  Normal   Started           5m54s  kubelet            Started container nginx

```

发现pod已经Running。

在定义节点亲和性时，`requiredDuringSchedulinglgnoredDuringExecution`字段的值是一个对象列表，用于定义节点硬亲和性，它可由一到多个`nodeSelectorTerm`定义的对象组成， 彼此间为“逻辑或”的关系，进行匹配度检查时，在多个`nodeSelectorTerm`之间只要满足其中之一 即可。`nodeSelectorTerm`用于定义节点选择器条目，其值为对象列表，它可由一个或多个`matchExpressions`对象定义的匹配规则组成，多个规则彼此之间为“逻辑与”的关系， 这就意味着某节点的标签需要完全匹配同一个`nodeSelectorTerm`下所有的`matchExpression`对象定义的规则才算成功通过节点选择器条目的检查。而`matchExmpressions`又可由 一到多 个标签选择器组成，多个标签选择器彼此间为“逻辑与”的关系 。

给两个节点打上`ssd=true`的标签：

```bash
kubectl label node k8s-master40 ssd=true
kubectl label node k8s-node42 ssd=true
```

下面的资源配置清单示例中定义了调度拥有**两个标签选择器**的节点挑选条目，两个标签选择器彼此之间为“逻辑与”的关系，因此，满足其条件的节点为`k8s-master40`和`k8s-node42`：

vim with-required-nodeaffinity-2.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity-2
spec:
  affinity:
    nodeAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - {key: zone, operator: In, values: ["foo", "bar"]}
          - {key: ssd, operator: Exists, values: []}
  containers:
  - name: nginx
    image: nginx

```

执行后会发现pod会运行在`k8s-master40`或`k8s-node42`任意节点。

构建标签选择器表达式中支持使用操作符有`In`、`Notln`、`Exists`、`DoesNotExist`、`Lt`和`Gt`等

- In：`label`的值在某个列表中
- NotIn：`label`的值不在某个列表中
- Gt：`label`的值大于某个值
- Lt：`label`的值小于某个值
- Exists：某个`label`存在
- DoesNotExist：某个`label`不存在

另外，调度器在调度`Pod`资源时，节点亲和性`MatchNodeSelector`仅是其节点预选策略中遵循的预选机制之一，其他配置使用的预选策略依然正常参与节点预选过程。 

例如，将上面资源配置清单示例中定义的`Pod`对象容器修改为如下内容并进行测试：

vim with-required-nodeaffinity-3.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-required-nodeaffinity-3
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - {key: zone, operator: In, values: ["foo", "bar"]}
          - {key: ssd, operator: Exists, values: []}
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 6
        memory: 20Gi

```

执行`kubectl apply -f with-required-nodeaffinity-3.yaml`后；会发现`with-required-nodeaffinity-3`一直处于`Pending`；因为`0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.`没有节点符合cpu和memory的要求。

在预选策略`PodFitsResources`根据节点资源可用性进行节点预选的过程中，它会获取给定节点的可分配资源量（资源总量减去已被运行于其上的各`Pod`对象的`requests`属性之和），去除那些无法容纳新`Pod`对象请求的资源量的节点，如果资源不够，同样会调度失败。

由上述操作过程可知，节点硬亲和性实现的功能与节点选择器`nodeSelector`相似， 但亲和性支持使用匹配表达式来挑选节点，这一点提供了灵活且强大的选择机制，因此可被理解为新一代的节点选择器。



## 3.2 Node软亲和性

节点软亲和性为节点选择机制提供了一种柔性控制逻辑，被调度的`Pod`对象不再是“必须”而是“应该”放置于某些特定节点之上，当条件不满足时它也能够接受被编排于其他不符合条件的节点之上。另外，它还为每种倾向性提供了`weight`属性以便用户定义其优先级，取值范围是`1 ～ 100`，数字越大优先级越高 。

vim myapp-deploy-with-node-affinity.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deploy-with-node-affinity
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 60
            preference:
              matchExpressions:
              - {key: zone, operator: In, values: ["foo"]}
          - weight: 30
            preference:
              matchExpressions:
              - {key: ssd, operator: Exists, values: []}
      containers:
      - name: nginx
        image: nginx
```

`Pod`资源模板定义了节点软亲和性以选择运行在拥有`zone=foo`和`ssd`标签（无论其值为何）的节点之上， 其中`zone=foo`是更为重要的倾向性规则， 它的权重为`60`，相比较来说，`ssd`标签就没有那么关键， 它的权重为`30`。 这么一来， 如果集群中拥有足够多的节点，那么它将被此规则分为四类 ： 同时满足拥有`zone=foo`和`ssd`标签、仅具有`zoo=foo`标 签、 仅具有`ssd`标签， 以及不具备此两个标签。

如图所示:

<img src="assets/20200421-02.png" alt="20200421-02" style="zoom: 67%;" />

示例环境共有三个节点，相对于定义的节点亲和性规则来说，它们所拥有的倾向性权重分别如图所示。在创建需要`3`个`Pod`对象的副本时，其运行效果为三个`Pod`对象被分散运行于集群中的三个节点之上，而非集中运行于某一个节点 。

之所以如此，是因为使用了节点软亲和性的预选方式，所有节点均能够通过调度器上`MatchNodeSelector`预选策略的筛选，因此，可用节点取决于其他预选策略的筛选结果。在第二阶段的优选过程中，除了`NodeAffinityPriority`优选函数之外，还有其他几个优选函数参与优先级评估，尤其是`SelectorSpreadPriority`，它会将同一个`ReplicaSet`控制器管控的所有`Pod`对象分散到不同的节点上运行；以抵御节点故障带来的风险 。不过，这种节点亲和性的权重依然在发挥作用，如果把副本数量扩展至越过节点数很多，如`15`个， 那么它们将被调度器以接近节点亲和性权重比值`90:60:30`的方式分置于相关的节点之上。





# 4 Pod资源亲和调度

在出于高效通信的需求，有时需要将一些Pod调度到相近甚至是同一区域位置（比如同一节点、机房、区域）等等，比如业务的前端Pod和后端Pod，此时这些Pod对象之间的关系可以叫做亲和性。

同时出于安全性的考虑，也会把一些Pod之间进行隔离，此时这些Pod对象之间的关系叫做反亲和性（anti-affinity）。

调度器把第一个Pod放到任意位置，然后和该Pod有亲和或反亲和关系的Pod根据该动态完成位置编排，这就是Pod亲和性和反亲和性调度的作用。Pod的亲和性定义也存在硬亲和性和软亲和性的区别，其约束的意义和节点亲和性类似。

Pod的亲和性调度要求各相关的Pod对象运行在同一位置，而反亲和性则要求它们不能运行在同一位置。这里的位置实际上取决于节点的位置拓扑，拓扑的方式不同，Pod是否在同一位置的判定结果也会有所不同。

如果基于各个节点的`kubernetes.io/hostname`标签作为评判标准，那么会根据节点的`hostname`去判定是否在同一位置区域，如图所示：

<img src="assets/20200421-03.png" alt="20200421-03" style="zoom:80%;" />

如果是基于所划分的故障转移域来进行评判，同一位置， 而`server2`和`server3`属于另一个意义上的同一位置：

<img src="assets/20200421-04.png" alt="20200421-04" style="zoom:80%;" />

因此，在定义`Pod`对象的亲和性与反亲和性时，需要借助于标签选择器来选择被依赖的`Pod`对象，并根据选出的`Pod`对象所在节点的标签来判定“同一位置”的具体意义。



## 4.1 Pod硬亲和性

`Pod`强制约束的亲和性调度也使用`requiredDuringSchedulinglgnoredDuringExecution`属性进行定义。`Pod`亲和性用于描述一个`Pod`对象与具有某特征的现存`Pod`对象运行位置的依赖关系，因此，测试使用`Pod`亲和性约束，需要事先存在被依赖的`Pod`对象，它们具有特别的识别标签。

例如创建一个有着标签`app=tomcat`的`Deployment`资源部署一个`Pod`对象：







