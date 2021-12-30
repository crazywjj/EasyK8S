[TOC]







# Helm应用包管理工具

**1 简介**

Helm是一个kubernetes应用的包管理工具，用来管理预先配置好的安装包资源。
Helm chart是用来封装kubernetes原生应用程序的yaml文件，可以在你部署应用的时候自定义应用程序的一些metadata，便与应用程序的分发。

**2 名词解释**

**Helm：**

是一个命令行下的客户端工具。主要用于 Kubernetes 应用程序 Chart 的创建、打包、发布以及创建和管理本地和远程的 Chart 仓库。

**Tiller：**

是 Helm 的服务端，部署在 Kubernetes 集群中。Tiller 用于接收 Helm 的请求，并根据 Chart 生成 Kubernetes 的部署文件（ Helm 称为 Release ），然后提交给 Kubernetes 创建应用。Tiller 还提供了 Release 的升级、删除、回滚等一系列功能。

**Chart：**

Helm 的软件包，采用 TAR 格式。类似于 APT 的 DEB 包或者 YUM 的 RPM 包，其包含了一组定义 Kubernetes 资源相关的 YAML 文件。

**Repoistory：**

Helm 的软件仓库，Repository 本质上是一个 Web 服务器，该服务器保存了一系列的 Chart 软件包以供用户下载，并且提供了一个该 Repository 的 Chart 包的清单文件以供查询。Helm 可以同时管理多个不同的 Repository。

**Release：**
使用 helm install 命令在 Kubernetes 集群中部署的 Chart 称为 Release。

> 注：需要注意的是：Helm 中提到的 Release 和我们通常概念中的版本有所不同，这里的 Release 可以理解为 Helm 使用 Chart 包部署的一个应用实例. 



**Chart Install 过程：**

```
Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
Helm 将指定的 Chart 结构和 Values 信息通过 gRPC 传递给 Tiller。
Tiller 根据 Chart 和 Values 生成一个 Release。
Tiller 将 Release 发送给 Kubernetes 用于生成 Release。
```

**Chart Update 过程：**

```
Helm 从指定的目录或者 TAR 文件中解析出 Chart 结构信息。
Helm 将需要更新的 Release 的名称、Chart 结构和 Values 信息传递给 Tiller。
Tiller 生成 Release 并更新指定名称的 Release 的 History。
Tiller 将 Release 发送给 Kubernetes 用于更新 Release。
```

**Chart Rollback 过程：**

```
Helm 将要回滚的 Release 的名称传递给 Tiller。
Tiller 根据 Release 的名称查找 History。
Tiller 从 History 中获取上一个 Release。
Tiller 将上一个 Release 发送给 Kubernetes 用于替换当前 Release。
```

**Chart 处理依赖说明：**

Tiller 在处理 Chart 时，直接将 Chart 以及其依赖的所有 Charts 合并为一个 Release，同时传递给 Kubernetes。因此 Tiller 并不负责管理依赖之间的启动顺序。Chart 中的应用需要能够自行处理依赖关系。

**Helm和charts的主要作用：**

```
应用程序封装
版本管理
依赖检查
便于应用程序分发
```



