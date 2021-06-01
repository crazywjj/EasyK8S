[TOC]





# Pod健康状态

Kubernetes对 Pod 的健康状态可以通过三类探针（Probe）来检查： `LivenessProbe` 和 `ReadinessProbe`、`startupProbe`，kubelet定期执行这三类探针来诊断容器的健康状况。

- **livenessProbe**：判断容器是否正在运行（Running状态）。如果**存活探测**失败，则 kubelet 会杀死容器，并且根据重启策略进行处理。如果容器不包含存活探针，则默认状态为 `Success`。

- **readinessProbe**：判断容器服务是否可用（Ready状态）。如果**就绪探测**失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。初始延迟之前的就绪状态默认为 `Failure`。如果容器不提供就绪探针，则默认状态为 `Success。

- **startupProbe**：判断容器内的应用程序是否已启动。如果配置了**启动探测**，在则在启动探针状态为 Succes 之前，其他所有探针都处于无效状态，直到它成功后其他探针才起作用。如果启动探测失败，kubelet 将杀死容器，容器将服从其重启策略。如果容器没有配置启动探测，则默认状态为 Success。（这个1.17版本增加的）

  

[Probe](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#probe-v1-core)（探针） 是由 [kubelet](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/) 对容器执行的定期诊断。 要执行诊断，需要 kubelet 调用由容器实现的 [Handler](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#handler-v1-core) （处理程序）。

每个探针有三种类型的处理程序：

- [ExecAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#execaction-v1-core)： 在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- [TCPSocketAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#tcpsocketaction-v1-core)： 对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- [HTTPGetAction](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#httpgetaction-v1-core)： 对容器的 IP 地址上指定端口和路径执行 HTTP Get 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。

每次探测都将获得以下三种结果之一：

- `Success`：容器诊断通过。
- `Failure`：容器诊断失败。
- `Unknown`：诊断失败，因此不应采取任何措施。



# 1 livenessProbe存活探测

有不少应 程序长时间持续运行后会逐渐转为不可用状态 ，并且仅能通过重启操作恢复， Kubemetes 的容器存活性探测机制可发现这类的问题，并依据探测结果结合重启策略进行重启。存活性探测是隶属于容器级别的配置， kubelet 可基于它判定何时需要重启一个容器。

Pod spec 为容器列表中的相应容器定义其 用的探针（存活性探测机制）即可启用存活性探测。



## 1.1 设置 exec 探针

exec 类型的探针通过在目标容器中执行由用户自定义的命令来判定容器的健康状态，若命令状态返回值为 则表示“成功”通过检测，其值均为“失败”状态。“`spec.containers.livenessProbe exec `”字段用于定义此类检测，它只有只有一个可用属性“ command ”，用于指定要执行的命令。

定义资源清单文件 [liveness-exec.yaml](yaml\liveness-exec.yaml) 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness-exec
  name: liveness-exec
spec:
  containers:
  - name: liveness-demo
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 60; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - test
        - -e
        - /tmp/healthy

```

上面的资源清单中定义了一个 Pod 对象， 基于 busybox 镜像启动一个运行“ touch /tmp/healthy; sleep 60; rm -rf/tmp/healthy; sleep 600。”命令的容器，此命令在容器启动时创建 /tmp/healthy 件，并于 60 秒之后将其删除。存活性探针运行“ test -e tmp healthy ”命令检查 /tmp/healthy 文件的存在性，若文件存在则返回状态码 ，表示成功通过测试。换句话说，60 秒之内查看pod的Events详细信息，其存活性探测不会出现错误。而超过 60 秒之后，再次查看其详细信息可以发现，存活性探测出现了故障，并且隔更长一段时间之后再查看甚至还可以看到容器重启的相关信息：

```bash
$ kubectl describe pod liveness-exec
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  8m43s                  default-scheduler  Successfully assigned default/liveness-exec to k8s-node42
  Normal   Created    4m18s (x3 over 8m26s)  kubelet            Created container liveness-demo
  Normal   Started    4m18s (x3 over 8m26s)  kubelet            Started container liveness-demo
  Warning  Unhealthy  2m58s (x9 over 7m18s)  kubelet            Liveness probe failed:
  Normal   Killing    2m58s (x3 over 6m58s)  kubelet            Container liveness-demo failed liveness probe, will be restarted
  Normal   Pulling    2m28s (x4 over 8m31s)  kubelet            Pulling image "busybox"
  Normal   Pulled     2m22s (x4 over 8m26s)  kubelet            Successfully pulled image "busybox"

```

需要特别说明的 exec 指定命令运于容器 ，会耗容器资源配额，另外， 考虑探测操作的效率本身 ，探测的命令应该尽可能简单和轻量。



## 1.2 设置 http 探针

