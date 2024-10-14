# 08 探针livenessAandReadiness probe

通过deployment创建pod，非常方便。也有了高可用。思考一个问题：pod状态一直是running，但是里面的文件丢失了怎么办？deploy是管不了的。 需要有另外一种探测方式，叫探针，就是探察探测这个pod是否是正常工作的。 它会根据发现的问题，处理的方式也会不同，有两种探测方式：liveness probe 存活探针/readiness probe 就绪探针 1.16版本新增startup probe 启动探针

### 1.liveness probe

重启大法，或者重装（这里重启表示的是重建）。 那如何知道是否是好的呢？3种方式：command/httpget/tcp

command方式 \[root@kmaster \~]# mkdir /pro \[root@kmaster \~]# cd /pro \[root@kmaster pro]# vim pod1.yaml \[root@kmaster pro]# cat pod1.yaml apiVersion: v1 kind: Pod metadata: labels: test: liveness name: liveness-exec spec: containers:

* name: liveness image: busybox imagePullPolicy: IfNotPresent args:
  * /bin/sh
  * \-c
  * touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600 livenessProbe: exec: command:
    * cat
    * /tmp/healthy initialDelaySeconds: 10 periodSeconds: 5

\[root@kmaster pro]# kubectl get pod NAME READY STATUS RESTARTS AGE liveness-exec 1/1 Running 0 4s

\[root@kmaster pro]# kubectl get pod -w （持续观察） NAME READY STATUS RESTARTS AGE liveness-exec 1/1 Running 0 83s

每隔0.5秒或者1秒动态观察 \[root@kmaster pro]# watch -n .5 'kubectl get pod' \[root@kmaster pro]# watch -n 1 'kubectl get pod'

可以看到pod一直在运行中，但是文件还存在吗？没了 \[root@kmaster \~]# kubectl exec -ti liveness-exec -- bash error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "b563984a2c0a5a2f71fb5f42359bdab4b3ba34f66183632393c9b7ff73dc7cc2": OCI runtime exec failed: exec failed: unable to start container process: exec: "bash": executable file not found in $PATH: unknown

\#注意：这里报错，原因是因为busybox没有bash，换成/bin/sh即可 \[root@kmaster \~]# kubectl exec -ti liveness-exec -- /bin/sh / # ls /tmp/ / # exit \[root@kmaster \~]# kubectl exec -ti liveness-exec -- ls /tmp

也就是容器里面的文件已经被删除了，但是pod状态依然是running正常运行。

再次修改yaml，使用command方式进行探测。 \[root@kmaster pro]# vim pod1.yaml \[root@kmaster pro]# cat pod1.yaml apiVersion: v1 kind: Pod metadata: labels: test: liveness name: liveness-exec spec: containers:

* name: liveness image: busybox imagePullPolicy: IfNotPresent args:
  * /bin/sh
  * \-c
  * touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600 livenessProbe: exec: command:
    * cat
    * /tmp/healthy initialDelaySeconds: 5 periodSeconds: 5

***

```
  initialDelaySeconds: 5  启动5秒之内不探测
  periodSeconds: 5  每隔5s探测一次
```

例如：执行命令成功后，返回0；没有成功，返回1，语法错误返回127，等等 \[root@kmaster pro]# cat /etc/issue \S Kernel \r on an \m

\[root@kmaster pro]# echo $? 0 \[root@kmaster pro]# cat /etc/aaaa cat: /etc/aaaa: No such file or directory \[root@kmaster pro]# echo $? 1

如果等会容器反馈0，说明文件还在，返回1，说明文件被删除了。 \[root@kmaster pro]# kubectl apply -f pod1.yaml pod/liveness-exec created \[root@kmaster pro]# kubectl exec -ti liveness-exec -- ls /tmp healthy \[root@kmaster pro]# kubectl exec -ti liveness-exec -- ls /tmp healthy

