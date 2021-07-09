[TOC]







# Service

Service是Kubernetes的核心概念，通过创建Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载 分发到后端的各个容器应用上。



# 1 Service 资源及实现模型

Kubernetes Service 定义了这样一种抽象：通过规则定义出由多个 Pod 对象组合而成的逻辑集合，以及访问这组 Pod的策略。Service 资源基于标签选择器将一组Pod 定义成一个逻辑组合，并通过自己的 IP 地址和端口调度代理请求至组内 Pod对象上。

![image-20210618150633852](assets/image-20210618150633852.png)

由 Deployment 等控制器管理的 Pod 对象中断后会由新建的资源对象所取代，而扩缩容后的应用则会带来 Pod 对象的变动，随之变化的还有 Pod 的 IP 地址访问接口等，这也是编排系统之上的应用程序必然要面临的问题。为此， Kubemetes 特地设计了 Service 资源来解决此类问题。

![image-20210618145339750](assets/image-20210618145339750.png)





# 2 Kubernetes的三种IP

- Node IP
  - Node IP是Kubernetes集群中节点的物理网卡IP地址(一般为内网)，属于这个网络的服务器之间都可以直接通信，所以Kubernetes集群外要想访问Kubernetes集群内部的某个节点或者服务，肯定得通过Node IP进行通信（这个时候一般是通过外网IP了）
- Cluster IP
  - Cluster IP是Service的IP地址，是一个虚拟的IP，仅仅作用于Kubernetes Service这个对象，由Kubernetes自己来进行管理和分配地址，当然我们也无法ping这个地址，他没有一个真正的实体对象来响应，他只能结合Service Port来组成一个可以通信的服务,相当于VIP地址，来代理后端服务。
- Pod IP
  - Pod IP是每个Pod的IP地址，它是Docker Engine根据docker0网桥的IP地址段进行分配的（我们这里使用的是flannel这种网络插件保证所有节点的Pod IP不会冲突）



# 3 虚拟IP和Service代理

Service 对象的 IP 地址也称为 Cluster IP ，它位于为 Kubernetes 集群配置指定专用 IP 地址的范围之内，而且是一种虚拟 IP 地址，它在 Service 对象创建后即保持不变，并且能够被同一集群中的 Pod 资源所访问。Service 端口用于接收客户端请求并将其转发至其后端的Pod中应用的相应端口之上 ，因此，这种代理机制也称为"端口代理"（ port proxy ）或四层代理，它工作于 TCP/IP 协议的传输层。

一个 Service 对象就是工作节点上的一些 iptables或ipvs 规则，用于将到达 Service对象IP 地址的流量调度转发至相应的 Endpoints 对象指向的 IP 地址和端口上。每个工作节点的 kube-proxy组件通过 API Server 监控 Service 及与其关联的 Pod对象，并将其 创建或变动实时反映至当前工作节点上相应的 iptables 或 ipvs 规则上。			

> **==注意==：**Servic 并不直接链接至 Pod对象，还有个中间层 `Endpoints `资源对象，它是一个由IP地址和端口组成的列表。



kube-proxy 将请求代理至相应端点的方式有三种： `userspace `（用户空间 ）、 `iptables ` 和 `ipvs`。

## 3.1 userspace 代理模式

此处的 userspace 是指 Linux 操作系统的用户空间。这种模式，kube-proxy 会监视API Server上Service 对象和 Endpoints 对象的添加和移除操作。 对每个 Service，它会在本地 Node 上打开一个端口（随机选择）。 任何连接到“代理端口”的请求，都会被代理到 Service 的后端的某个`Pod` 上。 使用哪个后端 Pod，是 kube-proxy 基于 `SessionAffinity` 来确定的。

最后，它配置 iptables 规则，捕获到达该 Service 的 `clusterIP`（是虚拟 IP） 和 `Port` 的请求，并重定向到代理端口，代理端口再代理请求到后端Pod。

![image-20210618160129059](assets/image-20210618160129059.png)

默认情况下，用户空间模式下的 kube-proxy 通过轮询算法选择后端。这种代理模型中，请求流量到达内核空间后经由套接字送往用户空间的 kube-proxy,而后再由它送回内核空 ，并调度至后端 Pod 这种方式中，请求在内核空间和用户空间来回转发必然会导致效率不高。







## 3.2 iptables 代理模型









