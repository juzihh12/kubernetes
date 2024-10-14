# 06 deployment

### deployment控制器

1.deployment及副本数 2.副本数修改方法 3.动态扩展HPA 4.镜像滚动升级及回滚 5.其它控制器

### 1.deployment及副本数

在k8s里面，最小的调度单位是pod，但是pod本身不稳定，导致系统不健壮，没有可再生性（自愈功能）。 在集群中，业务需要成百上千甚至成千上万个pod，而对于这些pod的管理，k8s提供了很多控制器来管理他们。其中一个就叫deploment。 查看下所有的简称 \[root@kmaster \~]# kubectl api-resources |grep depl deployments deploy apps/v1 true Deployment

集群中只需要告诉deploy，需要多少个pod即可，一旦某个pod宕掉，deploy会生成新的pod，保证集群中的一定存在3个pod。少一个，生成一个，多一个，删除一个。

yaml文件声明式文本。你直接告诉他干什么就完了。不需要想命令式文本一样，每一步都要告诉它怎么做。

deploy可通过yaml或命令行来创建。 1.17及之前 kubectl run 名称，这个命令默认创建的是deploy 1.17之后，kubectl run 命令，这个命令默认创建的是pod

1.17之后版本，创建deploy要使用 kubectl create deploy ，这个命令是创建deploy

\[root@kmaster \~]# kubectl create deployment dep1 --image nginx --dry-run=client -o yaml > dep1.yaml

apiVersion: apps/v1 kind: Deployment --指定的类型 metadata: --这个dep的属性信息 creationTimestamp: null labels: app: dep1 name: dep1 spec: replicas: 1 --副本数 selector: matchLabels: --这里的matchlabels 必须匹配下面labels里面的标签（下面的标签可以有多个，但至少匹配一个），如果不匹配，deploy无法管理，会直接报错。 app: dep1 strategy: {} template: --所有的副本通过这个模板创建出来的 metadata: creationTimestamp: null labels: --创建出来pod的标签信息 app: dep1 spec: containers: - image: nginx name: nginx resources: {} status: {}

\[root@kmaster \~]# vim dep1.yaml \[root@kmaster \~]# cat dep1.yaml apiVersion: apps/v1 kind: Deployment metadata: creationTimestamp: null labels: app: dep1 name: dep1 spec: replicas: 3 selector: matchLabels: app: dep1 strategy: {} template: metadata: creationTimestamp: null labels: app: dep1 abb: dep2 spec: containers: - image: nginx name: nginx imagePullPolicy: IfNotPresent resources: {} status: {} \[root@kmaster \~]# kubectl apply -f dep1.yaml deployment.apps/dep1 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-9q9hl 1/1 Running 0 6s dep1-b848559fc-h4c87 1/1 Running 0 6s dep1-b848559fc-zz9fq 1/1 Running 0 6s

查看pod标签 \[root@kmaster \~]# kubectl get pod --show-labels NAME READY STATUS RESTARTS AGE LABELS dep1-b848559fc-9q9hl 1/1 Running 0 60s abb=dep2,app=dep1,pod-template-hash=b848559fc dep1-b848559fc-h4c87 1/1 Running 0 60s abb=dep2,app=dep1,pod-template-hash=b848559fc dep1-b848559fc-zz9fq 1/1 Running 0 60s abb=dep2,app=dep1,pod-template-hash=b848559fc 使用的都是一个deploy创建出来的，标签都一致。

查看deploy \[root@kmaster \~]# kubectl get deployments.apps NAME READY UP-TO-DATE AVAILABLE AGE dep1 3/3 3 3 117s \[root@kmaster \~]# kubectl get deployments.apps -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 3/3 3 3 2m28s nginx nginx app=dep1

selector指定的是当前deploy是通过app=dep1这个标签来管理和控制pod的。会监控这些pod。 （deploy本身的标签不受到 matchLabels 的影响，但是 matchLabels 标签一定要和下面的 template 模板里面的标签一致） 删除一个，就是启动一个，如果不把deploy删除，那么pod是永远删除不掉的。比如： \[root@kmaster \~]# kubectl delete pods dep1-b848559fc-9q9hl pod "dep1-b848559fc-9q9hl" deleted

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-h4c87 1/1 Running 0 5m1s dep1-b848559fc-njwb4 1/1 Running 0 10s dep1-b848559fc-zz9fq 1/1 Running 0 5m1s