创建好之后，查看容器文件。等待30秒，被删除 \[root@kmaster pro]# kubectl exec -ti liveness-exec -- ls /tmp \[root@kmaster pro]# kubectl exec -ti liveness-exec -- ls /tmp

这时候，因为探测到了文件被删除了，所以，pod会进行重建。 \[root@kmaster pro]# kubectl get pod liveness-exec -w NAME READY STATUS RESTARTS AGE liveness-exec 1/1 Running 0 6s liveness-exec 1/1 Running 1 (0s ago) 76s liveness-exec 1/1 Running 2 (1s ago) 2m32s

其他参数 https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

http方式

\[root@kmaster pro]# cat pod2.yaml apiVersion: v1 kind: Pod metadata: labels: test: liveness name: pod2 spec: containers:

* name: pod2 image: nginx imagePullPolicy: IfNotPresent livenessProbe: httpGet: path: /index.html port: 80 initialDelaySeconds: 3 periodSeconds: 3

nginx默认网站根目录 /usr/share/nginx/html/index.html apache（httpd）默认网站根目录 /var/www/html/index.html /：/usr/share/nginx/html/index.html

\#执行创建pod \[root@kmaster pro]# kubectl apply -f pod2.yaml pod/pod2 created

\#持续观察pod状态 \[root@kmaster pro]# kubectl get pod pod2 -w NAME READY STATUS RESTARTS AGE pod2 0/1 ContainerCreating 0 13s pod2 1/1 Running 0 64s

\[root@kmaster pro]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod2 1/1 Running 0 79s 10.244.195.136 knode1

\#进入pod查看index文件 \[root@kmaster pro]# kubectl exec -ti pod2 -- bash root@pod2:/# cd /usr/share/nginx/html/ root@pod2:/usr/share/nginx/html# ls 50x.html index.html

\#删除index文件，等待一会，自动退出了，因为探针探测到了，直接重建容器。 root@pod2:/usr/share/nginx/html# rm -rf index.html root@pod2:/usr/share/nginx/html# ls 50x.html root@pod2:/usr/share/nginx/html# command terminated with exit code 137

\[root@kmaster pro]# kubectl get pod pod2 -w NAME READY STATUS RESTARTS AGE pod2 0/1 ContainerCreating 0 13s pod2 1/1 Running 0 64s pod2 1/1 Running 1 (1s ago) 2m43s

\#再次查看文件(有了) \[root@kmaster pro]# kubectl exec -ti pod2 -- bash root@pod2:/# cd /usr/share/nginx/html/ root@pod2:/usr/share/nginx/html# ls 50x.html index.html

\[root@kmaster pro]# kubectl describe pod pod2

Type Reason Age From Message

***

Normal Scheduled 7m18s default-scheduler Successfully assigned default/pod2 to knode1 Normal Pulling 7m18s kubelet Pulling image "nginx" Normal Pulled 6m14s kubelet Successfully pulled image "nginx" in 1m3.641978063s (1m3.641983639s including waiting) Normal Created 4m36s (x2 over 6m14s) kubelet Created container pod2 Normal Started 4m36s (x2 over 6m14s) kubelet Started container pod2 Warning Unhealthy 4m36s (x3 over 4m42s) kubelet Liveness probe failed: HTTP probe failed with statuscode: 404 Normal Killing 4m36s kubelet Container pod2 failed liveness probe, will be restarted Normal Pulled 4m36s kubelet Container image "nginx" already present on machine

tcpSocket方式 https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

\[root@kmaster pro]# cat pod3.yaml apiVersion: v1 kind: Pod metadata: name: pod3 labels: app: goproxy spec: containers:

* name: nginx image: nginx imagePullPolicy: IfNotPresent livenessProbe: tcpSocket: port: 81 initialDelaySeconds: 3 periodSeconds: 3

tcp三次握手，因为连不上80端口，所以导致失败。反复的重启。 \[root@kmaster pro]# kubectl get pod pod3 -w NAME READY STATUS RESTARTS AGE pod3 0/1 ContainerCreating 0 10s pod3 1/1 Running 0 46s pod3 1/1 Running 1 (0s ago) 101s pod3 1/1 Running 2 (0s ago) 2m41s

