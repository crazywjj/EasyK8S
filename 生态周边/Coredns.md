[TOC]





# Coredns

官网 ：https://coredns.io/ 

CoreDNS是Golang编写的一个插件式DNS服务器，是Kubernetes 1.13 后所内置的默认DNS服务器；由于其灵活性，它可以在多种环境中使用。

CoreDNS 其实就是一个 DNS 服务，而 DNS 作为一种常见的服务发现手段，所以很多开源项目以及工程师都会使用 CoreDNS 为集群提供服务发现的功能，Kubernetes 就在集群中使用 CoreDNS 解决服务发现的问题。

CoreDNS 的大多数功能都是由插件来实现的，插件和服务本身都使用了 Caddy 提供的一些功能，所以项目本身也不是特别的复杂。



# 1、下载和配置 coredns

```bash
cd /opt/k8s/work
git clone https://github.com/coredns/deployment.git
mv deployment coredns-deployment
```

# 2、创建 coredns

```bash
cd /opt/k8s/work/coredns-deployment/kubernetes
source /opt/k8s/bin/environment.sh
./deploy.sh -i ${CLUSTER_DNS_SVC_IP} -d ${CLUSTER_DNS_DOMAIN} | kubectl apply -f -
```

# 3、检查 coredns 功能

```shell
$ kubectl get all -n kube-system -l k8s-app=kube-dns
NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-59845f77f8-l7rjh   1/1     Running   0          37h

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP,9153/TCP   37h

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   1/1     1            1           37h

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-59845f77f8   1         1         1       37h
```

查看创建的coredns的pod状态 ：

```bash
$ kubectl describe pod/coredns-59845f77f8-l7rjh -n kube-system
Name:                 coredns-59845f77f8-l7rjh
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Node:                 k8s-m01/10.0.0.61
Start Time:           Fri, 10 Apr 2020 20:24:29 +0800
Labels:               k8s-app=kube-dns
                      pod-template-hash=59845f77f8
Annotations:          <none>
Status:               Running
IP:                   172.30.40.3
IPs:
  IP:           172.30.40.3
Controlled By:  ReplicaSet/coredns-59845f77f8
```



# 4、新建一个 Deployment

```yml
cd /opt/k8s/work
cat > my-nginx.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: daocloud.io/library/nginx:latest
        ports:
        - containerPort: 80
EOF
kubectl create -f my-nginx.yaml
```

export 该 Deployment, 生成 my-nginx 服务：

```bash
$ kubectl expose deploy my-nginx
$ kubectl get services --all-namespaces |grep my-nginx
default       my-nginx     ClusterIP   10.254.97.143   <none>        80/TCP                   35s
```

创建一个dns测试工具Pod，查看 /etc/resolv.conf 是否包含 kubelet 配置的 `--cluster-dns` 和 `--cluster-domain`：

```yml
cd /opt/k8s/work
cat > dnsutils-ds.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: dnsutils-ds
  labels:
    app: dnsutils-ds
spec:
  type: NodePort
  selector:
    app: dnsutils-ds
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dnsutils-ds
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      app: dnsutils-ds
  template:
    metadata:
      labels:
        app: dnsutils-ds
    spec:
      containers:
      - name: my-dnsutils
        image: tutum/dnsutils:latest
        command:
          - sleep
          - "3600"
        ports:
        - containerPort: 80
EOF
kubectl create -f dnsutils-ds.yml
```

# 5、查看dnsutils的pod状态

确保是`Running`；如不是请查看`kubectl describe pod dnsutils-ds-2rlkm`

```bash
$ kubectl get pods -lapp=dnsutils-ds
NAME                READY   STATUS              RESTARTS   AGE
dnsutils-ds-2rlkm   1/1     Running			    0          5m28s
dnsutils-ds-9hw5m   1/1     Running             0          5m28s
dnsutils-ds-mlxnr   1/1     Running             0          5m28s
```

查看pod的`/etc/resolv.conf`

```bash
$ kubectl -it exec dnsutils-ds-26cpm  cat /etc/resolv.conf
nameserver 10.254.0.2
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

```bash
$ kubectl -it exec dnsutils-ds-26cpm nslookup kubernetes
Server:		10.254.0.2
Address:	10.254.0.2#53

Name:	kubernetes.default.svc.cluster.local
Address: 10.254.0.1
```

```bash
$ kubectl -it exec dnsutils-ds-26cpm nslookup www.baidu.com
Server:		10.254.0.2
Address:	10.254.0.2#53

Non-authoritative answer:
www.baidu.com	canonical name = www.a.shifen.com.
Name:	www.a.shifen.com
Address: 61.135.169.121
Name:	www.a.shifen.com
Address: 61.135.169.125
```

 发现可以将服务 my-nginx 解析到上面它对应的 Cluster IP 10.254.97.143 :

```bash
$ kubectl -it exec dnsutils-ds-26cpm nslookup my-nginx
Server:		10.254.0.2
Address:	10.254.0.2#53

Name:	my-nginx.default.svc.cluster.local
Address: 10.254.97.143
```















