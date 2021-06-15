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

<img src="assets/20200103132558830.gif" alt="20200103132558830" style="zoom:67%;" />

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

在pod中可以使用 `$(VAR_NAME)` Kubernetes 替换语法在容器的 `command` 和 `args` 部分中使用 ConfigMap 定义的环境变量。例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_LEVEL
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_TYPE
  restartPolicy: Never
  
```

**创建cm信息**

```bash
$ kubectl create configmap nginx-configmap --from-literal=server_port=80 --from-literal=server_name=www.crazyk8s.com
```

### 4.1.1 引用部分键值对

使用`valueFrom`、`configMapKeyRef`、`name`、`key`指定要用的key。

创建pod引用：

vim  [mypod-cm-v1.yaml](assets\mypod-cm-v1.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod-cm-v1
spec:
  containers:
  - name: mypod
    image: busybox
    args: [ "/bin/sh", "-c", "sleep 3000" ]
    env:
    - name: SERVER_PORT
      valueFrom:
        configMapKeyRef:
          name: nginx-configmap
          key: server_port
    - name: SERVER_NAME
      valueFrom:
        configMapKeyRef:
          name: nginx-configmap
          key: server_name

```

创建后查看：

```bash
$ kubectl create -f  mypod-cm-v1.yaml
pod/mypod-cm-v1 created

$ kubectl get po mypod-cm-v1
NAME                      READY   STATUS      RESTARTS   AGE
mypod-cm-v1               1/1     Running     0          4s

$ kubectl exec -it mypod-cm-v1 -- env|grep SERVER
SERVER_PORT=80
SERVER_NAME=www.crazyk8s.com

```

### 4.1.2 引用所有键值对

还可以通过`envFrom`、`configMapRef`、`name`使得configmap中的所有`key/value`键值对都自动变成环境变量。