基于 HTTP的探测（HTTPGetAction）向目标容器发起一个HTTP请求，根据其响应的状态码来判断。响应码 2xx 3xx 时表示检测通过 。`spec conta in ers.livenessProbe.`字段用于定义此类检测，它的可用配置字段包括如下几个：

- `host <string>`：请求的主机机地址，默认为 Pod IP ；也可以在 httpHeaders 中使用 “Host ：”来定义。
- `port <string>`： 请求的端口 ，必选字段。
- `httpHeaders <[]Object>`： 自定义的请求报文头部。
- `path <string>` ：请求的 HTTP 资源路径，即 URL path。
- `scheme`：建立连接使用的协议，仅可为 HTTP或者HTTPS ，默认认为 HTTP。

下面是一个定义在资源清单文件 [liveness-http.yaml](yaml\liveness-http.yaml)  的示例，它通过 lifecycle 中的 `postStart hook` 创建一个专用于 httpGet 测试的页面文件healthz：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness-demo
    image: nginx
    ports:
    - name: http
      containerPort: 80
    lifecycle:
      postStart:
        exec:
          command:
          - /bin/sh
          - -c
          - 'echo Healty > /usr/share/nginx/html/healthz'
    livenessProbe:
      httpGet:
        path: /healthz
        port: http
        scheme: HTTP

```

上面清单文件中定义的 http Get 测试中，请求的资源路径为“ /healthz ”，地址默认为Pod IP，端口使用了容器中定义的端口名称 HTTP ，这也是明确为容器指明要暴露的端口的用途之一。

创建此Pod：

```bash
kubectl create -f liveness-http.yaml
```

查看Events信息是正常的：

```bash
$ kubectl describe pod liveness-http
...
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  29s        default-scheduler  Successfully assigned default/liveness-http to k8s-node42
  Normal  Pulling    17s        kubelet            Pulling image "nginx"
  Normal  Pulled     <invalid>  kubelet            Successfully pulled image "nginx"
  Normal  Created    <invalid>  kubelet            Created container liveness-demo
  Normal  Started    <invalid>  kubelet            Started container liveness-demo

```

然后，删除由 postStart hook 创建的测试页面 healthz:

```bash
$ kubectl exec -it liveness-http -- rm -f /usr/share/nginx/html/healthz
$ kubectl describe pod liveness-http
...
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  3m26s               default-scheduler  Successfully assigned default/liveness-http to k8s-node42
  Normal   Pulling    7s (x2 over 3m13s)  kubelet            Pulling image "nginx"
  Warning  Unhealthy  7s (x3 over 27s)    kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    7s                  kubelet            Container liveness-demo failed liveness probe, will be restarted
  Normal   Pulled     3s (x2 over 2m53s)  kubelet            Successfully pulled image "nginx"
  Normal   Created    3s (x2 over 2m52s)  kubelet            Created container liveness-demo
  Normal   Started    2s (x2 over 2m52s)  kubelet            Started container liveness-demo
```

查看其详细的状态信息，事件输出中的信息可以表明HTTP探测失败，容器被杀掉后进行了重新创建。

一般来说， HTTP 类型的探测操作应该针对专用的 URL 路径进行，例如，前面示例中特别为其准备的 "healthz" 另外，此 URL 路径对应的 web 资源应该以轻量化的方式在内部对应用程序的各关键组件进行全面检测以确保它们可正常向客户端提供完整的服务。

需要注意的是，这种检测方式仅对分层架构中的当前一层有效，例如，它能检测应用程序工作正常与否的状态，但重启操作却无法解决其后端服务（如数据库或缓存服务）导致的故障。此时，容器可能会被一次次的重启，直到后端服务恢复正常为。其他两种检测方式也存在类似的问题。



## 1.3 设置 tcp 探针

基于 TCP 的存活性探测 （ TCPSocketAction ）用于向容器的特定端口发起 TCP 请求并尝试建立连接进行结果判定，连接建立成功即为通过检测。相比来说，它比基于 HTTP 的探测要更高效更节约资源，但精准度略低，毕竟连接建立成功未必意味着页面资源可用。`spec.containrs.livenessProbe.tcpSocket `字段用于定义此类检测，它主要包含以下两个可用的属性：

- `host <string>` ：请求连接的目标 IP 地址，默认为 Pod IP。
- `port <string> `：请求连接的目标端口，必选字段。

下面是 个定义在资源清单文件 [liveness-tcp.yaml](yaml\liveness-tcp.yaml) 中的示例，，它向 Po IP 80/tcp端口发起连接请求，并根据连接建的状态判定测试结果：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-tcp
spec:
  containers:
  - name: liveness-tcp-demo
    image: nginx
    ports:
    - name: tcp
      containerPort: 80
    livenessProbe:
      tcpSocket:
        port: tcp

```



## 1.4 存活性探测行为属性

使用 kubectl describe 令查看配置了存活性探测的 Pod 对象的详细信息时，其相关容器中会输出类似如下 行的内容：

