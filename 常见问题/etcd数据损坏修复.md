





# etcd数据损坏修复

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
