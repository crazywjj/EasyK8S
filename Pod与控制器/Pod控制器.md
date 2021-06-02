[TOC]





# Pod控制器

自主式 Pod 对象由调度器绑定至目标工作节点后，即由相应节点上的 kubelet 负责监控其容器的存活性，容器主进程崩溃后， kubelet 够自动重启相应的容器。不过， kubelet 对非主进程崩溃类的容器错误却无从感知，这依赖于用户为 Pod 资游对象自定义的存活性探测( liveness probe ）机制，以便 kubelet 能够探知到此类故障。然而，在 Pod 对象遭到意外删除，或者工作节点自身发生故障时，又该如何处理呢？

kubelet 是Kubemetes 集群节点代理程序，它在每个工作节点上都运行着一个实例。因而，集群中的某工作节点发生故障时，其 kubelet 也必将不再可用，于是，节点上的 Pod 资源的健康状态将无从得到保证，也无法再由 kubelet 重启。此种场景中的 Pod 存活性一般要由工作节点之外的 Pod 控制器来保证。事实上，遭到意外删除的 Pod 资源的恢复也依赖于其控制器。

Pod 控制器由 master的kube-controller-manager 组件提供，常见的此类控制器有 ReplicationController、ReplicaSet Deployment 、DaemonSet、StatefulSet、Job、CronJob 等，它们分别以不同的方式管理 Pod 资源对象。实践中，对 Pod 对象的管理通常都是由某种控制器的特定对象来实现的，包括其创建、删除及重新调度等操作。	



在Kubernetes平台上，我们很少会直接创建一个Pod，在大多数情况下会通过RC、Deployment、DaemonSet、Job等控制器完成对一 组Pod副本的创建、调度及全生命周期的自动控制任务。

**控制器** 又称之为**工作负载**，常见包含以下类型控制器：

- **ReplicationController（RC） 和 ReplicaSet（RS）**
  - ReplicaSet（简称RS）: 是Replication Controller 升级版本。当用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。
- **Deployment**
  - 工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器。支持滚动更新和回滚功能，还提供声明式配置。
- **DaemonSet**
  - 用于确保集群中的每一个节点只运行特定的pod副本，通常用于实现系统级后台任务,比如ingress,elk.服务是无状态的,服务必须是守护进程。
- **StatefulSet**
  - 管理有状态应用,比如redis,mysql。
- **Job/CronJob**
  - Job是一次性任务运行，完成就立即退出，不需要重启或重建。
  - CronJob是周期性任务控制，执行后就退出，不需要持续后台运行。
- **HorizontalPodAutoscaler（HPA）**自动水平伸缩



# 1 关于Pod控制器

Kubernetes 提供了众多的控制器来管理各种类型的资源，如 Node Lifecycle Controller Namespace Controller Service Controller 和 Deployment Controller 等，它们的功用几乎可以做到见名知义 创建完成后， 每一个控制器对象都可以通过内部的和解循环（ reconci iation loop ），不间断地监控着由其负责的所有资源并确保其处于或不断地逼近用户定义的目标状态。

## 1.1  Pod控制器概述

Master 的各组件中， API Server 仅负责将资源存储于etcd 中，并将其变动通知给各相关的客户端程序，如 kubelet、kube-scheduler kube-proxy、kube-controller-manager 等，kub-scheduler 监控到处于未绑定状态的 Pod 对象出现时，会立即启动调度器为其挑选适配的工作节点。

![image-20210602140135813](assets/image-20210602140135813.png)

## 1.2 控制器与 Pod 对象

通常，一个 Pod 控制器资源至少应该包含三个基本的组成部分：

- 标签选择器：匹配并关联 Pod 资源对象，并据此判断哪个pod归自己管理。
- 期望的副本数：当现存的pod数量不足，会根据pod资源模板进行新建帮助用户管理无状态的pod资源，精确反应用户定义的目标数量，但是RelicaSet不是直接使用的控制器，而是使用Deployment。
- Pod 模板 ：用于新建 Pod 资源对象的 Pod 模板资源。

>  *注意*：DaemonSet 用于确保集群中的每个工作节点或符合条件的每个节点上都运行着一个Pod 副本，而不是某个精确的数量值。因此不具有上面组成部分中的第二项。





## 1.3 Pod 模板资源

Pod Template 是Kubernetes API 的常用资源类型，常用于为控制器指定自动创建 Pod资源对象时所需要的配置信息。因为要内嵌于控制器中使用，所以 Pod 模板的配置信息中不需要apiVersion和kind 字段 ，但此之外的其他内容与定义自主式 Pod对象所支持的字段几乎完全相同，这包括 metadata和spec 及其 内嵌的其他各个字段。Pod 控制器类资源的spec字段通常 内嵌 replicas、selector和template 字段，其中template 即为 Pod 模板的定义。







# 2 RC和RS控制器

Replication Controller（简称RC）：是Kubernetes系统中的核心概念之一，即声明某种Pod的副本数量在任意时刻都符合某个预期值。

ReplicaSet（简称RS）: 是Replication Controller 升级版本。当用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。

应用升级时，通常会使用一个新的容器镜像版本替代旧版本。我们希望系统平滑升级，比如在当前系统中有10个对应的旧版本的Pod，则最佳的系统升级方式是旧版本的Pod每停止一个，就同时创建一个新版本的Pod，在整个升级过程中此消彼长，而运行中的Pod数量始终是10个，几分钟以后，当所有的Pod都已经是新版本时，系统升级完成。通过RC机制，Kubernetes很容易就实现了这种高级实用的特性，被称为“滚动升级”（Rolling Update）。

**Replica Set和Replication Controller的区别：**

Replica Set与RC当前的唯一区别是，**Replica Sets支持基于集合的Label selector（Set-based selector）如：version in (v1.0, v2.0) 或 env notin (dev, qa)**，**而 RC只支持基于等式的Label Selector（equality-based selector）如：env=dev或environment!=qa**，这使得Replica Set的功能更强。





## 2.1 RC用法

**1、RC定义了如下**

> 1.Pod期待的副本数(replicas)
>
> 2.用于筛选目标Pod的Label Seletcor(标签选择器)
>
> 3.当Pod的副本小于预期(replicas)时，用于创建新Pod的Pod模板(template)

**2、RC主要功能**

- 确保Pod数量: 它会确保Kubernetes中有指定数量的Pod在运行，如果少于指定数量的Pod，RC就会创建新的，反之会删除多余的，保证Pod的副本数量不变
- 确保Pod健康: 当Pod不健康，RC会杀死不健康的Pod，重新创建新的
- 弹性伸缩: 在业务高峰或者低峰的时候，可以用RC来动态调整Pod数量来提供资源的利用率吧，当然也可以使用HPA来实现
- 滚动升级: 滚动升级是一种平滑的升级方式，通过逐步替换的策略，保证整体系统的稳定性

**3、RC使用**

一个完整的RC定义的例子  [rc-nginx.yaml](yaml\rc-nginx.yaml)  ，即确保拥有app=rc-nginx标签的这个Pod（运行nginx容器）在整个Kubernetes集群中始终只有一个副本：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-nginx
spec:
  replicas: 1
  selector:
    app: rc-nginx
  template:
    metadata:
      labels:
        app: rc-nginx
    spec:
      containers:
      - name: nginx-demo
        image: nginx:1.14.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

```

执行并测试删除：

```bash
$ kubectl create -f  rc-nginx.yaml
$ kubectl get pod frontend-rc-kfffm
NAME                READY   STATUS    RESTARTS   AGE
rc-nginx-wzl9m            1/1     Running   0          37s
# 尝试删除rc-nginx-wzl9m，会发现系统会立刻创建一个新的rc-nginx-xxx
$ kubectl delete pod rc-nginx-wzl9m
pod "rc-nginx-wzl9m" deleted

$ kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
rc-nginx-bwk4n            1/1     Running   0          59s


```

RC将其提交到Kubernetes集群中后，Master 上的Controller Manager组件就得到通知，定期巡检系统中当前存活的目标Pod，并确保目标Pod实例的数量刚好等于此RC的期望值，有过多的Pod副本在运行，系统就会停掉一些Pod，少于1个就会自动创建，大大减少了系统管理员在传统IT环境中需要完成的许多手工运维工作（如主机监控脚本、应用监控脚本、故障恢复脚本等）。

**4、RC的副本数量动态缩放**

在Pod运行时，我们可以通过`kubectl scale`命令对RC的副本数量进行**动态缩放**（scaling）。

```bash
# 扩展Pod副本到3
kubectl scale rc [rc名称] --replicas=3 

# 缩减副本到1
kubectl scale rc [rc名称] --replicas=1

# 查看结果RC数量
$ kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
rc-nginx   1         1         1       5m9s

DESIRED   #rc设置的数量
CURRENT   #已经创建的数量
READY     #准备好的数量
```

将原来 rc-nginx.yaml 的 1个副本改为3个：

```bash
$ kubectl scale rc rc-nginx --replicas=3
$ kubectl get pod|grep rc-nginx
rc-nginx-6cjft            1/1     Running   0          46s
rc-nginx-bwk4n            1/1     Running   0          8m51s
rc-nginx-mwx8x            1/1     Running   0          46s

```

除了通过命令的方式修改，我们还可以通过yaml文件的方式进行缩放。

**注意：** 删除RC并不会影响通过该RC已创建号的Pod。为了删除所有Pod，可以设置replicas的值为0，然后更新该RC。另外,kubectl提供了`stop`和`delete`命令来一次性删除RC和RC控制的全部Pod



**5、基于RC滚动升级**

指定镜像升级，每10秒升级一个

```
# 基于命令
kubectl rolling-update rc-nginx --image=nginx:1.20.0 --update-period=10s
# 基于yaml文件升级
kubectl rolling-update rc-nginx -f rc-nginx-1.20.0.yaml --update-period=10s
```

使用kubectl rolling-update实现滚动更新的不足：

- rolling-update的逻辑是由kubectl发出N条命令到APIServer完成的，很可能因为网络原因导致update中断
- 需要创建一个新的rc，名字与要更新的rc不能一样
- 回滚还需要执行rolling-update，只是用老的版本替换新的版本
- service执行的rolling-update在集群中没有记录，后续无法跟踪rolling-update历史

现如今，RC的方式已经被Deployment替代。



## 2.2 RS用法

ReplicaSet（简称RS）: 是Replication Controller 升级版本。当用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。

**ReplicaSet 能够实现以下功能：**

- 确保 Pod 资源对象的数量： ReplicaSet 需要确保由其控制运行的 Pod副本数量精确吻合配置中定义的期望值，否则就会自动补足所缺或终止所余。

- 确保 Pod 健康运行：探测到由其管控的 Pod 对象因其所在的工作节点故障而不可用时，自动请求由调度器于其他工作节点创建缺失的 Pod 副本。

- 弹性伸缩：业务规模因各种原因时常存在明显波动，在波峰或波谷期间，可以通过ReplicaSet 控制器动态调整相关 Pod 资源对象的数量。 此外，在必要时还可以通过HPA (Hroizonta!PodAutoscaler ）控制器实现 Pod 资源的自动伸缩。

**1、创建 [rs-nginx.yaml](yaml\rs-nginx.yaml) 资源：**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rs-nginx
  template:
    metadata:
      labels:
        app: rs-nginx
    spec:
      containers:
      - name: nginx-demo
        image: nginx:1.14.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

```

从以上示例可以看出，它也由 kind、apiVersion、metadata、spec和status这5个一级字段组成，其中 status 为只读字段，因此需要在清单文件中配置的仅为前4个字段。它的 spec 字段一般嵌套使用以下几个属性字段：

- `replicas <integer>` ：期望的 Pod 对象副本数；

- `selector <Object>`：当前控制器匹配 Pod 对象副本的标签选择器，支持 `matchLabels`和`matchExpressions `两种匹配机制；

- `template <Object>`：用于补足 Pod 副本数量时使用的 Pod 模板资源；

- `minReadySeconds <integer>` ：新建的 Pod 对象，在启动后多长时间内如果其容器未发生崩溃等异常情况即被视为“就绪”；默认为 

  0秒，表示 一旦就绪性探测成功，即视为可用。

创建资源后查看详细：

```
$ kubectl get rs rs-nginx -o wide
NAME       DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES         SELECTOR
rs-nginx   1         1         1       7m49s   nginx-demo   nginx:1.14.0   app=rs-nginx

```

**注意：**强行修改RS控制器下管控的Pod资源的标签，会导致它不再被控制器作为副本计数，这样就会触发RS对副本对象进行补足机制。

测试如下：将` rs-nginx-sbf2b` 的标签 app 的值置空

```bash
#查看标签
$ kubectl get pods rs-nginx-sbf2b --show-labels
NAME             READY   STATUS    RESTARTS   AGE   LABELS
rs-nginx-sbf2b   1/1     Running   0          16m   app=rs-nginx

$ kubectl label pods rs-nginx-sbf2b --overwrite app=
pod/rs-nginx-sbf2b labeled

$ kubectl get po|grep rs-nginx
rs-nginx-5rfw8            1/1     Running   0          27s
rs-nginx-sbf2b            1/1     Running   0          19m

```

由此可见，修改 Pod 资源的标签即可将其从控制器的管控之下移出，当然，修改后标签的如果又能被其他控制器资源的标签选择器所命中，则此时它又成了隶属于另一控制器的副本。如果修改其标签后的 Pod 对象不再隶属于任何控制器，那么它就将成为自主式 Pod，与此前手动创建的 Pod 对象的特性相同，即误删 或所在的工作节点故障都会造成其永久性的消失。

**2、查看Pod资源变动的相关事件**

```bash
kubectl describe rs rs-nginx
Name:         rs-nginx
Namespace:    default
Selector:     app=rs-nginx
Labels:       <none>
Annotations:  <none>
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=rs-nginx
  Containers:
   nginx-demo:
    Image:        nginx:1.14.0
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age    From                   Message
  ----    ------            ----   ----                   -------
  Normal  SuccessfulCreate  24m    replicaset-controller  Created pod: rs-nginx-sbf2b
  Normal  SuccessfulCreate  4m59s  replicaset-controller  Created pod: rs-nginx-5rfw8

```

**3、更新 ReplicaSet 控制器**

Replic aSet控制器的核心组成部分是标签选择器、副本数量及 Pod 模板，但更新操作一般是围绕 replicas和template 两个字段值进行的，毕竟改变标签选择器的需求几乎不存在。

更新Pod模板：升级应用

vim  [rs-nginx-v2.yaml](yaml\rs-nginx-v2.yaml) 

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rs-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rs-nginx
  template:
    metadata:
      labels:
        app: rs-nginx
    spec:
      containers:
      - name: nginx-demo
        image: nginx:1.20.0  #修改镜像
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80

```

对新版本的清单文件执行`kubectl apply`  或  `kubectl replace `  命令即可完成 rs-nginx控制器资源的修改操作：

注意：使用`kubectl replace`进行更新后，需要手动删除原来的pod才能更新为新版

```bash

$ kubectl get pod rs-nginx-bgdlv
NAME             READY   STATUS    RESTARTS   AGE
rs-nginx-bgdlv   1/1     Running   0          17s
$ kubectl exec -it rs-nginx-bgdlv -- nginx -v
nginx version: nginx/1.14.0
$ kubectl apply -f rs-nginx-v2.yaml
Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply
replicaset.apps/rs-nginx configured

#会发现nginx的版本，没有发生变化，手动删除控制器现有的 Pod 对象（或修改与其匹配的控制器标签选择器的标签），并由控制器基于新的Pod 模板自动创建出足额的 Pod，即可完成一次应用的升级。
$ kubectl exec -it rs-nginx-bgdlv -- nginx -v
nginx version: nginx/1.14.0

# 这里采用一次性删除所有含app=rs-nginx标签的pod，生产环境中，建议着个删除，以确保最小的影响业务
$ kubectl delete pod rs-nginx-bgdlv -l app=rs-nginx
pod "rs-nginx-bgdlv" deleted

$ kubectl exec -it rs-nginx-b7q6t -- nginx -v
nginx version: nginx/1.20.0

```

**4、扩容和缩容**

```bash
# 扩容
$ kubectl scale replicasets rs-nginx --replicas=5

# 缩容
$ kubectl scale replicasets rs-nginx --replicas=1
```

另外， kubectl scale 还支持在现有 Pod副本数量，符合指定的值时才执行扩展操作，这仅需要为命令使用`--current-replica `选项即可:

```bash
$ kubectl scale replicasets rs-nginx --current-replicas=2 --replicas=5
```

尽管 ReplicaSet 控制器功能强大，但在实践中 ，它却并非是用户直接使用的控制器，而是要由比其更高一级抽象的 Deployment 控制器对象来调用。



# 3 Deployment控制器

Deployment是Kubernetes在1.2版本中引入的新概念，用于更好地解决Pod的编排问题。是一个更高层次的API对象，它管理ReplicaSets和Pod，并提供声明式更新等功能。官方建议使用Deployment管理ReplicaSets，而不是直接使用ReplicaSets。

Deployment 控制器比ReplicaSets多了很多特性：

- **事件和状态查看** ：必要时可以查 Deployment 对象升级的详细进度和状态。
- **回滚**：升级操作完成后发现问题时，支持使用回滚机制将应用返回到前一个或由用户指定的历史记记录中的版本。
- **版本记录** ：对 Deployment 对象的每次操作都予以保存，以供后回滚操作使用。
- **暂停和启动** ：对于每一次升级 ，都能够随时暂停和启动。
- **多种自动更新方案**：一是 Recreate 即重建更新机制，全面停止，删除旧pod后启用新版本代理；另一个是 RollingUpdate ，即滚动升级，着步替换旧的pod至新版本。



## 3.1 创建 Deployment

Deployment 是标准的 Kubernetes API 资源，它建构于 ReplicaSet 资源之上，于是其spec 段中嵌套使用的字段包含了ReplicaSet 控制器支持的 replicas、selector、template、minReadySeconds ，它也利用这些信息完成了其二级资源 ReplicaSet 对象的创建。

vim 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx：1.14.0
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m
              memory: 64Mi
            limits:
              cpu: 400m
              memory: 256Mi

```

执行后查看：

```
$ kubectl get pod |grep nginx-deployment
nginx-deployment-845c84b94c-g2g95   1/1     Running   0          7m16s
nginx-deployment-845c84b94c-gkj4q   1/1     Running   0          7m16s
nginx-deployment-845c84b94c-ztjm7   1/1     Running   0          7m16s

$ kubectl get pod -l name=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-845c84b94c-g2g95   1/1     Running   0          8m10s
nginx-deployment-845c84b94c-gkj4q   1/1     Running   0          8m10s
nginx-deployment-845c84b94c-ztjm7   1/1     Running   0          8m10s

$ kubectl get deploy nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           6m36s


```

对上面deploy输出中涉及的数量解释如下： 

- `READY` ：1/1左边1是真正运行的副本数，右边1是期望的副本数即replicas定义的副本数。
- `UP-TO-DATE`：显示已更新以实现期望状态的副本数。
- `AVAILABLE`：显示应用程序可供用户使用的副本数。
- `AGE` ：显示应用程序运行的时间量。



## 3.2 Deployment更新策略

ReplicaSet 控制器的应用更新，要手动分成多步并以特定的次序进行，过程复杂且容易出错，而 Deployment 却只需要由用户指定在 Pod 模板中要改动的内容，例如：容器镜像文件的版本，剩下的步骤由其自动完成。

Deployment 支持两种更新策略：滚动更新（ rolling update ）和重新创建（ recreate)，默认为滚动更新。

