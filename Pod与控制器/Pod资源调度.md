[TOC]





# Pod资源调度

API Server在接受客户端提交Pod对象创建请求后，然后是通过调度器（kube-schedule）从集群中选择一个可用的最佳节点来创建并运行Pod。而这一个创建Pod对象，在调度的过程当中有3个阶段：节点预选、节点优选、节点选定，从而筛选出最佳的节点。如图：

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

`Kubernetes`的默认调度器以预选、优选、选定机制完成将每个新的`Pod`资源绑定至为其选出的目标节点上，不过，它只是`Pod`对象的默认调度器，默认情况下调度器考虑的是资源足够，并且负载尽量平均。

在使用中，用户还可以自定义调度器插件，并在定义`Pod`资源配置清单时通过`spec.schedulerName`指定即可使用，这就是亲和性调度。

节点亲和性是用来确定Pod对象调度到哪一个节点的规则，这些规则基于**节点上的自定义标签**和**Pod对象上指定的标签选择器**进行定义。

定义节点亲和性规则有2种：硬亲和性（require）和软亲和性（preferred）

- 硬亲和性：实现的是强制性规则，是Pod调度时必须满足的规则，否则Pod对象的状态会一直是Pending
- 软亲和性：实现的是一种柔性调度限制，在Pod调度时可以尽量满足其规则，在无法满足规则时，可以调度到一个不匹配规则的节点之上。

定义节点亲和规则的两个要点：一是节点配置是否合乎需求的标签，而是Pod对象定义合理的标签选择器，这样才能够基于标签选择出期望的目标节点。

需要注意的是`preferredDuringSchedulingIgnoredDuringExecution`和`requiredDuringSchedulingIgnoredDuringExecution`名字中后半段字符串`IgnoredDuringExecution`表示的是，在Pod资源基于节点亲和性规则调度到某个节点之后，如果节点的标签发生了改变，调度器不会讲Pod对象从该节点上移除，因为该规则仅对新建的Pod对象有效。



## 3.1 Node硬亲和性

下面的配置清单中定义的Pod对象，使用节点硬亲和性和规则定义将当前Pod调度到标签为zone=foo的节点上：