### 2.readiness probe

liveness probe 存活探针通过重启来解决问题 readiness probe 检测到问题并不重启，只是svc接受的请求不再转发给此pod \[root@kmaster pro]# kubectl create deployment web1 --image=nginx --dry-run=client -o yaml > web1.yaml \[root@kmaster pro]# ls pod1.yaml pod2.yaml pod3.yaml web1.yaml \[root@kmaster pro]# vim pod2.yaml \[root@kmaster pro]# vim web1.yaml \[root@kmaster pro]# cat web1 cat: web1: No such file or directory \[root@kmaster pro]# cat web1.yaml apiVersion: apps/v1 kind: Deployment metadata: creationTimestamp: null labels: app: web1 name: web1 spec: replicas: 1 selector: matchLabels: app: web1 strategy: {} template: metadata: creationTimestamp: null labels: app: web1 spec: containers: - image: nginx name: nginx resources: {} imagePullPolicy: IfNotPresent readinessProbe: httpGet: path: /index.html port: 80 initialDelaySeconds: 3 periodSeconds: 3 status: {}

创建deploy \[root@kmaster pro]# kubectl apply -f web1.yaml deployment.apps/web1 created \[root@kmaster pro]# kubectl get pod NAME READY STATUS RESTARTS AGE web1-68945889d4-6dd5l 0/1 Running 0 2s web1-68945889d4-q76pm 0/1 Running 0 2s web1-68945889d4-wwbdf 0/1 Running 0 2s \[root@kmaster pro]# kubectl get pod NAME READY STATUS RESTARTS AGE web1-68945889d4-6dd5l 1/1 Running 0 53s web1-68945889d4-q76pm 1/1 Running 0 53s web1-68945889d4-wwbdf 1/1 Running 0 53s

\[root@kmaster pro]# kubectl exec -ti web1-68945889d4-6dd5l -- bash root@web1-68945889d4-6dd5l:/# echo host01 > /usr/share/nginx/html/host.html root@web1-68945889d4-6dd5l:/# exit exit \[root@kmaster pro]# kubectl exec -ti web1-68945889d4-q76pm -- bash root@web1-68945889d4-q76pm:/# echo host02 > /usr/share/nginx/html/host.html root@web1-68945889d4-q76pm:/# exit exit \[root@kmaster pro]# kubectl exec -ti web1-68945889d4-wwbdf -- bash root@web1-68945889d4-wwbdf:/# echo host03 > /usr/share/nginx/html/host.html root@web1-68945889d4-wwbdf:/# exit exit

每个pod里面都创建一个 host.html 文件，内容分别为 host01/host02/host03 创建svc，会转发到后端的pod上。

\[root@kmaster pro]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 10d \[root@kmaster pro]# kubectl expose deployment web1 --port 80 --target-port 80 service/web1 exposed \[root@kmaster pro]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 10d web1 ClusterIP 10.99.64.14 80/TCP 4s

创建一个svc，ip为10.99.64.14 ，在某个节点上测试访问。（正常情况下会随机转发到到所有的pod上面，看到对应的host内容） \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host03 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host01 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host03

\#假如，我们去host01上，把 index.html 删除掉。请问会影响 host.html 的访问吗？ 明明没有访问index.html，你觉得会不会对host.html有所影响？

\[root@kmaster pro]# kubectl exec -ti web1-68945889d4-6dd5l -- bash root@web1-68945889d4-6dd5l:/# rm -rf /usr/share/nginx/html/ 50x.html host.html index.html\
root@web1-68945889d4-6dd5l:/# rm -rf /usr/share/nginx/html/index.html

\#再次测试访问，会发现，不再转发到host01上了。为什么？

