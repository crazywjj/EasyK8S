[TOC]





# kube-state-metrics



# 1、概述

已经有了cadvisor、heapster、metric-server，几乎容器运行的所有指标都能拿到，但是下面这种情况却无能为力：

- 我调度了多少个replicas？现在可用的有几个？
- 多少个Pod是running/stopped/terminated状态？
- Pod重启了多少次？
- 我有多少job在运行中

而这些则是kube-state-metrics提供的内容，它基于client-go开发，轮询Kubernetes API，并将Kubernetes的结构化信息转换为metrics。

# 2、功能

kube-state-metrics提供的指标，按照阶段分为三种类别：

- 实验性质的：k8s api中alpha阶段的或者spec的字段。
- 稳定版本的：k8s中不向后兼容的主要版本的更新
- 被废弃的：已经不在维护的。

**指标类别包括：**

```bash
CronJob Metrics
DaemonSet Metrics
Deployment Metrics
Job Metrics
LimitRange Metrics
Node Metrics
PersistentVolume Metrics
PersistentVolumeClaim Metrics
Pod Metrics
Pod Disruption Budget Metrics
ReplicaSet Metrics
ReplicationController Metrics
ResourceQuota Metrics
Service Metrics
StatefulSet Metrics
Namespace Metrics
Horizontal Pod Autoscaler Metrics
Endpoint Metrics
Secret Metrics
ConfigMap Metrics
```

**以pod为例：**

```
kube_pod_info
kube_pod_owner
kube_pod_status_phase
kube_pod_status_ready
kube_pod_status_scheduled
kube_pod_container_status_waiting
kube_pod_container_status_terminated_reason
...
```

==kube-state-metrics与metric-server (或heapster)的对比：==

1. metric-server是从api-server中获取cpu,内存使用率这种监控指标，并把它们发送给存储后端，如influxdb或云厂商，它当前的核心作用是：为HPA等组件提供决策指标支持；
2. kube-state-metrics关注于获取k8s各种资源的最新状态，如deployment或者daemonset，之所以没有把kube-state-metrics纳入到metric-server的能力中，是因为它们的关注点本质上是不一样的。metric-server仅仅是获取、格式化现有数据，写入特定的存储，实质上是一个监控系统。而kube-state-metrics是将k8s的运行状况在内存中做了个快照，并且获取新的指标，但它没有能力导出这些指标；
3. 换个角度讲，kube-state-metrics本身是metric-server的一种数据来源，虽然现在没有这么做；
4. 另外，像Prometheus这种监控系统，并不会去用metric-server中的数据，它都是自己做指标收集、集成的（Prometheus包含了metric-server的能力），但Prometheus可以监控metric-server本身组件的监控状态并适时报警，这里的监控就可以通过kube-state-metrics来实现，如metric-serverpod的运行状态；

==kube-state-metrics本质上是不断轮询api-server，其性能优化：==

kube-state-metrics在之前的版本中暴露出两个问题：

1. /metrics接口响应慢(10-20s)
2. 内存消耗太大，导致超出limit被杀掉

问题一的方案：就是基于client-go的cache tool实现本地缓存，具体结构为：`var cache = map[uuid][]byte{}`

问题二的的方案是：对于时间序列的字符串，是存在很多重复字符的（如namespace等前缀筛选），可以用指针或者结构化这些重复字符。

==kube-state-metrics优化点和问题：==

1. 因为kube-state-metrics是监听资源的add、delete、update事件，那么在kube-state-metrics部署之前已经运行的资源的数据是不是就拿不到了？其实kube-state-metric利用client-go可以初始化所有已经存在的资源对象，确保没有任何遗漏；
2. kube-state-metrics当前不会输出metadata信息(如help和description）；
3. 缓存实现是基于golang的map，解决并发读问题当期是用了一个简单的互斥锁，应该可以解决问题，后续会考虑golang的sync.Map安全map；
4. kube-state-metrics通过比较resource version来保证event的顺序；
5. kube-state-metrics并不保证包含所有资源；



# 3、使用

```bash
cd /opt/k8s/work/
git clone https://github.com/kubernetes/kube-state-metrics.git
cd kube-state-metrics/examples/standard/

vim service.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: kube-state-metrics
    app.kubernetes.io/version: 1.9.5
  name: kube-state-metrics
  namespace: kube-system
  annotations:
   prometheus.io/scrape: "true" #添加此参数，允许prometheus自动发现
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 8080
    targetPort: http-metrics
  - name: telemetry
    port: 8081
    targetPort: telemetry
  selector:
    app.kubernetes.io/name: kube-state-metrics
    
#创建kube-state-metrics
kubectl delete -f .
```

查看状态：

```bash
$ kubectl get pod,svc -n kube-system|grep kube-state-metrics
pod/kube-state-metrics-6d4847485d-xwlm8   1/1     Running   0          5m20s
service/kube-state-metrics   ClusterIP   None            <none>        8080/TCP,8081/TCP        5m21s
```