vim  [mypod-cm-v2.yaml](assets\mypod-cm-v2.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod-cm-v2
spec:
  containers:
  - name: mypod
    image: busybox
    args: [ "/bin/sh", "-c", "sleep 3000" ]
    envFrom:
    - configMapRef:
        name: nginx-configmap

```

创建后查看：

```bash
$ kubectl create -f mypod-cm-v2.yaml
pod/mypod-cm-v2 created

$ kubectl get po mypod-cm-v2
NAME          READY   STATUS    RESTARTS   AGE
mypod-cm-v2   1/1     Running   0          6s

$ kubectl exec -it mypod-cm-v2 -- env|grep server
server_name=www.crazyk8s.com
server_port=80

```



## 4.2 通过volumeMount使用ConfigMap

若ConfigMap 对象中的键值内容较长，那么使用环境变量将其导人会使得变量值占据过多的内存而且不易处理。此类数据通常用于为容器应用提供配置文件，因此将其内容直接作为文件进行引用方为较好的选择。

创建configmap存储卷：

vim  [nginx.conf](yaml\nginx.conf) 

```bash
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  65535;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    gzip  on;

    include /etc/nginx/conf.d/*.conf;
}

```

vim   [www.conf](yaml\www.conf) 

```bash
server {
    listen 80;
    server_name www.crazy.com;

    location / {
    root html;
    index index.html index.htm;
  }
}

```

创建：

```bash
$ kubectl create cm nginx-conf --from-file=nginx.conf --from-file=www.conf
```



### 4.2.1 挂载ConfigMap所有键值到目录

当**容器内目录为空**时，configmap会直接挂载到目录下；目录不为空时，会清空目录下内容，然后挂载；目录不存在时，会先新建目录，然后挂载；

vim  [nginx-cm-demo.yaml](yaml\nginx-cm-demo.yaml) 

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cm-demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx-cm-demo
  template:
    metadata:
      labels:
        app: nginx-cm-demo
    spec:
      volumes:
      - name: config                      #volumes的名称
        configMap:
          name: nginx-conf                #指定使用ConfigMap的名称
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config                    #指定上面的volumes名称
          mountPath: "/etc/app"           #容器挂载的目录

```

创建查看：

```bash
$ kubectl create -f nginx-cm-demo.yaml

$ kubectl get pod nginx-cm-demo-85f9c99b-znj8l
NAME                           READY   STATUS    RESTARTS   AGE
nginx-cm-demo-85f9c99b-znj8l   1/1     Running   0          6m34s

#进入容器验证
$ kubectl exec -it nginx-cm-demo-85f9c99b-znj8l -- /bin/bash
root@nginx-cm-demo-85f9c99b-znj8l:/# cd /etc/app
root@nginx-cm-demo-85f9c99b-znj8l:/etc/app# ls
nginx.conf  www.conf
root@nginx-cm-demo-85f9c99b-znj8l:/etc/app# cat www.conf
server {
          listen       80;
          server_name  www.crazy.com;
          add_header Cache-Control no-cache;

          location / {
            root   /usr/share/nginx/html;
            proxy_read_timeout 220s
            index  index.html index.htm;
          }
            error_page   500 502 503 504  /50x.html;
          location = /50x.html {
            root   html;
          }
      }

```



### 4.2.2 挂载ConfigMap的部分键值到目录

修改`nginx-cm-demo.yaml`部分内容后为： [nginx-cm-demo-V1.yaml](yaml\nginx-cm-demo-V1.yaml) 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cm-demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx-cm-demo
  template:
    metadata:
      labels:
        app: nginx-cm-demo
    spec:
      volumes:
      - name: config                      #volumes的名称
        configMap:
          name: nginx-conf                #指定使用ConfigMap的名称
          items:
          - key: nginx.conf
            path: nginx.conf
            mode: 0644
          - key: www.conf
            path: www.conf
            mode: 0644
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config                    #指定上面的volumes名称
          mountPath: "/etc/app1"   #容器挂载的目录

```

创建后查看：

```bash
$ kubectl create -f nginx-cm-demo.yaml
deployment.apps/nginx-cm-demo created

$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
nginx-cm-demo-859f774bc6-gzkld   1/1     Running   0          3s

$ kubectl exec  -it nginx-cm-demo-859f774bc6-gzkld -- cat /etc/app1/www.conf
server {
          listen       80;
          server_name  www.crazy.com;
          add_header Cache-Control no-cache;

          location / {
            root   /usr/share/nginx/html;
            proxy_read_timeout 220s
            index  index.html index.htm;
          }
            error_page   500 502 503 504  /50x.html;
          location = /50x.html {
            root   html;
          }
      }

```

configMap 存储卷的 items 字段的值是一个对象列表，可嵌套使用的字段有三个，具体如下:

- `key <string>`： 要引用 键名称 ，必选字段。
- `path <string>`： 对应的键于挂载点目录中生成的文件的相对路径 ，可 以不同于键名称，必选字段。
- `mode <integer>` ：文件的权限模型，可用范围为 0至0777



注意：以上方法虽会清空该文件夹的内容，但我们通常配置文件存放的位置，都是一个空的文件夹，并且该方法有一个优点：就是我们在外面apply应用配置文件后，容器里的配置文件会自动刷新到新的配置，此时我们再通过软重启的方式，将容器加载最新的配置文件进行运行

### 4.2.3 单独挂载ConfigMap的键值到文件

如果想单独建ConfigMap的键值内容挂载到某个目录下，而不影响目录下其他文件时，就需要用到容器的 `VolumeMounts `字段中的

`subPath `字段来解决，它可以支持用户从存储卷挂载单个文件或单个目录而非整个存储卷。

vim  [nginx-cm-demo-V2.yaml](yaml\nginx-cm-demo-V2.yaml) 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-cm-demo
  namespace: default
spec:
  selector:
    matchLabels:
      app: nginx-cm-demo
  template:
    metadata:
      labels:
        app: nginx-cm-demo
    spec:
      volumes:
      - name: config
        configMap:
          name: nginx-conf
      containers:
      - name: nginx
        image: nginx
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: config
          mountPath: "/etc/nginx/nginx.conf"
          subPath: nginx.conf
        - name: config
          mountPath: "/etc/nginx/conf.d/www.conf" 
          subPath: www.conf

```



注意：此方法虽然不会覆盖或删除当前文件夹的内容，但是修改配置文件后的内容，容器里不能自动更新，必须对该容器进行删除（不能进行软重启），重新运行，会导致业务中断，短暂数据丢失的可能发生。



# 5 ConfigMap更新

Kubernetes中提供configmap，用来管理应用的配置，configmap具备热更新的能力，但只有通过目录挂载的configmap才具备热更新能力，其余通过环境变量或者subPath挂载的文件都不能动态更新。

在kubernetes中，更新configMap后，pod是不会自动识别configMap中的变动。configMap更新后，如果让pod中引用configMap的变量生效。

通常简单的做法是:
方法1. 删除该pod，让其自动产生一份新的pod.
方法2. 修改pod的配置，让其自动产生一份新的pod.
方法3. 增加一个sidecar，让其监控configMap的变化，来重启pod.

原理就是：通过更新deployment中的Annotations，增加一个version的key,每次需要更新configMap,只要upgrade一次kustomization.yaml中的commonAnnotations->version的值，发布后，pod就会自动重建一次，以此来发现confiMap的新值。





## 5.1 测试ConfigMap热更新

- 基于变量挂载

创建cm和应用引入变量

```bash
$ kubectl create configmap nginx-configmap --from-literal=server_port=80 --from-literal=server_name=www.crazyk8s.com

# 用4.1 中的应用进行测试

$ kubectl create -f  mypod-cm-v1.yaml
pod/mypod-cm-v1 created

$ kubectl get po mypod-cm-v1
NAME                      READY   STATUS      RESTARTS   AGE
mypod-cm-v1               1/1     Running     0          4s

$ kubectl exec -it mypod-cm-v1 -- env|grep SERVER
SERVER_PORT=80
SERVER_NAME=www.crazyk8s.com

# 修改cm键值对
$ kubectl edit cm nginx-configmap

apiVersion: v1
data:
  server_name: www.crazyk8s.com
  server_port: "88"   #修改此处端口
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-11T03:14:29Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:server_name: {}
        f:server_port: {}
    manager: kubectl
    operation: Update
    time: "2021-06-11T03:14:29Z"
  name: nginx-configmap
  namespace: default
  resourceVersion: "12762652"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-configmap
  uid: 87a830c9-5e6d-4180-a228-2ce4feb87668

# 会发现pod中的变量并未发生变化
$ kubectl exec -it mypod-cm-v1 -- env|grep SERVER
SERVER_PORT=80
SERVER_NAME=www.crazyk8s.com

```



- 基于目录挂载

使用4.2案例进行测试。

创建cm：

```bash
$ kubectl create cm nginx-conf --from-file=nginx.conf --from-file=www.conf

$ kubectl get cm nginx-conf -o yaml
apiVersion: v1
data:
  nginx.conf: |
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  65535;
    }


    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;
        tcp_nopush     on;
        keepalive_timeout  65;
        gzip  on;

        include /etc/nginx/conf.d/*.conf;
    }
  www.conf: |
    server {
        listen 80;
        server_name www.crazy.com;

        location / {
        root html;
        index index.html index.htm;
      }
    }
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-11T03:28:12Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:nginx.conf: {}
        f:www.conf: {}
    manager: kubectl
    operation: Update
    time: "2021-06-11T03:28:12Z"
  name: nginx-conf
  namespace: default
  resourceVersion: "12764175"
  selfLink: /api/v1/namespaces/default/configmaps/nginx-conf
  uid: f38780e7-1eab-4a44-8fa9-0eb25f52fa1a

```

创建应用引用cm：

```
$ kubectl create -f nginx-cm-demo.yaml
$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
nginx-cm-demo-85f9c99b-892k2   1/1     Running   0          10s

$ kubectl exec -it nginx-cm-demo-85f9c99b-892k2 -- ls /etc/app
nginx.conf  www.conf

$ kubectl exec -it nginx-cm-demo-85f9c99b-892k2 -- cat /etc/app/www.conf
server {
    listen 80;
    server_name www.crazy.com;

    location / {
    root html;
    index index.html index.htm;
  }
}

```

修改cm：

```bash
$ kubectl edit cm nginx-conf
www.conf: |
    server {
        listen 88;   #修改此处端口为88
        server_name www.crazy.com;

# 等待约10s后查看，会发现pod内的配置已经发生变化
$ kubectl exec -it nginx-cm-demo-85f9c99b-892k2 -- cat /etc/app/www.conf
server {
    listen 88;
    server_name www.crazy.com;

    location / {
    root html;
    index index.html index.htm;
  }
}

```

此时pod内已经发现ConfigMap的配置已经发生变化，但是pod内容器其实加载的还是旧的配置，需要重启或者重建pod才能加载新的配置。









