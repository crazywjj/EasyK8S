





# Etcd管理

etcd 是 一致且高可用的键值存储，用作 Kubernetes 所有集群数据的后台数据库。



# 1 先决条件

- 运行的 etcd 集群个数成员为奇数。

- etcd 是一个 leader-based 分布式系统。确保主节点定期向所有从节点发送心跳，以保持集群稳定。

- 确保不发生资源不足。

  集群的性能和稳定性对网络和磁盘 I/O 非常敏感。任何资源匮乏都会导致心跳超时， 从而导致集群的不稳定。不稳定的情况表明没有选出任何主节点。 在这种情况下，集群不能对其当前状态进行任何更改，这意味着不能调度新的 Pod。

- 保持 etcd 集群的稳定对 Kubernetes 集群的稳定性至关重要。 因此，请在专用机器或隔离环境上运行 etcd 集群， 以满足[所需资源需求](https://etcd.io/docs/current/op-guide/hardware/)。

- 在生产中运行的 etcd 的最低推荐版本是 `3.2.10+`。





# 2 资源需求

使用有限的资源运行 etcd 只适合测试目的。为了在生产中部署，需要先进的硬件配置。 在生产中部署 etcd 之前，请查看[所需资源参考文档](https://etcd.io/docs/current/op-guide/hardware/#example-hardware-configurations)。





# 3 使用 kubeadm 创建一个高可用 etcd 集群

> **说明：**在本指南中，使用 kubeadm 作为外部 etcd 节点管理工具，请注意 kubeadm 不计划支持此类节点的证书更换或升级。 对于长期规划是使用 [etcdadm](https://github.com/kubernetes-sigs/etcdadm) 增强工具来管理这些方面。



默认情况下，kubeadm 在每个控制平面节点上运行一个本地 etcd 实例。也可以使用外部的 etcd 集群，并在不同的主机上提供 etcd 实例。 这两种方法的区别在 [高可用拓扑的选项](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology) 页面中阐述。

这个任务将指导你创建一个由三个成员组成的高可用外部 etcd 集群，该集群在创建过程中可被 kubeadm 使用。



## 3.1 准备开始

- 三个可以通过 2379 和 2380 端口相互通信的主机。本文档使用这些作为默认端口。不过，它们可以通过 kubeadm 的配置文件进行自定义。

- 每个主机必须安装 systemd 和 bash 兼容的 shell。
- 每台主机必须[安装有容器运行时、kubelet 和 kubeadm](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)。

- 每个主机都应该能够访问 Kubernetes 容器镜像仓库 (registry.k8s.io)， 或者使用 `kubeadm config images list/pull` 列出/拉取所需的 etcd 镜像。 本指南将把 etcd 实例设置为由 kubelet 管理的[静态 Pod](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/static-pod/)。

- 一些可以用来在主机间复制文件的基础设施。例如 `ssh` 和 `scp` 就可以满足需求。



## 3.1 建立集群



1. 将 kubelet 配置为 etcd 的服务管理器。



**说明：** 你必须在要运行 etcd 的所有主机上执行此操作。

由于 etcd 是首先创建的，因此你必须通过创建具有更高优先级的新文件来覆盖 kubeadm 提供的 kubelet 单元文件。



```sh
cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf
[Service]
ExecStart=
# 将下面的 "systemd" 替换为你的容器运行时所使用的 cgroup 驱动。
# kubelet 的默认值为 "cgroupfs"。
# 如果需要的话，将 "--container-runtime-endpoint " 的值替换为一个不同的容器运行时。
ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock
Restart=always
EOF

systemctl daemon-reload
systemctl restart kubelet
```

检查 kubelet 的状态以确保其处于运行状态：

```shell
systemctl status kubelet
```





2. 为 kubeadm 创建配置文件。

使用以下脚本为每个将要运行 etcd 成员的主机生成一个 kubeadm 配置文件。

```sh
# 使用你的主机 IP 替换 HOST0、HOST1 和 HOST2 的 IP 地址
export HOST0=10.159.238.10
export HOST1=10.159.238.11
export HOST2=10.159.238.12

# 使用你的主机名更新 NAME0、NAME1 和 NAME2
export NAME0="k8s-m01"
export NAME1="k8s-m02"
export NAME2="k8s-m03"

# 创建临时目录来存储将被分发到其它主机上的文件
mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/

HOSTS=(${HOST0} ${HOST1} ${HOST2})
NAMES=(${NAME0} ${NAME1} ${NAME2})

for i in "${!HOSTS[@]}"; do
HOST=${HOSTS[$i]}
NAME=${NAMES[$i]}
cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: InitConfiguration
nodeRegistration:
    name: ${NAME}
localAPIEndpoint:
    advertiseAddress: ${HOST}
---
apiVersion: "kubeadm.k8s.io/v1beta3"
kind: ClusterConfiguration
etcd:
    local:
        serverCertSANs:
        - "${HOST}"
        peerCertSANs:
        - "${HOST}"
        extraArgs:
            initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380
            initial-cluster-state: new
            name: ${NAME}
            listen-peer-urls: https://${HOST}:2380
            listen-client-urls: https://${HOST}:2379
            advertise-client-urls: https://${HOST}:2379
            initial-advertise-peer-urls: https://${HOST}:2380
EOF
done
```



3. 生成证书颁发机构

如果你已经拥有 CA，那么唯一的操作是复制 CA 的 `crt` 和 `key` 文件到 `etc/kubernetes/pki/etcd/ca.crt` 和 `/etc/kubernetes/pki/etcd/ca.key`。 复制完这些文件后继续下一步，“为每个成员创建证书”。

如果你还没有 CA，则在 `$HOST0`（你为 kubeadm 生成配置文件的位置）上运行此命令。

```shell
kubeadm init phase certs etcd-ca
```

这一操作创建如下两个文件：

- `/etc/kubernetes/pki/etcd/ca.crt`
- `/etc/kubernetes/pki/etcd/ca.key`



4. 为每个成员创建证书

```shell
kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST2}/
# 清理不可重复使用的证书
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml
cp -R /etc/kubernetes/pki /tmp/${HOST1}/
find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete

kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml
# 不需要移动 certs 因为它们是给 HOST0 使用的

# 清理不应从此主机复制的证书
find /tmp/${HOST2} -name ca.key -type f -delete
find /tmp/${HOST1} -name ca.key -type f -delete
```



5. 复制证书和 kubeadm 配置

证书已生成，现在必须将它们移动到对应的主机。

```shell
USER=root
HOST=${HOST1}
scp -r /tmp/${HOST}/* ${USER}@${HOST}:
ssh ${USER}@${HOST}
USER@HOST $ sudo -Es
root@HOST $ chown -R root:root pki
root@HOST $ mv pki /etc/kubernetes/
```



6. 确保已经所有预期的文件都存在

`$HOST0` 所需文件的完整列表如下：

```none
/tmp/${HOST0}
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── ca.key
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

在 `$HOST1` 上：

```console
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```

在 `$HOST2` 上：

```console
$HOME
└── kubeadmcfg.yaml
---
/etc/kubernetes/pki
├── apiserver-etcd-client.crt
├── apiserver-etcd-client.key
└── etcd
    ├── ca.crt
    ├── healthcheck-client.crt
    ├── healthcheck-client.key
    ├── peer.crt
    ├── peer.key
    ├── server.crt
    └── server.key
```



7. 创建静态 Pod 清单

既然证书和配置已经就绪，是时候去创建清单了。 在每台主机上运行 `kubeadm` 命令来生成 etcd 使用的静态清单。

```shell
root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml
root@HOST1 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
root@HOST2 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml
```



8. 可选：检查集群运行状况

如果 `etcdctl` 不可用，你可以在容器镜像内运行此工具。 你可以使用 `crictl run` 这类工具直接在容器运行时执行此操作，而不是通过 Kubernetes。

```sh
ETCDCTL_API=3 etcdctl \
--cert /etc/kubernetes/pki/etcd/peer.crt \
--key /etc/kubernetes/pki/etcd/peer.key \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--endpoints https://${HOST0}:2379 endpoint health
...
https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms
https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms
https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms
```

- 将 `${HOST0}` 设置为要测试的主机的 IP 地址。



















**背景：**由于机房异常断电导致，etcd数据损坏，信息如下：

报错信息：

1、执行`kubectl get node` 时报错

```
Error from server: etcdserver: request timed out
```

2、执行`systemctl start etcd`启动etcd 查看etcd日志时（`journal -fu etcd`)，报错信息：

```
Job for etcd.service failed because the control process exited with error code. See "systemctl status etcd.service" and "journalctl -xe" for details.
```

详细日志：

```
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_NAME, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_DATA_DIR, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_LISTEN_PEER_URLS, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_LISTEN_CLIENT_URLS, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_INITIAL_ADVERTISE_PEER_URLS, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_ADVERTISE_CLIENT_URLS, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_INITIAL_CLUSTER, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_INITIAL_CLUSTER_TOKEN, but unused: shadowed by corresponding flag
Mar 28 10:39:19 k8s-marster1 etcd[2581]: recognized environment variable ETCD_INITIAL_CLUSTER_STATE, but un
```









etcd会在默认的工作目录下生成两个子目录：snap和wal。两个目录的作用说明如下：

snap：用于存放快照数据。etcd为了防止WAL文件过多就会创建快照，snap用于存储etcd的快照数据状态
wal：用于存放预写式日志，其最大的作用是记录整个数据变化的全部历程。在etcd中，所有数据的修改在提交之前，都要写入WAL中。使用WAL进行数据的存储使得etcd拥有故障快速恢复和数据回滚两个重要的功能
故障快速恢复：如果你的数据遭到颇快，就可以通过执行所有WAL中记录的修改操作，快速从原始的数据恢复到数据损坏之前的状态

数据回滚（undo）/重做（redo）：因为所有的修改操作都被记录在WAL中，所以进行回滚或者重做时，只需要反响或者正向执行日志即可