删除所有的pod \[root@kmaster \~]# kubectl delete pods dep1-b848559fc-{h4c87,njwb4,zz9fq} pod "dep1-b848559fc-h4c87" deleted pod "dep1-b848559fc-njwb4" deleted pod "dep1-b848559fc-zz9fq" deleted

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 3s dep1-b848559fc-cntvn 1/1 Running 0 3s dep1-b848559fc-j7sjd 1/1 Running 0 3s

查看deploy详细信息 \[root@kmaster \~]# kubectl describe deployments.apps dep1 Name: dep1 Namespace: sales CreationTimestamp: Thu, 09 Feb 2023 22:06:19 +0800 Labels: app=dep1

### 2.副本数修改方法

deploy知道是什么了，那么如果要修改deploy副本数规则呢？修改的方法：直接在线修改、命令行工具、yaml文件 修改副本数 1.在线修改 类似于vim编辑器的格式，类似于ubuntu \[root@kmaster \~]# kubectl edit deployments.apps dep1 deployment.apps/dep1 edited

搜索 /replicas 修改为 10

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 3m15s dep1-b848559fc-8ng7w 1/1 Running 0 4s dep1-b848559fc-b4mbn 1/1 Running 0 4s dep1-b848559fc-bbhx4 1/1 Running 0 4s dep1-b848559fc-cntvn 1/1 Running 0 3m15s dep1-b848559fc-cr6kt 1/1 Running 0 4s dep1-b848559fc-j7sjd 1/1 Running 0 3m15s dep1-b848559fc-kv6xl 1/1 Running 0 4s dep1-b848559fc-vklq5 1/1 Running 0 4s dep1-b848559fc-xt4m5 1/1 Running 0 4s

那为什么要运行这么多呢？跑nginx多副本，目的可以进行负载均衡。

2.命令行 \[root@kmaster \~]# kubectl scale deployment dep1 --replicas 5 deployment.apps/dep1 scaled \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 5m54s dep1-b848559fc-8ng7w 1/1 Running 0 2m43s dep1-b848559fc-cntvn 1/1 Running 0 5m54s dep1-b848559fc-j7sjd 1/1 Running 0 5m54s dep1-b848559fc-xt4m5 1/1 Running 0 2m43s

\[root@kmaster \~]# kubectl scale deployment dep1 --replicas 2 deployment.apps/dep1 scaled \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 6m9s dep1-b848559fc-j7sjd 1/1 Running 0 6m9s

3.yaml文件 \[root@kmaster \~]# vim dep1.yaml \[root@kmaster \~]# cat dep1.yaml |grep repli replicas: 5

\[root@kmaster \~]# kubectl apply -f dep1.yaml deployment.apps/dep1 configured \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 7m15s dep1-b848559fc-cb4z6 1/1 Running 0 3s dep1-b848559fc-gjvqj 1/1 Running 0 3s dep1-b848559fc-j7sjd 1/1 Running 0 7m15s dep1-b848559fc-rz5r4 1/1 Running 0 3s

### 3.动态扩展HPA

以上修改副本数，都是基于手工来修改的，如果面对未知的业务系统，业务并发量忽高忽低，总不能手工来来回回修改，那怎么办呢？ 是否可以根据pod的负载，让它自动调节？使用HPA，类似于公有云的弹性负载AS。 比如规定每个pod的cpu阈值为80%，这时候就可以进行扩展。 HPA （Horizontal Pod Autoscaler） 水平自动伸缩，通过检测pod cpu的负载，解决deploy里某pod负载过高，动态伸缩pod的数量来实现负载均衡。 HPA一旦监测pod负载过高，就会通知deploy，要创建更多的副本数，这样每个pod负载就会轻一些。

需要注意的是，HPA是通过metricservice组件来进行检测的，之前已经安装好了。 \[root@kmaster \~]# kubectl top pod NAME CPU(cores) MEMORY(bytes)\
dep1-b848559fc-5srxc 0m 2Mi\
dep1-b848559fc-cb4z6 0m 2Mi\
dep1-b848559fc-gjvqj 0m 2Mi\
dep1-b848559fc-j7sjd 0m 2Mi\
dep1-b848559fc-rz5r4 0m 2Mi

\[root@kmaster \~]# kubectl autoscale deployment dep1 --min 1 --max 6 horizontalpodautoscaler.autoscaling/dep1 autoscaled \[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 /80% 1 6 0 4s

