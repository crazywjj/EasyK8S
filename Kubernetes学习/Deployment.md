



# Deployment

出现原因：ReplicationController和ReplicaSet这两种资源对象需要其他控制器进行配合才可以实现滚动升级，并且难度大，因此k8s提供了一种基于ReplicaSet的资源对象Deployment可以支持`声明式`地更新应用。

# 1 介绍

Deployment是Kubernetes在1.2版本中引入的新概念，用于更好地解决Pod的编排问题。是一个更高层次的API对象，它管理ReplicaSets和Pod，并提供声明式更新等功能。官方建议使用Deployment管理ReplicaSets，而不是直接使用ReplicaSets。



# 2 应用场景

Deployment的典型使用场景有以下几个。 

- 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建。 （定义一组pod）
- 检查Deployment的状态来看部署动作是否完成（维持Pod副本数量与预期的一致）。 
- 更新Deployment以创建新的Pod（比如镜像升级）。 
- 如果当前Deployment不稳定，则回滚到一个早先的Deployment版本。 （支持版本回滚）
- 暂停Deployment以便于一次性修改多个PodTemplateSpec的配置项，之后再恢复Deployment，进行新的发布。 
- 扩展Deployment以应对高负载。 
- 查看Deployment的状态，以此作为发布是否成功的指标。 
- 清理不再需要的旧版本ReplicaSets。 

除了API声明与Kind类型等有所区别，Deployment的定义与Replica Set的定义很类似：

![image-20210421162732321](assets/image-20210421162732321.png)



# 3 使用

下面是一个部署的例子。创建了一个ReplicaSet，以建立三个nginx Pods。

vim nginx-deployment.yaml

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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



创建一个名为  [frontend-deployment.yaml](assets\frontend-deployment.yaml) 的Deployment描述文件，内容如下： 

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: app-demo
        tier: frontend
    spec:
      containers:
      - name: tomcat-demo
        image: tomcat
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080

```

创建Deployment：

```bash
# kubectl create -f frontend-deployment.yaml
deployment.apps/frontend created
```

查看Deployment的信息

```bash
# kubectl get deployments
NAME       READY   UP-TO-DATE   AVAILABLE   AGE
frontend   1/1     1            1           23m
```

对上述输出中涉及的数量解释如下。 

- `READY` ：1/1左边1是真正运行的副本数，右边1是期望的副本数即replicas定义的副本数。
- `UP-TO-DATE`：显示已更新以实现期望状态的副本数。
- `AVAILABLE`：显示应用程序可供用户使用的副本数。
- `AGE` ：显示应用程序运行的时间量。

要查看Deployment创建的ReplicaSet（rs），运行`kubectl get rs`输出，看到它的命名与Deployment的名称有关系： 

ReplicaSet 命名规则是：[DEPLOYMENT-NAME]-[RANDOM-STRING]

```bash
# kubectl get rs
NAME                  DESIRED   CURRENT   READY   AGE
frontend-797f47d685   1         1         1       44m
```

查看对应的pod：

发现Pod的命名以Deployment对应的Replica Set的名称为前缀。

```bash
# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
frontend-797f47d685-q5zc6   1/1     Running   0          53m
```

运行kubectl describe deployments，可以清楚地看到Deployment控制的Pod的水平扩展过程。









```
通过kubectl创建deployment
# kubectl create -f deployment.yaml --record
–record参数，使用此参数将记录后续创建对象的操作，方便管理与问题追溯

查看deployment具体信息
# kubectl describe deployment frontend

通过deployment修改Pod副本数量（需要修改yaml文件的spec.replicas字段到目标值，然后替换旧的yaml文件）
# kubectl edit deployment hello-deployment

使用rollout history命令，查看Deployment的历史信息
kubectl rollout history deployment hello-deployment

使用Deployment可以回滚到上一版本，但要加上–revision参数，指定版本号
kubectl rollout history deployment hello-deployment --revision=2

使用rollout undo回滚到上一版本
kubectl rollout undo deployment hello-deployment 

使用–to-revision可以回滚到指定版本
kubectl rollout undo deployment hello-deployment --to-revision=2
	
```





https://kubernetes.io/docs/concepts/workloads/controllers/deployment/