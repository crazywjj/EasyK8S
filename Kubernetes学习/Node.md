[TOC]







# Node

​		Node是Kubernetes集群中的工作负载节点也是一种资源对象，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机时，其上的工作负载会被Master自动转移到其他节点上。

每个Node节点主要由三个模块组成：kubelet、kube-proxy、Container runtime。

| 组件              | 作用                                                         |
| ----------------- | ------------------------------------------------------------ |
| Container runtime | 负责本机的容器创建和管理工作。                               |
| kube-proxy        | 实现Kubernetes Service的通信与负载均衡机制的重要组件。       |
| kubelet           | 负责Pod对应的容器的创建、启停等任务，同时与 Master密切协作，实现集群管理的基本功能。 |



除了核心组件，还有一些推荐的Add-ons（插件）：

- kube-dns负责为整个集群提供DNS服务
- Ingress Controller为服务提供外网入口
- Heapster提供资源监控
- Dashboard提供GUI
- Federation提供跨可用区的集群
- Fluentd-elasticsearch提供集群日志采集、存储与查询



查看node状态和详细信息

```bash
$ kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master40   Ready    master   22d   v1.18.0
k8s-node41     Ready    <none>   21d   v1.18.0
k8s-node42     Ready    <none>   21d   v1.18.0
$ kubectl describe node k8s-node41
Name:               k8s-node41
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=k8s-node41
                    kubernetes.io/os=linux
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"aa:e3:88:dd:ac:95"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.159.238.41
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Wed, 07 Apr 2021 18:20:15 +0800
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  k8s-node41
  AcquireTime:     <unset>
  RenewTime:       Thu, 29 Apr 2021 16:13:50 +0800
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Wed, 07 Apr 2021 18:21:50 +0800   Wed, 07 Apr 2021 18:21:50 +0800   FlannelIsUp                  Flannel is running on this node
  MemoryPressure       False   Thu, 29 Apr 2021 16:13:34 +0800   Wed, 07 Apr 2021 18:20:15 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         False   Thu, 29 Apr 2021 16:13:34 +0800   Wed, 07 Apr 2021 18:20:15 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure          False   Thu, 29 Apr 2021 16:13:34 +0800   Wed, 07 Apr 2021 18:20:15 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                True    Thu, 29 Apr 2021 16:13:34 +0800   Wed, 07 Apr 2021 18:21:55 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  10.159.238.41
  Hostname:    k8s-node41
Capacity:
  cpu:                4
  ephemeral-storage:  102197500Ki
  hugepages-2Mi:      0
  memory:             8155628Ki
  pods:               110
Allocatable:
  cpu:                4
  ephemeral-storage:  94185215845
  hugepages-2Mi:      0
  memory:             8053228Ki
  pods:               110
System Info:
  Machine ID:                 614f9c7e4a914a04976eda64d2b781ab
  System UUID:                ee72c489-8617-42e6-88a2-95b5735e91a5
  Boot ID:                    1786c3a0-7540-4545-a261-b40faaa377b0
  Kernel Version:             5.4.109-1.el7.elrepo.x86_64
  OS Image:                   CentOS Linux 7 (Core)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  docker://18.6.3
  Kubelet Version:            v1.18.0
  Kube-Proxy Version:         v1.18.0
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
Non-terminated Pods:          (16 in total)
  Namespace                   Name                                         CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                   ----                                         ------------  ----------  ---------------  -------------  ---
  default                     web-1                                        0 (0%)        0 (0%)      0 (0%)           0 (0%)         46h
  kube-system                 kube-flannel-ds-5f7fw                        100m (2%)     100m (2%)   50Mi (0%)        50Mi (0%)      21d
  kube-system                 kube-proxy-4z6jw                             0 (0%)        0 (0%)      0 (0%)           0 (0%)         21d
  kubernetes-dashboard        dashboard-metrics-scraper-dc6947fbf-h6597    0 (0%)        0 (0%)      0 (0%)           0 (0%)         21d
  kubernetes-dashboard        kubernetes-dashboard-5d4dc8b976-j5x8q        0 (0%)        0 (0%)      0 (0%)           0 (0%)         21d
  monitoring                  alertmanager-main-0                          104m (2%)     200m (5%)   150Mi (1%)       150Mi (1%)     21d
  monitoring                  alertmanager-main-1                          104m (2%)     200m (5%)   150Mi (1%)       150Mi (1%)     21d
  monitoring                  alertmanager-main-2                          104m (2%)     200m (5%)   150Mi (1%)       150Mi (1%)     21d
  monitoring                  blackbox-exporter-56bc9d4987-xq98x           30m (0%)      60m (1%)    60Mi (0%)        120Mi (1%)     21d
  monitoring                  grafana-c7b7b49b7-8qwjx                      100m (2%)     200m (5%)   100Mi (1%)       200Mi (2%)     21d
  monitoring                  kube-state-metrics-79b955f5d6-5bj9g          40m (1%)      160m (4%)   230Mi (2%)       330Mi (4%)     21d
  monitoring                  node-exporter-67zm5                          112m (2%)     270m (6%)   200Mi (2%)       220Mi (2%)     21d
  monitoring                  prometheus-adapter-85797fb6c8-cg9t9          0 (0%)        0 (0%)      0 (0%)           0 (0%)         21d
  monitoring                  prometheus-k8s-0                             100m (2%)     100m (2%)   450Mi (5%)       50Mi (0%)      21d
  monitoring                  prometheus-k8s-1                             100m (2%)     100m (2%)   450Mi (5%)       50Mi (0%)      21d
  monitoring                  prometheus-operator-5c4b65f789-q7d8x         110m (2%)     220m (5%)   120Mi (1%)       240Mi (3%)     21d
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                1004m (25%)   1810m (45%)
  memory             2110Mi (26%)  1710Mi (21%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
Events:              <none>

```

上述命令展示了Node的如下关键信息。

- Node的基本信息：名称、标签、创建时间等。
- Node当前的运行状态：Node启动后会做一系列的自检工作，比如磁盘空间是否不足（DiskPressure）、内存是否不足
  （MemoryPressure）、网络是否正常（NetworkUnavailable）、PID资源是否充足（PIDPressure）。在一切正常时设置Node为Ready状态（Ready=True），该状态表示Node处于健康状态，Master将可以在其上调度新的任务了（如启动Pod）。
- Node的主机地址与主机名。
- Node上的资源数量：描述Node可用的系统资源，包括CPU、内存数量、最大可调度Pod数量等。
- Node可分配的资源量：描述Node当前可用于分配的资源量。
- 主机系统信息：包括主机ID、系统UUID、Linux kernel版本号、操作系统类型与版本、Docker版本号、kubelet与kube-proxy的版本号等。
- 当前运行的Pod列表概要信息。
- 已分配的资源使用概要信息，例如资源申请的最低、最大允许使用量占系统总量的百分比。
- Node相关的Event信息。