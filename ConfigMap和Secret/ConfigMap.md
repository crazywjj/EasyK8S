[TOC]







# ConfigMap



# 1 ConfigMap介绍

<img src="assets/configmap.png" alt="configmap" style="zoom:67%;" />

ConfigMap 和Secret是Kubernetes 系统上两种特殊类型的存储卷。

ConfigMap是一种用于**存储应用所需配置信息的资源类型**，用于保存配置数据的键值对，可以用来保存单个属性，也可以用来保存配置文件。

ConfigMap注入的方式一般有两种，一种是**挂载存储卷**，一种是**传递变量**。ConfigMap被引用之前必须存在，属于名称空间级别，不能跨名称空间使用，**内容明文显示**。ConfigMap内容修改后，对应的pod必须重启或者重新加载配置。

通过ConfigMap可以方便的做到配置解耦，使得不同环境有不同的配置。相比环境变量，Pod中引用的ConfigMap可以做到实时更新，当您更新ConfigMap的数据后，Pod中引用的ConfigMap会同步刷新。

> **提示：**国内分布式配置中心相关的开源项目有 Di amond （阿里）、 （携程）、Qconf （奇虎360）和 disconf （百度）等。



# 2 ConfigMap典型用法

ConfigMap供容器使用的典型用法如下：

- 生成为容器内的环境变量。
- 设置容器启动命令的启动参数（需设置为环境变量）。 
- 以Volume的形式挂载为容器内部的文件或目录。

ConfigMap以一个或多个key:value的形式保存在Kubernetes系统中供应用使用，既可以用于表示一个变量的值（例如 

apploglevel=info），也可以用于表示一个完整配置文件的内容（例如 server.xml=<?xml...>...）。 

可以通过YAML配置文件或者直接使用`kubectl create configmap` 命令行的方式来创建ConfigMap



# 3 ConfigMap创建

## 3.1 查看命令帮助

```bash
$ kubectl create  configmap --help
Aliases:
configmap, cm  #可以使用cm替代

Examples:
  # Create a new configmap named my-config based on folder bar
  kubectl create configmap my-config --from-file=path/to/bar   #从目录创建  文件名称为键  文件内容为值

  # Create a new configmap named my-config with specified keys instead of file basenames on disk
  kubectl create configmap my-config --from-file=key1=/path/to/bar/file1.txt --from-file=key2=/path/to/bar/file2.txt  #从文件创建 key1为键 文件内容为值

  # Create a new configmap named my-config with key1=config1 and key2=config2
  kubectl create configmap my-config --from-literal=key1=config1 --from-literal=key2=config2  
#直接命令行给定,键为key1 值为config1

  # Create a new configmap named my-config from the key=value pairs in the file
  kubectl create configmap my-config --from-file=path/to/bar   #从文件创建 文件名为键 文件内容为值

  # Create a new configmap named my-config from an env file
  kubectl create configmap my-config --from-env-file=path/to/bar.env

```



## 3.2 基于键值创建

```bash
$ kubectl create configmap nginx-config --from-literal=nginx_port=80 --from-literal=server_name=www.crazyk8s.com
configmap/nginx-config created

$ kubectl get cm
NAME           DATA   AGE
nginx-config   2      34s

$ kubectl get cm nginx-config -o yaml
apiVersion: v1
data:
  nginx_port: "80"
  server_name: www.crazyk8s.com
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-09T06:53:34Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:nginx_port: {}
        f:server_name: {}
    manager: kubectl
    operation: Update
    time: "2021-06-09T06:53:34Z"
  name: nginx-config
  namespace: default
  resourceVersion: "12384623"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-config
  uid: 2cd4c6b5-c378-483a-b579-d77723cdf821

```



## 3.3 基于file文件创建

创建方式：

```
kubectl create configmap <configmap_name> --from-file <path-to-file>
```

具体过程：

```bash
# 创建nginx子配置文件
vim www.conf
server {
  server_name www.crazyk8s.com;
  listen 80;
  root /data/web/html;
}

# 基于文件创建cm
$ kubectl create cm www --from-file=./www.conf  #直接以文件名称为键，文件内容为值,也可以使用--from-file=key=/path/to/file自定义键


$ kubectl get cm www
NAME   DATA   AGE
www    1      47s

$ kubectl get cm www -o yaml
apiVersion: v1
data:
  www.conf: |
    server {
      server_name www.crazyk8s.com;
      listen 80;
      root /data/web/html;
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-09T07:01:52Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:www.conf: {}
    manager: kubectl
    operation: Update
    time: "2021-06-09T07:01:52Z"
  name: www
  namespace: default
  resourceVersion: "12385802"
  selfLink: /api/v1/namespaces/default/configmaps/www
  uid: 4da2953b-9ab0-4fd0-95fa-57ec245ae291

```



## 3.4 基于YAML文件创建



```bash
$ vim nginx-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-conf
data:
  listen: 80
  server_name: www.crazyk8s.com
  
$ kubectl apply -f nginx-configmap.yaml
configmap/nginx-conf created

$ kubectl get cm
NAME         DATA   AGE
nginx-conf   2      7s

$ kubectl get cm nginx-conf -o yaml
apiVersion: v1
data:
  listen: "80"
  server_name: www.crazyk8s.com
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"listen":"80","server_name":"www.crazyk8s.com"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"nginx-conf","namespace":"default"}}
  creationTimestamp: "2021-06-09T07:23:32Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:listen: {}
        f:server_name: {}
      f:metadata:
        f:annotations:
          .: {}
          f:kubectl.kubernetes.io/last-applied-configuration: {}
    manager: kubectl
    operation: Update
    time: "2021-06-09T07:23:32Z"
  name: nginx-conf
  namespace: default
  resourceVersion: "12388889"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-conf
  uid: 64ce6df7-2506-470d-8769-75bb1aae1be2

```



## 3.5 基于目录创建

如果配置文件数量较多且存储于有限的目录中时， kubectl 还提供了基于目录直接将多个文件分别收纳为键值数据 ConfigMap 资源创建方式。将 `--from-file` 后面所跟的路径指向一个目录路径就能将目录下的所有文件一同创建于同 一个ConfigMap 资源中 ，格式如下：

```bash
kubectl create configmap <configmap_name> --from-file=<path-to-directory>
```





# 4 ConfigMap使用

ConfigMap最为常见的使用方式就是在环境变量和Volume中引用。

## 4.1 在环境变量中引用ConfigMap

创建cm信息：

```bash
$ kubectl create configmap test-configmap --from-literal=server_port=88 --from-literal=server_name=www.crazyk8s.com
```

创建pod引用：







































