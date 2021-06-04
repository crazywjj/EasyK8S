![img](assets/k8s-logo.jpeg)



# 😘Kubernetes

​		Kubernetes，简称k8s，是用8代替8个字符"ubernete"而成的缩写。是一个开源的管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效，Kubernetes提供了应用部署，规划，更新，维护的一种机制。



# 一 入门

主要包含k8s是什么、组件、工作流程、主要功能和优势。

<table border="0">
<tr>
   <td><a href="概念介绍/kubernetes介绍.md">kubernetes介绍</a></td>
</tr>
</table>



# 二 安装部署

主要介绍以kubeadm方式部署单主和多主高可用方式、k8s版本升级。

<table border="0">
<tr>
   <td><a href="安装部署/kubeadm部署--单主集群.md">kubeadm部署--单主集群</a></td>
   <td><a href="安装部署/kubeadm部署--多主集群.md">kubeadm部署--多主集群</a></td>
   <td><a href="安装部署/kubectl命令与资源管理.md">kubectl命令与资源管理</a></td>
   <td><a href="安装部署/kubernetes版本升级.md">kubernetes版本升级</a></td>
</tr>
</table>






# 三 数据存储

主要包含volume存储卷类型、PV资源池、PVC资源请求、共享存储NFS使用、StorageClass类存储、GlusterFS 持久化存储等。

<table border="0">
<tr>
   <td><a href="数据存储/Volume存储卷.md">Volume存储卷</a></td>
   <td><a href="数据存储/PV和PVC.md">PV和PVC</a></td>
   <td><a href="数据存储/StorageClass.md">StorageClass</a></td>
   <td><a href="数据存储/GlusterFS持久化存储.md">GlusterFS 持久化存储</a></td>
    </tr>
</table>




# 四 Pod与控制器

主要包含Pod概念、Pod的创建、重启、终止过程；livenessProbe、readinessProbe存活和就绪探针、Node、Pod软硬亲和性；taints污点、tolerations容忍度

<table border="0">
<tr>
   <td><a href="Pod与控制器/Pod介绍.md">Pod介绍</a></td>
   <td><a href="Pod与控制器/Pod生命周期.md">Pod生命周期</a></td>
   <td><a href="Pod与控制器/Pod健康状态.md">Pod健康状态</a></td>
   <td><a href="Pod与控制器/Pod资源调度.md">Pod资源调度</a></td>
</tr>
<tr>
   <td><a href="Pod与控制器/Pod资源管理与QoS.md">Pod资源管理与QoS</a></td>
   <td><a href="Pod与控制器/Pod控制器.md">Pod控制器</a></td>
</tr>
</table>










# 五 Service与服务发现









# 六 集群管理









# 七 周边生态











<table border="0">
<tr>
   <td><a href="Kubernetes学习/Node.md">Node</a></td>
   <td><a href="Kubernetes学习/Pod.md">Pod</a></td>
   <td><a href="Kubernetes学习/Label.md">Label</a></td>
   <td><a href="Kubernetes学习/RC(Replication Co.mdntroller).md">RC(Replication Controller)</a></td>
</tr>
<tr>
   <td><a href="Kubernetes学习/Deployment.md">Deployment</a></td>
</tr>
<tr>
   <td><a href="Kubernetes学习/02-kubernetes的基本概念.md">02-kubernetes的基本概念</a></td>
</tr>
</table>














# 附录

<table border="0">
    <tr>
        <td><a href="附录/01-Kubernets高可用集群.md">01-Kubernets 高可用集群</a></td>
        <td><a href="附录/02-Kubernetes集群插件.md">02-Kubernetes 集群插件</a></td>
    </tr>
</table>
<table border="0">
    <tr>
    <td><a href="附录/promethues/prometheus+grafana监控部署实践.md">prometheus+grafana 监控部署实践</a></td>
    <td><a href="附录/promethues/prometheus查询语法.md">prometheus 查询语法</a></td>
    <td><a href="附录/promethues/prometheus告警规则.md">prometheus 告警规则</a></td>
    <td><a href="附录/promethues/prometheus浅析.md">prometheus 浅析</a></td>
    </tr>
</table>       
<table border="0">
    <tr>
        <td><a href="k8s使用常见问题.md">k8s使用常见问题</a></td>
    </tr>
</table>    