\[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host03 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host03 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host02 \[root@knode1 \~]# curl -s 10.99.64.14/host.html host03

因为，yaml脚本里面定义的探测，是探测 index.html，既然该文件不存在了，那么说明有问题了。但定义的是readiness probe，它并不会重启pod，所以不会自动修复。

apiVersion: v1 kind: Pod metadata: labels: test: liveness name: pod2 spec: containers:

*   name: pod2 image: nginx imagePullPolicy: IfNotPresent livenessProbe: httpGet: path: /index.html port: 80 initialDelaySeconds: 3 periodSeconds: 3

    startupProbe: httpGet: path: /healthz port: liveness-port failureThreshold: 30 periodSeconds: 10

livenessprobe和readinessprobe 这两种探针是在pod【启动后-运行中】生效检测的。 startupprobe启动探针1.16版本后出现的，是在pod【启动中/前】生效检测的，必须等待内部的容器全部启动完毕后，startup功能就完成了，不会二次执行了。

livenessprobe 存活periodSeconds: 3 每隔3秒检测一次，一旦检测不成功，会重启。 假如定义了一个POD,这个pod里面要运行一个应用程序，这个程序有点大，例如程序启动过程需要60秒。 这时候就会进入到无限循环中。

初始化和依赖项准备：有时，容器在启动后可能需要一些额外的时间来完成初始化过程。这可能涉及数据库连接、加载配置文件或进行其他初始化任务。在这种情况下，使用 startupProbe 可以确保容器在完全初始化完成之前不会接收流量。同时，您还可以定义依赖项，如数据库或其他服务是否已准备就绪，使得容器能够正确运行。

避免不必要的重启：在容器启动的早期阶段，应用程序可能尚未完全启动，或者依赖项尚未就绪。如果此时将流量发送到容器，可能会导致应用程序出现故障或功能不完整。通过使用 startupProbe，可以确保在容器完全启动并准备好接收流量之前，不会将流量发送到容器，从而避免了不必要的重启。

自愈能力：在某些情况下，应用程序可能会在启动过程中出现故障。如果没有正确的探测机制，并及时采取适当的恢复措施，应用程序可能会持续失败或导致其他问题。使用 startupProbe 可以帮助 Kubernetes 在启动过程中自动检测并处理出现的问题，并执行相应的重启策略。

startupProbe 的意义在于确保容器在启动过程中顺利完成且准备就绪，以提供稳定的服务并避免不必要的应用程序故障。它是一种有助于提高可靠性和自愈能力的重要机制。

startup 启动探针 startup探针仅在容器启动的时候发生作用，一旦完成（单次），后续就不会再运行了。

startup -- 优先级最高 liveness readeness 三个探针可以同时存在

在 Kubernetes 中，readinessProbe、livenessProbe 和 startupProbe 是三种不同用途的探针，用于监测容器的健康状态。下面是它们的区别：

readinessProbe（就绪性探针）:

作用： 用于确定容器是否已准备好接收网络流量。 触发条件： 当 readinessProbe 返回成功（HTTP 状态码200-399）时，容器被认为已准备好，可以接收流量；否则，它被标记为未准备好，不会接收流量。 使用场景： 在应用程序需要一些启动时间来初始化或在启动后可能出现短暂的不可用状态时使用。 livenessProbe（活跃性探针）:

作用： 用于确定容器是否在运行状态。 触发条件： 当 livenessProbe 返回失败（非 HTTP 状态码200-399）时，容器被认为失败，Kubernetes 将尝试重新启动该容器。 使用场景： 在应用程序可能进入死锁或无响应状态时使用，以便及时检测并重新启动容器。 startupProbe（启动性探针）:

作用： 用于确定容器是否已完成启动和初始化。 触发条件： startupProbe 在容器启动后的初始一段时间内运行，并在此期间确定容器是否准备好接收流量。与 readinessProbe 不同，startupProbe 只在启动阶段运行一次。 使用场景： 在应用程序启动时可能需要一些时间来初始化或完成一些特定任务时使用。它不会阻止流量，但可以确保应用程序在接收流量之前已经初始化。
