





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



### 3.1.1 启动 etcd 集群

本节介绍如何启动单节点和多节点 etcd 集群。

**1、单节点 etcd 集群**

只为测试目的使用单节点 etcd 集群。

1. 运行以下命令：

   ```sh
   etcd --listen-client-urls=http://$PRIVATE_IP:2379 \
      --advertise-client-urls=http://$PRIVATE_IP:2379
   ```

2. 使用参数 `--etcd-servers=$PRIVATE_IP:2379` 启动 Kubernetes API 服务器。

   确保将 `PRIVATE_IP` 设置为 etcd 客户端 IP。

**2、多节点 etcd 集群**

出于耐用性和高可用性考量，在生产环境中应以多节点集群的方式运行 etcd，并且定期备份。 建议在生产环境中使用五个成员的集群。 有关该内容的更多信息，请参阅[常见问题文档](https://etcd.io/docs/current/faq/#what-is-failure-tolerance)。

可以通过静态成员信息或动态发现的方式配置 etcd 集群。 有关集群的详细信息，请参阅 [etcd 集群文档](https://etcd.io/docs/current/op-guide/clustering/)。

例如，考虑运行以下客户端 URL 的五个成员的 etcd 集群：`http://$IP1:2379`、 `http://$IP2:2379`、`http://$IP3:2379`、`http://$IP4:2379` 和 `http://$IP5:2379`。 要启动 Kubernetes API 服务器：

1. 运行以下命令：

   ```shell
   etcd --listen-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379 --advertise-client-urls=http://$IP1:2379,http://$IP2:2379,http://$IP3:2379,http://$IP4:2379,http://$IP5:2379
   ```

2. 使用参数 `--etcd-servers=$IP1:2379,$IP2:2379,$IP3:2379,$IP4:2379,$IP5:2379` 启动 Kubernetes API 服务器。

   确保将 `IP<n>` 变量设置为客户端 IP 地址。

**3、使用负载均衡器的多节点 etcd 集群**

要运行负载均衡的 etcd 集群：

1. 建立一个 etcd 集群。
2. 在 etcd 集群前面配置负载均衡器。例如，让负载均衡器的地址为 `$LB`。
3. 使用参数 `--etcd-servers=$LB:2379` 启动 Kubernetes API 服务器。

# 4 加固 etcd 集群

对 etcd 的访问相当于集群中的 root 权限，因此理想情况下只有 API 服务器才能访问它。 考虑到数据的敏感性，建议只向需要访问 etcd 集群的节点授予权限。

想要确保 etcd 的安全，可以设置防火墙规则或使用 etcd 提供的安全特性，这些安全特性依赖于 x509 公钥基础设施（PKI）。 首先，通过生成密钥和证书对来建立安全的通信通道。 例如，使用密钥对 `peer.key` 和 `peer.cert` 来保护 etcd 成员之间的通信， 而 `client.key` 和 `client.cert` 用于保护 etcd 与其客户端之间的通信。 请参阅 etcd 项目提供的[示例脚本](https://github.com/coreos/etcd/tree/master/hack/tls-setup)， 以生成用于客户端身份验证的密钥对和 CA 文件。

## 4.1 安全通信

若要使用安全对等通信对 etcd 进行配置，请指定参数 `--peer-key-file=peer.key` 和 `--peer-cert-file=peer.cert`，并使用 HTTPS 作为 URL 模式。

类似地，要使用安全客户端通信对 etcd 进行配置，请指定参数 `--key-file=k8sclient.key` 和 `--cert-file=k8sclient.cert`，并使用 HTTPS 作为 URL 模式。 使用安全通信的客户端命令的示例：

```
ETCDCTL_API=3 etcdctl --endpoints 10.2.0.9:2379 \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  member list
```

## 4.2 限制 etcd 集群的访问



# 5 替换失败的 etcd 成员

etcd 集群通过容忍少数成员故障实现高可用性。 但是，要改善集群的整体健康状况，请立即替换失败的成员。当多个成员失败时，逐个替换它们。 替换失败成员需要两个步骤：删除失败成员和添加新成员。

虽然 etcd 在内部保留唯一的成员 ID，但建议为每个成员使用唯一的名称，以避免人为错误。 例如，考虑一个三成员的 etcd 集群。假定 URL 分别为：`member1=http://10.0.0.1`、`member2=http://10.0.0.2` 和 `member3=http://10.0.0.3`。当 `member1` 失败时，将其替换为 `member4=http://10.0.0.4`。

1、获取失败的 `member1` 的成员 ID：

```shell
etcdctl --endpoints=http://10.0.0.2,http://10.0.0.3 member list
```

显示以下信息：

```console
8211f1d0f64f3269, started, member1, http://10.0.0.1:2380, http://10.0.0.1:2379
91bc3c398fb3c146, started, member2, http://10.0.0.2:2380, http://10.0.0.2:2379
fd422379fda50e48, started, member3, http://10.0.0.3:2380, http://10.0.0.3:2379
```

2、执行以下操作之一：

（1）如果每个 Kubernetes API 服务器都配置为与所有 etcd 成员通信， 请从 `--etcd-servers` 标志中移除删除失败的成员，然后重新启动每个 Kubernetes API 服务器。

（2）如果每个 Kubernetes API 服务器都与单个 etcd 成员通信， 则停止与失败的 etcd 通信的 Kubernetes API 服务器。

3、止故障节点上的 etcd 服务器。除了 Kubernetes API 服务器之外的其他客户端可能会造成流向 etcd 的流量， 可以停止所有流量以防止写入数据目录。

4、移除失败的成员：

```shell
etcdctl member remove 8211f1d0f64f3269
```

显示以下信息：

```console
Removed member 8211f1d0f64f3269 from cluster
```

5、增加新成员：

```shell
etcdctl member add member4 --peer-urls=http://10.0.0.4:2380
```

显示以下信息：

```console
Member 2be1eb8f84b7f63e added to cluster ef37ad9dc622a7c4
```

6、在 IP 为 `10.0.0.4` 的机器上启动新增加的成员：

```shell
export ETCD_NAME="member4"
export ETCD_INITIAL_CLUSTER="member2=http://10.0.0.2:2380,member3=http://10.0.0.3:2380,member4=http://10.0.0.4:2380"
export ETCD_INITIAL_CLUSTER_STATE=existing
etcd [flags]
```

7、执行以下操作之一：

（1）如果每个 Kubernetes API 服务器都配置为与所有 etcd 成员通信， 则将新增的成员添加到 `--etcd-servers` 标志，然后重新启动每个 Kubernetes API 服务器。

（2）如果每个 Kubernetes API 服务器都与单个 etcd 成员通信，请启动在第 2 步中停止的 Kubernetes API 服务器。 然后配置 Kubernetes API 服务器客户端以再次将请求路由到已停止的 Kubernetes API 服务器。 这通常可以通过配置负载均衡器来完成。

有关集群重新配置的详细信息，请参阅 [etcd 重构文档](https://etcd.io/docs/current/op-guide/runtime-configuration/#remove-a-member)。



# 6 备份恢复 etcd 集群

所有 Kubernetes 对象都存储在 etcd 上。 定期备份 etcd 集群数据对于在灾难场景（例如丢失所有控制平面节点）下恢复 Kubernetes 集群非常重要。 快照文件包含所有 Kubernetes 状态和关键信息。为了保证敏感的 Kubernetes 数据的安全，可以对快照文件进行加密。

备份 etcd 集群可以通过两种方式完成：==etcd 内置快照和卷快照==。

## 6.1 内置快照

etcd 支持内置快照。快照可以从使用 `etcdctl snapshot save` 命令的活动成员中获取， 也可以通过从 etcd [数据目录](https://etcd.io/docs/current/op-guide/configuration/#--data-dir) 复制 `member/snap/db` 文件，该 etcd 数据目录目前没有被 etcd 进程使用。获取快照不会影响成员的性能。

下面是一个示例，用于获取 `$ENDPOINT` 所提供的键空间的快照到文件 `snapshotdb`：

```shell
ETCDCTL_API=3 etcdctl --endpoints $ENDPOINT snapshot save snapshotdb
```

执行备份：

```bash
[root@k8s-m01 ~]# ETCDCTL_API=3 /usr/local/bin/etcdctl   --cacert=/etc/etcd/pki/ca.pem   --cert=/etc/etcd/pki/server.pem   --key=/etc/etcd/pki/server-key.pem   --endpoints="https://10.159.238.10:2379" snapshot save snapshotdb
{"level":"info","ts":1679951140.9955761,"caller":"snapshot/v3_snapshot.go:110","msg":"created temporary db file","path":"snapshotdb.part"}
{"level":"info","ts":1679951140.9993742,"caller":"snapshot/v3_snapshot.go:121","msg":"fetching snapshot","endpoint":"https://10.159.238.10:2379"}
{"level":"info","ts":1679951141.1569984,"caller":"snapshot/v3_snapshot.go:134","msg":"fetched snapshot","endpoint":"https://10.159.238.10:2379","took":0.161248593}
{"level":"info","ts":1679951141.157167,"caller":"snapshot/v3_snapshot.go:143","msg":"saved","path":"snapshotdb"}
Snapshot saved at snapshotdb
```

验证快照:

```shell
[root@k8s-m01 ~]# ETCDCTL_API=3 etcdctl --write-out=table snapshot status snapshotdb
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| c9697f07 |   141239 |       1792 |     5.4 MB |
+----------+----------+------------+------------+
```





## 6.2 卷快照

如果 etcd 运行在支持备份的存储卷（如 Amazon Elastic Block 存储）上，则可以通过获取存储卷的快照来备份 etcd 数据。

使用 etcdctl 选项的快照

我们还可以使用 etcdctl 提供的各种选项来制作快照。例如：

```shell
ETCDCTL_API=3 etcdctl -h 
```

列出 etcdctl 可用的各种选项。例如，你可以通过指定端点、证书等来制作快照，如下所示：

```shell
ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=<trusted-ca-file> --cert=<cert-file> --key=<key-file> \
  snapshot save <backup-file-location>
```

可以从 etcd Pod 的描述中获得 `trusted-ca-file`、`cert-file` 和 `key-file`。





## 6.3 恢复etcd

etcd 支持从 [major.minor](http://semver.org/) 或其他不同 patch 版本的 etcd 进程中获取的快照进行恢复。 还原操作用于恢复失败的集群的数据。

在启动还原操作之前，必须有一个快照文件。它可以是来自以前备份操作的快照文件， 也可以是来自剩余[数据目录](https://etcd.io/docs/current/op-guide/configuration/#--data-dir)的快照文件。 例如：

```shell
ETCDCTL_API=3 etcdctl --endpoints 10.2.0.9:2379 snapshot restore snapshotdb
```

恢复时也可以指定操作选项，例如：

```shell
ETCDCTL_API=3 etcdctl snapshot restore --data-dir <data-dir-location> snapshotdb
```

有关从快照文件还原集群的详细信息和示例，请参阅 [etcd 灾难恢复文档](https://etcd.io/docs/current/op-guide/recovery/#restoring-a-cluster)。

如果还原的集群的访问 URL 与前一个集群不同，则必须相应地重新配置 Kubernetes API 服务器。 在本例中，使用参数 `--etcd-servers=$NEW_ETCD_CLUSTER` 而不是参数 `--etcd-servers=$OLD_ETCD_CLUSTER` 重新启动 Kubernetes API 服务器。用相应的 IP 地址替换 `$NEW_ETCD_CLUSTER` 和 `$OLD_ETCD_CLUSTER`。 如果在 etcd 集群前面使用负载均衡，则可能需要更新负载均衡器。

如果大多数 etcd 成员永久失败，则认为 etcd 集群失败。在这种情况下，Kubernetes 不能对其当前状态进行任何更改。 虽然已调度的 Pod 可能继续运行，但新的 Pod 无法调度。在这种情况下， 恢复 etcd 集群并可能需要重新配置 Kubernetes API 服务器以修复问题。

**说明：**

如果集群中正在运行任何 API 服务器，则不应尝试还原 etcd 的实例。相反，请按照以下步骤还原 etcd：

- 停止**所有** API 服务实例
- 在所有 etcd 实例中恢复状态
- 重启所有 API 服务实例

我们还建议重启所有组件（例如 `kube-scheduler`、`kube-controller-manager`、`kubelet`）， 以确保它们不会依赖一些过时的数据。请注意，实际中还原会花费一些时间。 在还原过程中，关键组件将丢失领导锁并自行重启。



# 7 升级 etcd 集群

有关 etcd 升级的更多详细信息，请参阅 [etcd 升级](https://etcd.io/docs/latest/upgrades/)文档。

**说明：**

在开始升级之前，请先备份你的 etcd 集群。





# 背景：由于机房异常断电导致，etcd数据损坏：

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



3、Etcd数据库备份与恢复

查看集群状态

```bash
[root@k8s-master ~]# /opt/etcd/bin/etcdctl --help

      --cacert=""       verify certificates of TLS-enabled secure servers using this CA bundle
      --cert=""         identify secure client using this TLS certificate file
      --key=""          identify secure client using this TLS key file
      --endpoints=[127.0.0.1:2379]    gRPC endpoints

[root@k8s-master ~]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.179.99:2379,https://192.168.179.100:2379,https://192.168.179.101:2379" member list
1cd5f52adf869d89, started, etcd-1, https://192.168.179.99:2380, https://192.168.179.99:2379, false
55857deef69d787b, started, etcd-2, https://192.168.179.100:2380, https://192.168.179.100:2379, false
8bcf42695ccd8d89, started, etcd-3, https://192.168.179.101:2380, https://192.168.179.101:2379, false


[root@k8s-master ~]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://192.168.179.99:2379,https://192.168.179.100:2379,https://192.168.179.101:2379" endpoint health
https://192.168.179.100:2379 is healthy: successfully committed proposal: took = 33.373965ms
https://192.168.179.101:2379 is healthy: successfully committed proposal: took = 41.146436ms
https://192.168.179.99:2379 is healthy: successfully committed proposal: took = 41.593452ms
```

这三个节点的信息是相互同步的，要去备份只需要备份一个节点就行了，连接其中一个节点备份就行。

```bash
ETCDCTL_API=3 /opt/etcd/bin/etcdctl \
snapshot save snap.db \
--endpoints=https://192.168.179.99:2379 \
--cacert=/opt/etcd/ssl/ca.pem \
--cert=/opt/etcd/ssl/server.pem \
--key=/opt/etcd/ssl/server-key.pem

[root@k8s-master ~]# ETCDCTL_API=3 etcdctl \
> snapshot save snap.db \
> --endpoints=https://192.168.179.99:2379 \
> --cacert=/opt/etcd/ssl/ca.pem \
> --cert=/opt/etcd/ssl/server.pem \
> --key=/opt/etcd/ssl/server-key.pem
{"level":"info","ts":1608451206.8816888,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"snap.db.part"}
{"level":"info","ts":"2020-12-20T16:00:06.895+0800","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1608451206.8958433,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"https://192.168.179.99:2379"}
{"level":"info","ts":"2020-12-20T16:00:07.222+0800","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1608451207.239597,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"https://192.168.179.99:2379","size":"3.4 MB","took":0.357763211}
{"level":"info","ts":1608451207.2398226,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"snap.db"}
Snapshot saved at snap.db

[root@k8s-master ~]# ll /opt/etcd/ssl/
total 16
-rw------- 1 root root 1679 Sep 15 11:37 ca-key.pem
-rw-r--r-- 1 root root 1265 Sep 15 11:37 ca.pem
-rw------- 1 root root 1675 Sep 15 11:37 server-key.pem
-rw-r--r-- 1 root root 1338 Sep 15 11:37 server.pem
```



```
[root@k8s-master ~]# kubectl get deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
kubia   3/3     3            3           142d

[root@k8s-master ~]# kubectl create deployment nginx --image=nginx
deployment.apps/nginx created

[root@k8s-master ~]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
kubia-859d757f8c-74g6s   1/1     Running   0          142d
kubia-859d757f8c-97znt   1/1     Running   0          142d
kubia-859d757f8c-9mjf9   1/1     Running   0          142d
nginx-f89759699-jttrw    1/1     Running   0          49s
```

现在需要恢复了，对所有的etcd节点都做暂停。如果是多master那么上面apisrevr都要停止

1.先暂停kube-apiserver和etcd

```bash
[root@k8s-master ~]# systemctl stop kube-apiserver

[root@k8s-master ~]# systemctl stop etcd
[root@k8s-node1 ~]# systemctl stop etcd
[root@k8s-node2 ~]# systemctl stop etcd
```

 2.在每个节点上恢复

先来看看我的配置

```bash
[root@k8s-master ~]# cat /opt/etcd/cfg/etcd.conf 
#[Member]
ETCD_NAME="etcd-1"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.179.99:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.179.99:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.179.99:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.179.99:2379"
ETCD_INITIAL_CLUSTER="etcd-1=https://192.168.179.99:2380,etcd-2=https://192.168.179.100:2380,etcd-3=https://192.168.179.101:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

在第一个节点恢复

```bash
ETCDCTL_API=3 etcdctl snapshot restore /root/snap.db \
--name etcd-1 \
--initial-cluster="etcd-1=https://192.168.179.99:2380,etcd-2=https://192.168.179.100:2380,etcd-3=https://192.168.179.101:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.179.99:2380 \
--data-dir=/var/lib/etcd/default.etcd


--name etcd-1 \   #需要修改为当前节点名称
--initial-advertise-peer-urls=https://192.168.179.99:2380 \  #当前节点IP


[root@k8s-master ~]# ETCDCTL_API=3 etcdctl snapshot restore /root/snap.db \
> --name etcd-1 \
> --initial-cluster="etcd-1=https://192.168.179.99:2380,etcd-2=https://192.168.179.100:2380,etcd-3=https://192.168.179.101:2380" \
> --initial-cluster-token=etcd-cluster \
> --initial-advertise-peer-urls=https://192.168.179.99:2380 \
> --data-dir=/var/lib/etcd/default.etcd
{"level":"info","ts":1608453271.6452653,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/root/snap.db","wal-dir":"/var/lib/etcd/default.etcd/member/wal","data-dir":"/var/lib/etcd/default.etcd","snap-dir":"/var/lib/etcd/default.etcd/member/snap"}
{"level":"info","ts":1608453271.7769744,"caller":"mvcc/kvstore.go:380","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":93208}
{"level":"info","ts":1608453271.8183022,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"1b21d5d68d61885a","local-member-id":"0","added-peer-id":"1cd5f52adf869d89","added-peer-peer-urls":["https://192.168.179.99:2380"]}
{"level":"info","ts":1608453271.8184474,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"1b21d5d68d61885a","local-member-id":"0","added-peer-id":"55857deef69d787b","added-peer-peer-urls":["https://192.168.179.100:2380"]}
{"level":"info","ts":1608453271.818473,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"1b21d5d68d61885a","local-member-id":"0","added-peer-id":"8bcf42695ccd8d89","added-peer-peer-urls":["https://192.168.179.101:2380"]}
{"level":"info","ts":1608453271.8290143,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/root/snap.db","wal-dir":"/var/lib/etcd/default.etcd/member/wal","data-dir":"/var/lib/etcd/default.etcd","snap-dir":"/var/lib/etcd/default.etcd/member/snap"}


[root@k8s-master ~]# ls /var/lib/etcd/
default.etcd  default.etcd.bak
```

拷贝到其他节点，再去恢复

```
[root@k8s-master ~]# scp snap.db root@192.168.179.100:~
root@192.168.179.100's password: 
snap.db                                                                                           100% 3296KB  15.4MB/s   00:00    
[root@k8s-master ~]# scp snap.db root@192.168.179.101:~
root@192.168.179.101's password: 
snap.db   
```

在二节点恢复

```
[root@k8s-node1 ~]# ls /var/lib/etcd/
default.etcd.bak

ETCDCTL_API=3 etcdctl snapshot restore /root/snap.db \
--name etcd-2 \
--initial-cluster="etcd-1=https://192.168.179.99:2380,etcd-2=https://192.168.179.100:2380,etcd-3=https://192.168.179.101:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.179.100:2380 \
--data-dir=/var/lib/etcd/default.etcd

[root@k8s-node1 ~]# ls /var/lib/etcd/
default.etcd  default.etcd.bak
```

在三节点恢复

```
ETCDCTL_API=3 etcdctl snapshot restore /root/snap.db \
--name etcd-3 \
--initial-cluster="etcd-1=https://192.168.179.99:2380,etcd-2=https://192.168.179.100:2380,etcd-3=https://192.168.179.101:2380" \
--initial-cluster-token=etcd-cluster \
--initial-advertise-peer-urls=https://192.168.179.101:2380 \
--data-dir=/var/lib/etcd/default.etcd
```

现在恢复成功，下面将服务启动

登录后复制 

```
[root@k8s-master ~]# systemctl start kube-apiserver

[root@k8s-master ~]# systemctl start etcd
[root@k8s-node1 ~]# systemctl start etcd
[root@k8s-node2 ~]# systemctl start etcd
```

启动完看看集群是否正常

```
[root@k8s-master ~]#  ETCDCTL_API=3 etcdctl --cacert=/opt/etcd/ssl/ca.pem  --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem  --endpoints="https://192.168.179.99:2379,https://192.168.179.100:2379,https://192.168.179.101:2379" endpoint health
https://192.168.179.100:2379 is healthy: successfully committed proposal: took = 25.946686ms
https://192.168.179.99:2379 is healthy: successfully committed proposal: took = 27.290324ms
https://192.168.179.101:2379 is healthy: successfully committed proposal: took = 30.621904ms
```

```
[root@k8s-master ~]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
kubia-859d757f8c-74g6s   1/1     Running   0          142d
kubia-859d757f8c-97znt   1/1     Running   0          142d
kubia-859d757f8c-9mjf9   1/1     Running   0          142d
```

可以看到之前的nginx消失了，即数据恢复成功。

之前备份是找了其中一个节点去备份的，找任意节点去备份都行，但是建议找两个节点去备份，如果其中一个节点挂了，那么备份就会失败了。

注意在每个节点进行恢复，一个是恢复数据，一个是重塑身份

























