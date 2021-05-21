

# k8s使用常见问题

# 1 node(s) had taints that the pod didn't tolerate

直译意思是节点有了污点无法容忍，执行kubectl get no -o yaml | grep taint -A 5 之后发现该节点是不可调度的。这是因为kubernetes使用kubeadm初始化的集群，出于安全考虑Pod不会被调度到Master Node上，也就是说Master Node不参与工作负载。

允许master节点部署pod，使用命令如下:

```bash
# 查看信息
kubectl get no -o yaml | grep taint -A 5
kubectl taint nodes --all node-role.kubernetes.io/master-
```

输出如下:

node “k8s” untainted

输出error: taint “node-role.kubernetes.io/master:” not found错误忽略。

禁止master部署pod

```
kubectl taint nodes k8s node-role.kubernetes.io/master=true:NoSchedule
```



# 2 pod has unbound immediate PersistentVolumeClaims







# 3 no matches for kind "DaemonSet" in version "extensions/v1beta1"

DaemonSet、Deployment、StatefulSet 和 ReplicaSet 在 v1.16 中将不再从 extensions/v1beta1、apps/v1beta1 或 apps/v1beta2 提供服务。

解决方法：

将yaml配置文件内的api接口修改为 apps/v1 ，导致原因为之间使用的kubernetes 版本是1.14.x版本，1.16.x 版本放弃部分API支持。







# 4 kubelet  Back-off restarting failed container

在通过glusterfs-daemonset.json部署glusterfs时，查看pod是不是就重启报错Back-off restarting failed container。

需要在image后加入如下内容：

```json
 "image": "gluster/gluster-centos:latest",
            "command": [
              "/bin/bash",
              "-ce",
              "tail -f /dev/null"
            ],
```





# 5 Failed to get D-Bus connection: Operation not permitted

Liveness probe failed: Failed to get D-Bus connection: Operation not permitted

Readiness probe failed: Failed to get D-Bus connection: Operation not permitted

