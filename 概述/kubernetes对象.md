

[TOC]



# kubernetes对象





# 1 理解 Kubernetes 对象

在 Kubernetes 系统中，**Kubernetes 对象** 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。 比较特别地是，它们描述了如下信息：

- 哪些容器化应用正在运行（以及在哪些节点上运行）
- 可以被应用使用的资源
- 关于应用运行时表现的策略，比如重启策略、升级策略以及容错策略

Kubernetes 对象是“目标性记录” —— 一旦创建该对象，Kubernetes 系统将不断工作以确保该对象存在。 通过创建对象，你就是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的 **期望状态（Desired State）**。

操作 Kubernetes 对象 —— 无论是创建、修改或者删除 —— 需要使用 [Kubernetes API](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api)。 比如，当使用 `kubectl` 命令行接口（CLI）时，CLI 会调用必要的 Kubernetes API； 也可以在程序中使用[客户端库](https://kubernetes.io/zh-cn/docs/reference/using-api/client-libraries/)， 来直接调用 Kubernetes API。

### 对象规约（Spec）与状态（Status）

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 **`spec`（规约）** 和 对象 **`status`（状态）**。 对于具有 `spec` 的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： **期望状态（Desired State）**。

`status` 描述了对象的**当前状态（Current State）**，它是由 Kubernetes 系统和组件设置并更新的。 在任何时刻，Kubernetes [控制平面](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-control-plane) 都一直都在积极地管理着对象的实际状态，以使之达成期望状态。

例如，Kubernetes 中的 Deployment 对象能够表示运行在集群中的应用。 当创建 Deployment 时，可能会去设置 Deployment 的 `spec`，以指定该应用要有 3 个副本运行。 Kubernetes 系统读取 Deployment 的 `spec`， 并启动我们所期望的应用的 3 个实例 —— 更新状态以与规约相匹配。 如果这些实例中有的失败了（一种状态变更），Kubernetes 系统会通过执行修正操作来响应 `spec` 和状态间的不一致 —— 意味着它会启动一个新的实例来替换。

关于对象 spec、status 和 metadata 的更多信息，可参阅 [Kubernetes API 约定](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md)。



### 描述 Kubernetes 对象

创建 Kubernetes 对象时，必须提供对象的 `spec`，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（直接创建，或经由 `kubectl`）， API 请求必须在请求本体中包含 JSON 格式的信息。 **大多数情况下，你需要提供 `.yaml` 文件为 kubectl 提供这些信息**。 `kubectl` 在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 `.yaml` 示例文件，展示了 Kubernetes Deployment 的必需字段和对象 `spec`：



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # 告知 Deployment 运行 2 个与该模板匹配的 Pod
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

相较于上面使用 `.yaml` 文件来创建 Deployment，另一种类似的方式是使用 `kubectl` 命令行接口（CLI）中的 [`kubectl apply`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) 命令， 将 `.yaml` 文件作为参数。下面是一个示例：

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

输出类似下面这样：

```
deployment.apps/nginx-deployment created
```



### 必需字段

在想要创建的 Kubernetes 对象所对应的 `.yaml` 文件中，需要配置的字段如下：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 帮助唯一标识对象的一些数据，包括一个 `name` 字符串、`UID` 和可选的 `namespace`
- `spec` - 你所期望的该对象的状态

对每个 Kubernetes 对象而言，其 `spec` 之精确格式都是不同的，包含了特定于该对象的嵌套字段。 [Kubernetes API 参考](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/)可以帮助你找到想要使用 Kubernetes 创建的所有对象的规约格式。

例如，参阅 Pod API 参考文档中 [`spec` 字段](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/pod-v1/#PodSpec)。 对于每个 Pod，其 `.spec` 字段设置了 Pod 及其期望状态（例如 Pod 中每个容器的容器镜像名称）。 另一个对象规约的例子是 StatefulSet API 中的 [`spec` 字段](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/#StatefulSetSpec)。 对于 StatefulSet 而言，其 `.spec` 字段设置了 StatefulSet 及其期望状态。 在 StatefulSet 的 `.spec` 内，有一个为 Pod 对象提供的[模板](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/#pod-templates)。 该模板描述了 StatefulSet 控制器为了满足 StatefulSet 规约而要创建的 Pod。 不同类型的对象可以由不同的 `.status` 信息。API 参考页面给出了 `.status` 字段的详细结构， 以及针对不同类型 API 对象的具体内容。





# 2 Kubernetes 对象管理

`kubectl` 命令行工具支持多种不同的方式来创建和管理 Kubernetes 对象。 本文档概述了不同的方法。 阅读 [Kubectl book](https://kubectl.docs.kubernetes.io/zh/) 来了解 kubectl 管理对象的详细信息。



| 管理技术       | 作用于   | 建议的环境 | 支持的写者 | 学习难度 |
| -------------- | -------- | ---------- | ---------- | -------- |
| 指令式命令     | 活跃对象 | 开发项目   | 1+         | 最低     |
| 指令式对象配置 | 单个文件 | 生产项目   | 1          | 中等     |
| 声明式对象配置 | 文件目录 | 生产项目   | 1+         | 最高     |



## 指令式命令

使用指令式命令时，用户可以在集群中的活动对象上进行操作。用户将操作传给 `kubectl` 命令作为参数或标志。

这是开始或者在集群中运行一次性任务的推荐方法。因为这个技术直接在活跃对象 上操作，所以它不提供以前配置的历史记录。

通过创建 Deployment 对象来运行 nginx 容器的实例：

```sh
kubectl create deployment nginx --image nginx
```



## 指令式对象配置

在指令式对象配置中，kubectl 命令指定操作（创建，替换等），可选标志和 至少一个文件名。指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义。

有关对象定义的详细信息，请查看 [API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.26/)。

**警告：**

`replace` 指令式命令将现有规范替换为新提供的规范，并放弃对配置文件中 缺少的对象的所有更改。此方法不应与对象规约被独立于配置文件进行更新的 资源类型一起使用。比如类型为 `LoadBalancer` 的服务，它的 `externalIPs` 字段就是独立于集群配置进行更新。

创建配置文件中定义的对象：

```sh
kubectl create -f nginx.yaml
```

删除两个配置文件中定义的对象：

```sh
kubectl delete -f nginx.yaml -f redis.yaml
```

通过覆盖活动配置来更新配置文件中定义的对象：

```sh
kubectl replace -f nginx.yaml
```



## 声明式对象配置

使用声明式对象配置时，用户对本地存储的对象配置文件进行操作，但是用户 未定义要对该文件执行的操作。 `kubectl` 会自动检测每个文件的创建、更新和删除操作。 这使得配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作。

**说明：**

声明式对象配置保留其他编写者所做的修改，即使这些更改并未合并到对象配置文件中。 可以通过使用 `patch` API 操作仅写入观察到的差异，而不是使用 `replace` API 操作来替换整个对象配置来实现。

处理 `configs` 目录中的所有对象配置文件，创建并更新活跃对象。 可以首先使用 `diff` 子命令查看将要进行的更改，然后在进行应用：

```sh
kubectl diff -f configs/
kubectl apply -f configs/
```

递归处理目录：

```sh
kubectl diff -R -f configs/
kubectl apply -R -f configs/
```



# 3 对象名称和 ID

集群中的每一个对象都有一个[**名称**](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/#names)来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个 [**UID**](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/#uids) 来标识在整个集群中的唯一性。

比如，在同一个[名字空间](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/) 中只能有一个名为 `myapp-1234` 的 Pod，但是可以命名一个 Pod 和一个 Deployment 同为 `myapp-1234`。

对于用户提供的非唯一性的属性，Kubernetes 提供了[标签（Label）](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)和 [注解（Annotation）](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/annotations/)机制。





## 名称

客户端提供的字符串，引用资源 URL 中的对象，如`/api/v1/pods/some name`。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

**名称在同一资源的所有 [API 版本](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)中必须是唯一的。 这些 API 资源通过各自的 API 组、资源类型、名字空间（对于划分名字空间的资源）和名称来区分。 换言之，API 版本在此上下文中是不相关的。**

**说明：**

当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

以下是比较常见的四种资源命名约束。

### **DNS 子域名**

很多资源类型需要可以用作 DNS 子域名的名称。 DNS 子域名的定义可参见 [RFC 1123](https://tools.ietf.org/html/rfc1123)。 这一要求意味着名称必须满足如下规则：

- 不能超过 253 个字符
- 只能包含小写字母、数字，以及 '-' 和 '.'
- 必须以字母数字开头
- 必须以字母数字结尾

### RFC 1123 标签名

某些资源类型需要其名称遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 必须以字母数字开头
- 必须以字母数字结尾

### RFC 1035 标签名

某些资源类型需要其名称遵循 [RFC 1035](https://tools.ietf.org/html/rfc1035) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

- 最多 63 个字符
- 只能包含小写字母、数字，以及 '-'
- 必须以字母开头
- 必须以字母数字结尾

### 路径分段名称

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 `.`、`..`，也不可以包含 `/` 或 `%` 这些字符。

下面是一个名为 `nginx-demo` 的 Pod 的配置清单：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

**说明：**

某些资源类型可能具有额外的命名约束。

## UID

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 UID，它旨在区分类似实体的历史事件。

Kubernetes UID 是全局唯一标识符（也叫 UUID）。 UUID 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667。





# 4 标签和选择算符

**标签（Labels）** 是附加到 Kubernetes 对象（比如 Pod）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

标签能够支持高效的查询和监听操作，对于用户界面和命令行是很理想的。 应使用[注解](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/annotations/)记录非识别信息。

## 动机

标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象，而无需客户端存储这些映射。

服务部署和批处理流水线通常是多维实体（例如，多个分区或部署、多个发行序列、多个层，每层多个微服务）。 管理通常需要交叉操作，这打破了严格的层次表示的封装，特别是由基础设施而不是用户确定的严格的层次结构。

示例标签：

- `"release" : "stable"`, `"release" : "canary"`
- `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
- `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
- `"partition" : "customerA"`, `"partition" : "customerB"`
- `"track" : "daily"`, `"track" : "weekly"`

有一些[常用标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/common-labels/)的例子；你可以任意制定自己的约定。 请记住，标签的 Key 对于给定对象必须是唯一的。





































标签和选择算符
名字空间
注解
字段选择器
Finalizers
属主与附属
推荐使用的标签