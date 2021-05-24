





# Pod控制器

在Kubernetes平台上，我们很少会直接创建一个Pod，在大多数情况下会通过RC、Deployment、DaemonSet、Job等控制器完成对一 组Pod副本的创建、调度及全生命周期的自动控制任务。

控制器 又称之为工作负载，常见包含以下类型控制器：

- ReplicationController（RC） 和 ReplicaSet（RS）
- Deployment
- DaemonSet
- StatefulSet
- Job/CronJob
- HorizontalPodAutoscaler（HPA）自动水平伸缩



ReplicaSet（简称RS）



pod控制器有多种类型：

- : 是Replication Controller 升级版本。代用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。

  ReplicaSet主要三个组件组成：

  - 用户期望的pod副本数量
  - 标签选择器，判断哪个pod归自己管理
  - 当现存的pod数量不足，会根据pod资源模板进行新建帮助用户管理无状态的pod资源，精确反应用户定义的目标数量，但是RelicaSet不是直接使用的控制器，而是使用Deployment。

- Deployment：工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器。支持滚动更新和回滚功能，还提供声明式配置。

- DaemonSet：用于确保集群中的每一个节点只运行特定的pod副本，通常用于实现系统级后台任务,比如ingress,elk.服务是无状态的,服务必须是守护进程。

- Job：一次性任务运行，完成就立即退出，不需要重启或重建。

- Cronjob：周期性任务控制，执行后就退出，不需要持续后台运行。

- StatefulSet：管理有状态应用,比如redis,mysql。









