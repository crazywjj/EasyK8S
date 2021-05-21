[TOC]







# kubernetes版本升级

# 1 二进制升级 

Kubernetes的版本升级需要考虑到不要让当前集群中正在运行的容器受到影响。应对集群中的各Node逐个进行隔离，然后等待在其 上运行的容器全部执行完成，再更新该Node上的kubelet和kube-proxy 服务，将全部Node都更新完成后，再更新Master的服务。 

- 通过官网获取最新版本的二进制包kubernetes.tar.gz，解压后提取服务的二进制文件。 
- 逐个隔离Node，等待在其上运行的全部容器工作完成后，更新kubelet和kube-proxy服务文件，然后重启这两个服务。 
- 更新Master的kube-apiserver、kube-controller-manager、kube-scheduler服务文件并重启。



# 2 kubeadm升级集群

kubeadm提供了upgrade命令用于对kubeadm安装的Kubernetes集群进行升级。每次只能升级一个版本，不支持跨版本升级。



## 2.1 升级步骤

查看当前系统支持的所有k8s版本和当前版本

```bash
$ yum list --showduplicates kubeadm --disableexcludes=kubernetes

```

### 2.1.1 升级控制节点

1、查看当前版本和升级计划（即可以从目前版本升级到哪个版本）

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:58:59Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:50:46Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}

$ kubeadm upgrade plan
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       AVAILABLE
Kubelet     3 x v1.18.0   v1.18.18

Upgrade to the latest version in the v1.18 series:

COMPONENT            CURRENT   AVAILABLE
API Server           v1.18.0   v1.18.18
Controller Manager   v1.18.0   v1.18.18
Scheduler            v1.18.0   v1.18.18
Kube Proxy           v1.18.0   v1.18.18
CoreDNS              1.6.7     1.6.7
Etcd                 3.4.3     3.4.3-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.18.18

Note: Before you can perform this upgrade, you have to update kubeadm to v1.18.18.

```

> 说明：`kubeadm upgrade` 也会自动对 kubeadm 在节点上所管理的证书执行续约操作。 如果需要略过证书续约操作，可以使用标志 `--certificate-renewal=false`。



2、升级kubeadm

```bash
$ yum install -y kubeadm-1.18.18-0 --disableexcludes=kubernetes
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.18", GitCommit:"6f6ce59dc8fefde25a3ba0ef0047f4ec6662ef24", GitTreeState:"clean", BuildDate:"2021-04-15T03:29:14Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}

查看升级后所需要的镜像
$ kubeadm config images list
k8s.gcr.io/kube-apiserver:v1.18.18
k8s.gcr.io/kube-controller-manager:v1.18.18
k8s.gcr.io/kube-scheduler:v1.18.18
k8s.gcr.io/kube-proxy:v1.18.18
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7

```

确保上面的容器镜像已经下载（如果没有提前下载，可能被网络阻隔导致挂起），然后执行升级：

```bash
$ kubectl drain k8s-master40 --ignore-daemonsets
$ kubeadm upgrade apply v1.18.18
$ kubectl uncordon k8s-master40
```

看到下面信息，就OK了。

```bash
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.18.18". Enjoy!
```

再次查看版本

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.0", GitCommit:"9e991415386e4cf155a24b1da15becaa390438d8", GitTreeState:"clean", BuildDate:"2020-03-25T14:58:59Z", GoVersion:"go1.13.8", Compiler:"gc", Platform:"linux/amd64"}
Server Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.18", GitCommit:"6f6ce59dc8fefde25a3ba0ef0047f4ec6662ef24", GitTreeState:"clean", BuildDate:"2021-04-15T03:23:41Z", GoVersion:"go1.13.15", Compiler:"gc", Platform:"linux/amd64"}

```

可以看到，虽然kubectl还是1.18.0，服务端的控制平面已经升级到了1.18.18，但是查看Node版本，会发现Node版本还是滞后的： 

```bash
$ kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master40   Ready    master   30d   v1.18.0   #此处版本低
k8s-node41     Ready    <none>   28d   v1.18.0
k8s-node42     Ready    <none>   85m   v1.18.0
```

3、升级kubelet和kubectl

```bash
yum install -y kubelet-1.18.18-0 kubectl-1.18.18-0 --disableexcludes=kubernetes
systemctl daemon-reload
systemctl restart kubelet
```

查看版本

```bash
$ kubectl get nodes
NAME           STATUS   ROLES    AGE    VERSION
k8s-master40   Ready    master   30d    v1.18.18
k8s-node41     Ready    <none>   28d    v1.18.0
k8s-node42     Ready    <none>   129m   v1.18.0

```



### 2.1.2 升级工作节点

```bash
$ yum install -y kubeadm-1.18.18-0 --disableexcludes=kubernetes
$ kubectl drain k8s-node41 --ignore-daemonsets
$ kubeadm upgrade node
$ yum install -y kubelet-1.18.18-0 kubectl-1.18.18-0 --disableexcludes=kubernetes
$ systemctl daemon-reload
$ systemctl restart kubelet
$ kubectl uncordon k8s-node41
$ kubectl get nodes
NAME           STATUS   ROLES    AGE    VERSION
k8s-master40   Ready    master   30d    v1.18.18
k8s-node41     Ready    <none>   29d    v1.18.18
k8s-node42     Ready    <none>   138m   v1.18.0
```