滚动更新时，应用升级期间还要确保可用的 Pod 对象数量不低于某阈值，以确保可以持续处理客户端的服务请求，变动的方式和 Pod 对象的数量范围将通过 `spec.strategy.rollingUpdate.maxSurge` 和 `spec.strategy.rollingUpdate.maxUnavailable` 两个属性协同进行定义，它的功能如下：

- maxSurge：指定升级期间存在的总 Pod 对象数量最多可超出期望值的个数，其值可以是0或正整数，也可以是一个期望值的百分比；例如，如果期望值为3 ，当前的属性值为1，则表示 Pod 对象的总数不能超过4。
- maxUnavailable ：升级期间正常可用的 Pod 副本数（包括新旧版本）最多不能低于期望数值的个数 ，其值可以是0或正整数，也可以是 个期望值的百分比；默认值为1，该值意味着如果期望值是3 ，则升级期间至少要有两个 Pod 对象处于正常提供服务的状态。

> **注意：**max Surge max Unavailab 性的值不可同时为0 ，否则 Pod对象的副本数量在符合用户期望的数量后无法做出合理变动以进行滚动更新操作。

配置时，用户还可以使用 Deplpoyment 控制器的 `spec.minReadySeconds` 属性来控制应用升级的速度。Deployment 控制器也支持用户保留其滚动更新历史中的旧 ReplicaSet 对象版本，使用`Spec.revisionHistoryLimit`，进行定义保存历史版本数量。

> **注意**：为了保存版本升级的历史，需要在创建 Deployment 对象时于命令中使用`--record`选项。



尽管滚动更新以节约系统资源著称，但它也存在。直接改动现有环系统引人不确定性风险，而且升级过程出现问题后，执行回滚操作也 较为缓慢。有鉴于此， 金丝雀部署可能是较为理想的方式，当然，如果不考虑虑系统资源的可用性，那么传统的蓝绿部署也是不错的选择。