```bash
Liveness : exec [test -e /tmp/healthy] delay=Os timeout=ls period=lOs #success=l #falure=3
```

给出了探测方式及其额外的配置属性 delay、timeout、period、success、failure 及其各自的相关属性值。用户没有明确定义这些字段时，它们会使用各自的默认值，例如上面显示出的设定。这些属性信息可通过`spec.containers.livenessProbe` 的如下属性字段来给出。

- `initialDelaySeconds <integer> `：存活性探测延迟时长，即容器启动多久之后再开始第一次探测操作，显示为 delay 属性；默认为0秒，即容器启动后立刻开始进行探测。

- `timeoutSeconds <integer＞`：存活性探测的超时时长，显示为 timeout 属性， 默认为 1s，最小值也是1s。

- `periodSeconds <integer> `：存活性探测的频度，显示为 period 属性，默认为10s ，最小值为1s；过高的频率会对 Pod 对象带来较大的额外开销，而过低的频率又会使得对错误的反应不及时。
- `successThreshold <integer>` ：处于失败状态时，探测操作至少连续多少次的成功才被认为是通过检测， 显示为＃success 属性，默认值为 1，最小值也为1。
- `failureThreshold `：处于成功状态时，探测操作连续多少次的失败才被视为是检测不通过，显示为＃failure 性，默认值为3，最小值为1。

如下示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
spec:
      containers:
        livenessProbe:            # 存活健康状态检查
          exec:
            command:
            - touch
            - /tmp/healthy
          initialDelaySeconds: 30 # 初始化时间，单位：秒
          timeoutSeconds: 5       # 探测超时时长，单位：秒
          periodSeconds: 30       # 探测时间间隔，单位：秒
          successThreshold: 1     # 失败后探测成功的最小连续成功次数
          failureThreshold: 5     # 最大失败次数
```





# 2 readinessProbe就绪探测

Pod 对象启动后，容器应用通常 段时间才能完成其初始化过程，例如加载配置或数据，甚至有些程序需要运行某类的预热过程，若在此阶段完成之前即接人客户端的请求，势必会因为等待太久而影响用户体验。因此，应该避免于 Pod 对象启动后立即让其处理客户端请求，而是等待容器初始化工作执行完成并转为**“就绪”状态**，尤其是存在其他提供相同服务的 Pod 的场景更是如此。

与存活性探测机制相同，就绪性探测也支持 Exec、 HTTP GET 和 TCP Socket 三种探测方式，且各自的定义机制也都相同。但与存活性探测触发的操作不同的是，探测失败时，就**绪性探测不会杀死或重启容器以保证其健康性**，而是通知其尚未就绪，并触发依赖于其就绪状态的操作（例如，从 Service 对象中移除此 Pod 对象）以确保不会有客户端请求接入此Pod对象。不过，即便是在运行过程中， Pod 就绪性探测依然有其价值所在，例如 PodA 赖到的 Pod B 因网络故障等原因而不可用时， Pod  A上的服务应该转为未就绪状态，以免无法向客户端提供完整的响应。

例如：

一个简单的示例如下面的配置清单 [readiness-exec.yaml](yaml\readiness-exec.yaml)  所示，它会在 Pod 对象创建完成 5 秒钟后使用` test -e /tmp/ready `来探测容器的就绪性，命令执行成功为就绪，探测周期为 5 秒钟： 

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: readiness-exec
  name: readiness-exec
spec:
  containers:
  - name: readiness-demo
    image: busybox
    args: ["/bin/sh", "-c", "while true; do rm -f /tmp/ready; sleep 10; touch /tmp/ready; sleep 60; done"]
    readinessProbe:
      exec:
        command: ["test", "-e", "/tmp/ready"]
      initialDelaySeconds: 5
      periodSeconds: 5

```

执行并实时监视变动：

```bash
# kubectl get -w 监视其资源变动信息；另起一个窗口运行如下命令：
$ kubectl get pod -l test=readiness-exec -w

$ kubectl create -f readiness-exec.yaml
pod/readiness-exec created

# 会看到整个pod从创建到Running最后到READY的状态
$ kubectl get pod -l test=readiness-exec -w
NAME             READY   STATUS    RESTARTS   AGE
readiness-exec   0/1     Pending   0          0s
readiness-exec   0/1     Pending   0          0s
readiness-exec   0/1     ContainerCreating   0          0s
readiness-exec   0/1     Running             0          6s
readiness-exec   1/1     Running             0          38s

```

未定义就绪性探测的 Pod 象在 Pod 入 Running 状态后将立即就绪。在容器需要时间进行初始化的场景中，在应用真正就绪之前必然无法正常响应客户端请求 ，因此， 生产实践中，必须为关键性 Pod 资源中的容器定义就绪性探测。





# 3 startupProbe启动探针
