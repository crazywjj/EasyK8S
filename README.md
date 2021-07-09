![img](assets/k8s-logo.jpeg)



# 😘Kubernetes

​		Kubernetes，简称k8s，是用8代替8个字符"ubernete"而成的缩写。是一个开源的管理云平台中多个主机上的容器化的应用，Kubernetes的目标是让部署容器化的应用简单并且高效，Kubernetes提供了应用部署，规划，更新，维护的一种机制。



# 一 入门

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
    <tr>
     <td>1</td>
     <td><a href="概念介绍/kubernetes介绍.md">kubernetes介绍</a></td>
     <td>包含kubernetes是什么、组件(API server、Controller-manager、scheduler、kubelet、kube-proxy等)、工作流程、主要功能和优势</td>
     </tr>
   </tbody>
</table>


# 二 安装部署

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
    <tr> 
     <td>1</td> 
     <td><a href="安装部署/kubeadm部署--单主集群.md">kubeadm部署--单主集群</a></td> 
     <td>包含kubeadm介绍、部署、Flannel网络插件安装、node节点加入与移除、dashboard面板部署</td> 
     </tr>
    <tr> 
     <td>2</td> 
     <td><a href="安装部署/kubeadm部署--多主集群.md">kubeadm部署--多主集群</a></td> 
     <td>包含k8s高可用架构说明、etcd高可用集群、nginx+keepalived四层高可用代理、kubeadm部署高可用集群</td> 
    </tr>
    <tr> 
     <td>3</td> 
     <td><a href="安装部署/kubectl命令与资源管理.md">kubectl命令与资源管理</a></td> 
     <td>包含kubectl命令工具介绍、docker中mysql备份还原、k8s中mysql备份还原</td> 
    </tr>
    <tr> 
     <td>4</td> 
     <td><a href="安装部署/kubernetes版本升级.md">kubernetes版本升级</a></td> 
     <td>包含二进制升级方式、kubeadm升级集群</td> 
    </tr> 
   </tbody>
</table>


# 三 数据存储

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
    <tr> 
     <td>1</td> 
     <td><a href="数据存储/Volume存储卷.md">Volume存储卷</a></td> 
     <td>包含Volume存储卷、emptyDir临时存储卷、hostPath节点存储卷、NFS网络存储卷等介绍</td> 
    </tr>
     <tr> 
     <td>2</td> 
     <td><a href="数据存储/PV和PVC.md">PV和PVC</a></td> 
     <td>包含PV和PVC概念、生命周期、NFS网络存储卷使用</td> 
    </tr>
    <tr> 
     <td>3</td> 
     <td><a href="数据存储/StorageClass.md">StorageClass</a></td> 
     <td>包含StorageClass介绍、运行流程、关键参数、动态卷使用（NFS相关）、回收策略</td> 
    </tr>
    <tr> 
     <td>4</td> 
     <td><a href="数据存储/GlusterFS持久化存储.md">GlusterFS 持久化存储</a></td> 
     <td>包含GlusterFS介绍、集群搭建、手动挂载使用GlusterFS过程、GlusterFS+Heketi+StorageClass动态挂载使用</td> 
    </tr> 
   </tbody>
</table>


# 四 Pod与控制器

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
 <tr> 
     <td>1</td> 
     <td><a href="Pod与控制器/Pod介绍.md">Pod介绍</a></td> 
     <td>包含Pod概念、组成、定义和基本用法、静态Pod、Pod故障归类与排除方法</td> 
    </tr>
     <tr> 
     <td>2</td> 
     <td><a href="Pod与控制器/Pod生命周期.md">Pod生命周期</a></td> 
     <td>包含Pod状态、创建过程、重启策略、持久性与终止过程</td> 
    </tr>
    <tr> 
     <td>3</td> 
     <td><a href="Pod与控制器/Pod健康状态.md">Pod健康状态</a></td> 
     <td>包含livenessProbe存活探测、readinessProbe就绪探测</td> 
    </tr>
    <tr> 
     <td>4</td> 
     <td><a href="Pod与控制器/Pod资源调度.md">Pod资源调度</a></td> 
     <td>包含常用预选策略、node硬亲和性与软亲和性、Pod硬亲和性、Pod软亲和性、Pod反亲和性、taints污点、tolerations容忍度、Pod优先级和抢占式调度</td> 
    </tr>
	<tr> 
     <td>5</td> 
     <td><a href="Pod与控制器/Pod资源管理与QoS.md">Pod资源管理与QoS</a></td> 
     <td>包含Requests资源需求和Limits资源限制、QoS分类</td> 
    </tr>
    <tr> 
     <td>6</td> 
     <td><a href="Pod与控制器/Pod控制器.md">Pod控制器</a></td> 
     <td>包含Pod控制器概述、RC和RC、Deployment、DaemonSet、StatefulSet、Job、CronJob、HPA横向自动扩容等控制器</td> 
    </tr> 
   </tbody>
</table>



# 五 ConfigMap和Secret

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
 <tr> 
     <td>1</td> 
     <td><a href="ConfigMap和Secret/ConfigMap.md">ConfigMap</a></td> 
     <td>包含ConfigMap介绍、创建方式、使用方式、更新</td> 
    </tr>
  <tr> 
     <td>2</td> 
     <td><a href="ConfigMap和Secret/Secret.md">Secret</a></td> 
     <td>包含Secret概述、创建方式、使用方法</td> 
    </tr> 
   </tbody>
</table>  



# 六 Service和服务发现

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
 <tr> 
     <td>1</td> 
     <td><a href="Service和服务发现/Service.md">ConfigMap</a></td> 
     <td>包含</td> 
    </tr>
  <tr> 
     <td>2</td> 
     <td><a href="ConfigMap和Secret/Secret.md">Secret</a></td> 
     <td>包含</td> 
    </tr> 
   </tbody>
</table>  


# 七 集群管理

















# 八 生态周边

<table cellpadding="2" border="1">
   <tbody>
    <tr> 
     <th bgcolor="green"><font face="微软雅黑" color="white">序号</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">标题</font></th>
     <th bgcolor="green"><font face="微软雅黑" color="white">主要内容</font></th>
    </tr>
 <tr> 
     <td>1</td> 
     <td><a href="生态周边/EFK日志收集.md">EFK日志收集</a></td>
     <td>包含</td> 
    </tr>
  <tr> 
     <td>2</td> 
     <td><a href="生态周边/kube-prometheus监控.md">kube-prometheus监控</a></td>
     <td>包含</td> 
    </tr>     
     <tr> 
     <td>3</td> 
     <td><a href="生态周边/Reloader.md">Reloader</a></td>
     <td>包含</td> 
    </tr>
   </tbody>
</table>










<table border="0">
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










