[TOC]



# Volume存储卷

Volume（存储卷）是Pod中能够被多个容器访问的共享目录。首先，Kubernetes中的Volume被定义在Pod上，被一个Pod里的多个容器挂载到具体的文件目录下；其次，Kubernetes中的Volume与Pod的生命周期相同，但与容器的生命周期无关，当容器终止或者重启时，Volume中的数据也不会丢失。

Volume的使用前，需要Pod指定volume的类型和内容（spec.volume）和挂载点（spec.containers.volumeMounts）两个信息。

**Kubernetes支持Volume类型有：**

- emptyDir
- hostPath
- gcePersistentDisk
- awsElasticBlockStore
- nfs
- iscsi
- fc (fibre channel)
- flocker
- glusterfs
- rbd
- cephfs
- gitRepo
- secret
- persistentVolumeClaim
- downwardAPI
- projected
- azureFileVolume
- azureDisk
- vsphereVolume
- Quobyte
- PortworxVolume
- ScaleIO
- StorageOS
- local

**常用的数据卷：**
• 本地（hostPath，emptyDir）
• 网络（NFS，Ceph，GlusterFS）
• 公有云（AWS EBS）
• K8S资源（configmap，secret）

# 1 emptyDir（临时存储卷）

一个emptyDir Volume是在Pod分配到Node时创建的。初始内容为空，并且无须指定宿主机上对应的目录文件，pod创建时创建，pod移除时移除。注：删除容器不影响emptyDir。

```yml
apiVersion: v1
kind: Pod        #类型是Pod
metadata:
  labels:
    name: redis
    role: master        #定义为主redis
  name: redis-master
spec:
  containers:
    - name: master
      image: redis:latest
      env:        #定义环境变量
        - name: MASTER
          value: "true"
      ports:        #容器内端口
        - containerPort: 6379
      volumeMounts:        #容器内挂载点
        - mountPath: /data
          name: redis-data        #必须有名称
  volumes:
    - name: redis-data        #跟上面的名称对应
      emptyDir: {}        #宿主机挂载点
```

Emptydir创建后，在宿主机上的访问路径为`/var/lib/kubelet/pods/<pod uid>/volumes/kubernetes.io~empty-dir/redis-data`,如果在此目录中创建删除文件，都将对容器中的/data目录有影响。

Kubernetes支持几种不同类型的临时卷，用于不同目的：

- emptyDir：Pod启动时为空，存储来自本地kubelet基本目录（通常是根磁盘）或RAM；
- configMap， downwardAPI， secret：将不同种类的Kubernetes数据注入Pod；
- CSI短暂卷：类似收市成交量多种类，而是通过特殊的提供 CSI驱动 其专门支持此功能；
- 通用临时卷，可以由所有支持持久卷的存储驱动程序提供；

emptyDir，configMap，downwardAPI，secret是作为 本地存储短暂。它们由每个节点上的kubelet管理。

CSI临时卷必须由第三方CSI存储驱动程序提供。

# 2 hostPath（节点存储卷）

hostPath允许挂载Node上的文件系统到Pod里面去。如果Pod需要使用Node上的文件，可以使用hostPath。

挂载宿主机的/tmp目录到Pod容器的/data目录：

```yml
apiVersion: v1
kind: Pod        #类型是Pod
metadata:
  labels:
    name: redis
    role: master        #定义为主redis
  name: redis-master
spec:
  containers:
    - name: master
      image: redis:latest
      env:        #定义环境变量
        - name: MASTER
          value: "true"
      ports:        #容器内端口
        - containerPort: 6379
      volumeMounts:        #容器内挂载点
        - mountPath: /data
          name: redis-data        #必须有名称
  volumes:
    - name: redis-data        #跟上面的名称对应
      hostPath: 
        path: /data      #宿主机挂载点
```



# 3 gcePersistentDisk

gcePersistentDisk可以挂载GCE上的永久磁盘到容器，需要Kubernetes运行在GCE的VM中。与emptyDir不同，Pod删除时，gcePersistentDisk被删除，但[Persistent Disk](http://cloud.google.com/compute/docs/disks) 的内容任然存在。这就意味着gcePersistentDisk能够允许我们提前对数据进行处理，而且这些数据可以在Pod之间“切换”。

**提示：使用gcePersistentDisk，必须用gcloud或使用GCE API或UI 创建PD**

创建PD

使用GCE PD与pod之前，需要创建它

```
gcloud compute disks create --size=500GB --zone=us-central1-a my-data-disk
```

示例

```yml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    # This GCE PD must already exist.
    gcePersistentDisk:
      pdName: my-data-disk
      fsType: ext4
```



# 4 awsElasticBlockStore

awsElasticBlockStore可以挂载AWS上的EBS盘到容器，需要Kubernetes运行在AWS的EC2上。与emptyDir Pod被删除情况不同，Volume仅被卸载，内容将被保留。这就意味着awsElasticBlockStore能够允许我们提前对数据进行处理，而且这些数据可以在Pod之间“切换”。

提示：必须使用aws ec2 create-volumeAWS API 创建EBS Volume，然后才能使用。

**创建EBS Volume**

在使用EBS Volume与pod之前，需要创建它。

```bash
aws ec2 create-volume --availability-zone eu-west-1a --size 10 --volume-type gp2
```

AWS EBS配置示例

```yml
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-ebs
      name: test-volume
  volumes:
  - name: test-volume
    # This AWS EBS volume must already exist.
    awsElasticBlockStore:
      volumeID: <volume-id>
      fsType: ext4
```



# 5 NFS（网络存储卷）

使用NFS网络文件系统提供的共享目录存储数据时，需要在系统中部署一个NFS Server（建议直接部署到服务器）。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-web
spec:
  containers:
    - name: web
      image: nginx
      imagePullPolicy: Never        #如果已经有镜像，就不需要再拉取镜像
      ports:
        - name: web
          containerPort: 80
          hostPort: 80        #将容器的80端口映射到宿主机的80端口
      volumeMounts:
        - name : nfs        #指定名称必须与下面一致
          mountPath: "/usr/share/nginx/html"        #容器内的挂载点
  volumes:
    - name: nfs            #指定名称必须与上面一致
      nfs:            #nfs存储
        server: 192.168.66.50        #nfs服务器ip或是域名
        path: "/test"                #nfs服务器共享的目录
```


cephfs、glusterfs、iscsi、rbd、storageos可以参考如下链接：

https://github.com/kubernetes/examples/tree/master/volumes