接下来，我们创建超出hpa规定的副本数 \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 21m dep1-b848559fc-cb4z6 1/1 Running 0 13m dep1-b848559fc-gjvqj 1/1 Running 0 13m dep1-b848559fc-j7sjd 1/1 Running 0 21m dep1-b848559fc-rz5r4 1/1 Running 0 13m

执行手工管理 \[root@kmaster \~]# kubectl scale deployment dep1 --replicas 10 deployment.apps/dep1 scaled

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-4mrg5 1/1 Running 0 3s dep1-b848559fc-5srxc 1/1 Running 0 22m dep1-b848559fc-cb4z6 1/1 Running 0 14m dep1-b848559fc-gjvqj 1/1 Running 0 14m dep1-b848559fc-j7sjd 1/1 Running 0 22m dep1-b848559fc-jxksz 1/1 Running 0 3s dep1-b848559fc-nkxsk 1/1 Running 0 3s dep1-b848559fc-pwvs8 1/1 Running 0 3s dep1-b848559fc-rz5r4 1/1 Running 0 14m dep1-b848559fc-s8hlv 1/1 Running 0 3s

\[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 /80% 1 6 10 113s 因为hpa实时监测，所以等待一下，马上变为HPA规定的最高副本数6.不存在谁的优先级高，而是看谁修改的快，修改的频繁。

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-b848559fc-5srxc 1/1 Running 0 22m dep1-b848559fc-cb4z6 1/1 Running 0 15m dep1-b848559fc-gjvqj 1/1 Running 0 15m dep1-b848559fc-j7sjd 1/1 Running 0 22m dep1-b848559fc-jxksz 1/1 Running 0 16s dep1-b848559fc-rz5r4 1/1 Running 0 15m

\[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 /80% 1 6 6 3m57s

注意targets的值，有一个unknown，我们查看下hpa的具体信息 \[root@kmaster \~]# kubectl describe hpa dep1 Warning FailedComputeMetricsReplicas 2m32s (x12 over 5m32s) horizontal-pod-autoscaler invalid metrics (1 invalid out of 1), first error is: failed to get cpu resource metric value: failed to get cpu utilization: missing request for cpu Warning FailedGetResourceMetric 32s (x20 over 5m32s) horizontal-pod-autoscaler failed to get cpu utilization: missing request for cpu 好像里面有报错，就是因为这个原因而导致的unknown。看不到使用的cpu百分比，如果想要生效，必须启用deployment的资源请求。 修改deployment： \[root@kmaster \~]# kubectl delete hpa dep1 horizontalpodautoscaler.autoscaling "dep1" deleted

\[root@kmaster \~]# kubectl get deployments.apps NAME READY UP-TO-DATE AVAILABLE AGE dep1 6/6 6 6 41m \[root@kmaster \~]# kubectl edit deployments.apps dep1

```
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
```

***

```
spec:
  containers:
  - image: nginx
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: 
      requests:
        cpu: 500m
```

\[root@kmaster \~]# kubectl get deployments.apps dep1 NAME READY UP-TO-DATE AVAILABLE AGE dep1 6/6 6 6 43m

\[root@kmaster \~]# kubectl scale deployment dep1 --replicas 1 deployment.apps/dep1 scaled

\[root@kmaster \~]# kubectl autoscale deployment dep1 --min 1 --max 6 --cpu-percent 80 horizontalpodautoscaler.autoscaling/dep1 autoscaled

\[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 /80% 1 6 0 7s

稍等一会，unknown就会变为具体的值 \[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 0%/80% 1 6 1 16s

### 测试HPA

模拟一个pod负载很重，现在系统中只有一个副本 \[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 0%/80% 1 6 1 5m47s \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-7f7d4d9b56-9nkvk 1/1 Running 0 7m51s

\[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES dep1-7f7d4d9b56-9nkvk 1/1 Running 0 9m7s 10.244.69.245 knode2

测试-消耗cpu \[root@kmaster \~]# kubectl exec -ti pods/dep1-7f7d4d9b56-9nkvk -- bash root@dep1-7f7d4d9b56-9nkvk:/# cat /dev/zero > /dev/null & \[1] 37 root@dep1-7f7d4d9b56-9nkvk:/# cat /dev/zero > /dev/null & \[2] 38 root@dep1-7f7d4d9b56-9nkvk:/# cat /dev/zero > /dev/null & \[3] 39 root@dep1-7f7d4d9b56-9nkvk:/# cat /dev/zero > /dev/null & \[4] 40 root@dep1-7f7d4d9b56-9nkvk:/# cat /dev/zero > /dev/null & \[5] 41

等待一会，查看pod，6个pod就创建出来了。 \[root@kmaster \~]# kubectl top pod NAME CPU(cores) MEMORY(bytes)\
dep1-7f7d4d9b56-7t6q5 0m 2Mi\
dep1-7f7d4d9b56-9nkvk 1926m 4Mi\
dep1-7f7d4d9b56-f9ckv 0m 2Mi\
dep1-7f7d4d9b56-gqx8g 0m 2Mi\
dep1-7f7d4d9b56-q8l5q 0m 2Mi\
dep1-7f7d4d9b56-smplt 0m 2Mi

可是，现在有个问题，不是负载均衡吗？怎么扩展了，依然负载很重呢？因为你这个负载只是形式上的负载均衡，来自于内部产生的负载，所以无法转移，如果是外部来源，就会负载均衡。 \[root@knode2 \~]# ps -ef |grep cat root 1950 1877 0 Feb08 ? 00:00:00 runsv allocate-tunnel-addrs root 1957 1950 0 Feb08 ? 00:00:04 calico-node -allocate-tunnel-addrs root 537961 537862 38 23:05 pts/0 00:02:27 cat /dev/zero root 537987 537862 39 23:05 pts/0 00:02:31 cat /dev/zero root 537988 537862 38 23:05 pts/0 00:02:28 cat /dev/zero root 537989 537862 38 23:05 pts/0 00:02:28 cat /dev/zero root 538017 537862 39 23:05 pts/0 00:02:30 cat /dev/zero root 540196 536113 0 23:11 pts/0 00:00:00 grep --color=auto cat

\[root@knode2 \~]# kill -9 537961 537987 537988 537989 538017

当负载降下来之后，要等待5分钟左右，pod才会删除。

\[root@kmaster \~]# kubectl top pod NAME CPU(cores) MEMORY(bytes)\
dep1-7f7d4d9b56-9nkvk 0m 3Mi

### 外部访问压力测试

为deployment创建一个服务，接受外部请求，类型为NodePort。 \[root@kmaster \~]# kubectl expose --help |grep dep \[root@kmaster \~]# kubectl expose deployment dep1 --port=80 --target-port=80 --type=NodePort service/dep1 exposed \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE dep1 NodePort 10.110.150.188 80:32528/TCP 3s

我们直接访问master主机32528，就可以访问dep1的这个service了。这个service会把请求丢给pod。 \[root@kmaster \~]# yum install -y httpd-tools.x86\_64 安装ab工具，ab是apachebench命令的缩写，ab是apache自带的压力测试工具。 \[root@kmaster \~]# ab -h \[root@kmaster \~]# ab -t 600 -n 1000000 -c 1000 http://192.168.100.146:32528/index.html This is ApacheBench, Version 2.3 <$Revision: 1843412 $> Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/ Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 192.168.100.146 (be patient) Completed 100000 requests Completed 200000 requests Completed 300000 requests Completed 400000 requests Completed 500000 requests

等待，观察top情况及pod数量。 \[root@kmaster \~]# kubectl get hpa NAME REFERENCE TARGETS MINPODS MAXPODS REPLICAS AGE dep1 Deployment/dep1 70%/80% 1 6 1 61m

### 镜像升级与回滚

\[root@knode1 \~]# crictl pull nginx:1.9 Image is up to date for sha256:f568d3158b1e871b713cb33aca5a9377bc21a1f644addf41368393d28c35e894 下载老版本镜像 \[root@knode1 \~]# crictl img IMAGE TAG IMAGE ID SIZE docker.io/calico/cni v3.25.0 d70a5947d57e5 88MB docker.io/calico/node v3.25.0 08616d26b8e74 87.2MB docker.io/library/alpine latest 042a816809aac 3.37MB docker.io/library/centos latest 5d0da3dc97646 83.5MB docker.io/library/mysql latest 05b458cc32b96 153MB docker.io/library/nginx 1.9 f568d3158b1e8 71.2MB docker.io/library/nginx latest 9eee96112defa 56.9MB

1.在线修改 \[root@kmaster \~]# kubectl apply -f dep1.yaml deployment.apps/dep1 created \[root@kmaster \~]# kubectl get deployments.apps NAME READY UP-TO-DATE AVAILABLE AGE dep1 5/5 5 5 3s \[root@kmaster \~]# kubectl get deployments.apps -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 16s nginx nginx app=dep1

\[root@kmaster \~]# kubectl edit deployments.apps dep1 spec: containers: - image: nginx:1.9

等待一会 \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES dep1-c6d6b65d5-68jxh 1/1 Running 0 55s 10.244.69.253 knode2 dep1-c6d6b65d5-brlcn 1/1 Running 0 54s 10.244.69.255 knode2 dep1-c6d6b65d5-ml2xh 1/1 Running 0 54s 10.244.69.254 knode2 dep1-c6d6b65d5-q5nzg 1/1 Running 0 55s 10.244.195.138 knode1 dep1-c6d6b65d5-v9r2n 1/1 Running 0 55s 10.244.195.137 knode1 \[root@kmaster \~]# kubectl get deployments.apps -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 4m16s nginx nginx:1.9 app=dep1

当修改deployment的时候，本质上是删除旧的pod，重新创建新的pod，这是dep本身的特性。

2.yaml文件 直接修改yaml文件，修改为latest版本 \[root@kmaster \~]# kubectl apply -f dep1.yaml deployment.apps/dep1 configured \[root@kmaster \~]# kubectl get pod -o wide

\[root@kmaster \~]# kubectl get deployments.apps -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 8m2s nginx nginx:latest app=dep1

3.命令行 \[root@kmaster \~]# kubectl set image deploy dep1 nginx=nginx:1.9 \[root@kmaster \~]# kubectl get deployments.apps dep1 -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 10m nginx nginx:1.9 app=dep1

\[root@kmaster \~]# kubectl set image deploy dep1 nginx=nginx:latest \[root@kmaster \~]# kubectl get deployments.apps dep1 -o wide \[root@kmaster \~]# kubectl get pod -o wide

这时候有个问题，好像我们所有操作都是无法记录的。比如想查看我们升级或回滚操作记录。 \[root@kmaster \~]# kubectl rollout history deployment dep1 deployment.apps/dep1 REVISION CHANGE-CAUSE 1 2 3

\[root@kmaster \~]# kubectl set image deploy dep1 nginx=nginx:latest --record=true Flag --record has been deprecated, --record will be removed in the future deployment.apps/dep1 image updated \[root@kmaster \~]# kubectl rollout history deployment dep1 deployment.apps/dep1 REVISION CHANGE-CAUSE 1 4 5 kubectl set image deploy dep1 nginx=nginx:latest --record=true

如果发现某个镜像存在缺陷，可以通过上述方法进行更换，也可以进行撤销操作。 \[root@kmaster \~]# kubectl rollout undo deployment dep1 deployment.apps/dep1 rolled back \[root@kmaster \~]# kubectl get deployments.apps dep1 -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 20m nginx nginx:1.9 app=dep1

撤销指定版本 \[root@kmaster \~]# kubectl rollout history deployment dep1 deployment.apps/dep1 REVISION CHANGE-CAUSE 1 5 kubectl set image deploy dep1 nginx=nginx:latest --record=true 6

\[root@kmaster \~]# kubectl rollout undo deployment dep1 --to-revision=5 deployment.apps/dep1 rolled back \[root@kmaster \~]# kubectl get deployments.apps dep1 -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 22m nginx nginx:latest app=dep1

现在有5个pod，在升级的时候，是不是一次性把5个删除，然后一起创建5个pod？如果全部升级，pod无法对外提供服务。 现在想删除一个，升级一个，或者删除2个，升级2个，进行滚动升级。 滚动升级有两个重要的参数：maxSurge 一次升级多少 maxUnavailable 一次性删除几个 \[root@kmaster \~]# kubectl edit deployments.apps dep1 默认这个参数没有值，会采用默认值 strategy: rollingUpdate: maxSurge: 25% maxUnavailable: 25%

我们把它改成数量，1 strategy: rollingUpdate: maxSurge: 1 maxUnavailable: 1

\[root@kmaster \~]# kubectl get deployments.apps dep1 -o wide NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR dep1 5/5 5 5 30m nginx nginx:latest app=dep1

\[root@kmaster \~]# kubectl set image deploy dep1 nginx=nginx:1.9 --record=true Flag --record has been deprecated, --record will be removed in the future deployment.apps/dep1 image updated

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-96cf98598-b2bpl 1/1 Running 0 7m53s dep1-96cf98598-cpdkg 1/1 Terminating 0 7m53s dep1-96cf98598-g2sdd 1/1 Running 0 7m53s dep1-96cf98598-nrg9d 1/1 Terminating 0 7m52s dep1-c6d6b65d5-4jfzj 1/1 Running 0 1s dep1-c6d6b65d5-5nhxt 1/1 Running 0 1s dep1-c6d6b65d5-ntkq9 0/1 ContainerCreating 0 0s dep1-c6d6b65d5-tszvw 0/1 ContainerCreating 0 0s \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE dep1-c6d6b65d5-4jfzj 1/1 Running 0 15s dep1-c6d6b65d5-5nhxt 1/1 Running 0 15s dep1-c6d6b65d5-6rc9b 1/1 Running 0 13s dep1-c6d6b65d5-ntkq9 1/1 Running 0 14s dep1-c6d6b65d5-tszvw 1/1 Running 0 14s

### 5.其它控制器

daemonset也是一种控制器，也是用来创建pod的，但是和dep不一样，dep需要指定副本数，每个worker上都可以运行多个副本。 ds不需要指定副本数，会自动的在每个worker上都创建1个副本，不可运行多个。这东西有啥用？

作用就是在每个节点上收集日志、监控和管理等，还记得drain操作吗？里面包含驱逐操作，这个pod是不能删除的。 应用场景 网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络。 存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录，比如：glusterd、ceph。 监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集，比如：fluentd、logstash、Prometheus 等。

\[root@kmaster \~]# kubectl get ds No resources found in sales namespace. \[root@kmaster \~]# kubectl get ds -n kube-system NAME DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE SELECTOR AGE calico-node 3 3 3 3 3 kubernetes.io/os=linux 3d13h kube-proxy 3 3 3 3 3 kubernetes.io/os=linux 3d14h

创建ds \[root@kmaster \~]# kubectl create deployment ds1 --image busybox --dry-run=client -o yaml -- sh -c "sleep 36000" > ds1.yaml \[root@kmaster \~]# vim ds1.yaml \[root@kmaster \~]# cat ds1.yaml apiVersion: apps/v1 kind: DaemonSet metadata: creationTimestamp: null labels: app: ds1 name: ds1 spec: selector: matchLabels: app: ds1 template: metadata: creationTimestamp: null labels: app: ds1 spec: containers: - command: - sh - -c - sleep 36000 image: busybox name: busybox resources: {}

1.更改kind 2.删除副本 3.删除strategy 4.删除status

\[root@kmaster \~]# kubectl get ds No resources found in sales namespace. \[root@kmaster \~]# kubectl apply -f ds1.yaml daemonset.apps/ds1 created \[root@kmaster \~]# kubectl get ds NAME DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE SELECTOR AGE ds1 2 2 0 2 0 3s \[root@kmaster \~]# kubectl get pod -o wide

控制节点有吗？不是每个节点上都要有吗？没有，因为master上有污点。默认不会创建。 \[root@kmaster \~]# kubectl describe nodes kmaster |grep -i tai Annotations: kubeadm.alpha.kubernetes.io/cri-socket: unix:///var/run/containerd/containerd.sock Taints: node-role.kubernetes.io/control-plane:NoSchedule Container Runtime Version: containerd://1.6.16

\[root@kmaster \~]# kubectl delete ds ds1 --这里删除的是app名称 daemonset.apps "ds1" deleted \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES ds1-48gcx 1/1 Terminating 0 4m7s 10.244.195.154 knode1 ds1-psxmv 1/1 Terminating 0 4m7s 10.244.69.213 knode2

指定pod所在位置 \[root@kmaster \~]# kubectl get nodes --show-labels NAME STATUS ROLES AGE VERSION LABELS kmaster Ready control-plane 3d14h v1.25.4 beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kmaster,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers= knode1 Ready 3d14h v1.25.4 beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disktype=ssd,

\[root@kmaster \~]# vim ds1.yaml.bak

```
spec:
  nodeSelector:
    disktype: ssd
  containers:
  - command:
    - sh
    - -c
    - sleep 36000
```

\[root@kmaster \~]# kubectl apply -f ds1.yaml daemonset.apps/ds1 created \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES ds1-nlfgb 1/1 Running 0 5s 10.244.195.155 knode1
