# 10 Service SVC

### svc

1.svc是什么 2.创建svc 3.服务的发现：ClusterIP/变量/DNS方式（建议） 4.服务的发布：NodePort/LoadBalance/Ingress

k8s里面的最小调度单位是pod，pod里面包含的有容器，pod是最终对外提供服务的。

\[root@kmaster svc]# kubectl run pod1 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml > pod1.yaml \[root@kmaster svc]# ls pod1.yaml \[root@kmaster svc]# kubectl apply -f pod1.yaml pod/pod1 created \[root@kmaster svc]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 4s \[root@kmaster svc]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod1 1/1 Running 0 9s 10.244.195.172 knode1

pod的ip，只有集群内部可见，也就是说只有 master和node可以访问，其他pod也是可以访问的。 \[root@kmaster svc]# ping 10.244.195.172 PING 10.244.195.172 (10.244.195.172) 56(84) bytes of data. 64 bytes from 10.244.195.172: icmp\_seq=1 ttl=63 time=0.839 ms 64 bytes from 10.244.195.172: icmp\_seq=2 ttl=63 time=0.238 ms

为何可以内部互通？因为我们配置了calico网络，会建立起很多iptables及转发的规则。 但是外接的主机是无法连通这个pod的。 比如用物理windows主机。打开一个浏览器，是无法访问到pod里面的内容的。

之前讲过，如果想让外界访问，我们可以做端口映射 \[root@kmaster svc]# kubectl run pod3 --image nginx --image-pull-policy IfNotPresent --port 80 --dry-run=client -o yaml > pod3.yaml \[root@kmaster svc]# vim pod2.yaml \[root@kmaster svc]# cat pod2.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod1 name: pod1 spec: containers:

* image: nginx imagePullPolicy: IfNotPresent name: pod1 ports:
  * containerPort: 80 hostPort: 5000 resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {} \[root@kmaster svc]# kubectl apply -f pod2.yaml pod/pod1 created \[root@kmaster svc]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod1 1/1 Running 0 4s 10.244.195.173 knode1

查看该pod是在node1上运行的，所以通过windows访问node1的ip地址加端口号 http://192.168.146.140:5000/ 即可访问到nginx服务。 为啥可以访问到？因为添加的hostPort 会修改iptables规则。

\[root@knode1 \~]# iptables -S -t nat |grep '5500 -j' -A CNI-HOSTPORT-DNAT -p tcp -m comment --comment "dnat name: "k8s-pod-network" id: "dcc3aca1ddc72f3ea388f850681fb27769afbe5c005d63d5ff6799fa6c72c8bc"" -m multiport --dports 5000 -j CNI-DN-d1a0abedc4aa34360e04b -A CNI-DN-d1a0abedc4aa34360e04b -s 10.244.195.173/32 -p tcp -m tcp --dport 5000 -j CNI-HOSTPORT-SETMARK -A CNI-DN-d1a0abedc4aa34360e04b -s 127.0.0.1/32 -p tcp -m tcp --dport 5000 -j CNI-HOSTPORT-SETMARK -A CNI-DN-d1a0abedc4aa34360e04b -p tcp -m tcp --dport 5000 -j DNAT --to-destination 10.244.195.173:80

但是，这种方式比较麻烦，而且存在一个问题。比如：创建一个deployment https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

复制代码 \[root@kmaster svc]# vim pod3.yaml \[root@kmaster svc]# cat pod3.yaml --注意replicas 是3 apiVersion: apps/v1 kind: Deployment metadata: name: nginx-deployment labels: app: nginx spec: replicas: 3 selector: matchLabels: app: nginx template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx imagePullPolicy: IfNotPresent ports: - containerPort: 80 hostPort: 5500

这样写，会不会有问题呢？来执行下 \[root@kmaster svc]# kubectl apply -f pod3.yaml deployment.apps/nginx-deployment created \[root@kmaster svc]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES nginx-deployment-8985c899b-cbv8w 1/1 Running 0 6s 10.244.69.204 knode2 nginx-deployment-8985c899b-dnrnb 1/1 Running 0 6s 10.244.195.174 knode1 nginx-deployment-8985c899b-zltb6 0/1 Pending 0 6s

有没有发现，有一个出现了pending？其他两个是好的。 因为node1和node2端口都被占用了，第三台无法进行映射。 \[root@kmaster svc]# kubectl describe pods/nginx-deployment-8985c899b-zltb6 ... 2 node(s) didn't have free ports for the requested pod ports...

那我直接改成两个副本不就得了？一个pod映射一个端口。 \[root@kmaster svc]# vim pod3.yaml \[root@kmaster svc]# cat pod3.yaml apiVersion: apps/v1 kind: Deployment metadata: name: nginx-deployment labels: app: nginx spec: replicas: 2 selector: matchLabels: app: nginx template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx imagePullPolicy: IfNotPresent ports: - containerPort: 80 hostPort: 5500 \[root@kmaster svc]# kubectl apply -f pod3.yaml deployment.apps/nginx-deployment created \[root@kmaster svc]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES nginx-deployment-8985c899b-sgw8w 1/1 Running 0 9s 10.244.195.175 knode1 nginx-deployment-8985c899b-zfg6n 1/1 Running 0 9s 10.244.69.205 knode2

但是这时候，只有2个节点，是无法使用负载均衡的。因为你连接要么连接第一台，要么连接第二台。

那如何解决hostport以及负载均衡的问题呢？

```
   svc 负载均衡器
```

pod1 pod2 pod3

deloyment 也是通过标签来管理pod svc会通过标签会找到对应的pod，svc是负载均衡器，后端的pod叫real server（endpoint） 对于pod，是通过deploy来管理的，deploy创建的pod，它的pod标签都是一样的。 一旦某个pod宕掉，deploy会立刻创建一个新的pod，这时候因为标签和之前一样，所以svc会自动识别到。 客户端请求到了svc，svc将请求转发给（pod所在物理主机上的kube-proxy）后端的pod上，那么是由谁来进行转发的呢？是由kube-proxy（两种模式：iptables和ipvs 前者默认，后者性能好）

修改脚本 \[root@kmaster svc]# kubectl delete -f pod3.yaml deployment.apps "nginx-deployment" deleted \[root@kmaster svc]# vim pod3.yaml \[root@kmaster svc]# cat pod3.yaml apiVersion: apps/v1 kind: Deployment metadata: name: nginx-deployment labels: app: nginx spec: replicas: 3 selector: matchLabels: app: nginx template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx imagePullPolicy: IfNotPresent

## 执行

\[root@kmaster svc]# kubectl apply -f pod3.yaml deployment.apps/nginx-deployment created \[root@kmaster svc]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES nginx-deployment-57cc89bc77-4tdwj 1/1 Running 0 37s 10.244.195.176 knode1 nginx-deployment-57cc89bc77-czqx4 1/1 Running 0 37s 10.244.195.177 knode1 nginx-deployment-57cc89bc77-d7zgw 1/1 Running 0 37s 10.244.69.206 knode2

\#修改里面的index.html内容 \[root@kmaster svc]# kubectl exec -ti nginx-deployment-57cc89bc77-4tdwj -- bash root@nginx-deployment-57cc89bc77-4tdwj:/# echo 111 > /usr/share/nginx/html/index.html root@nginx-deployment-57cc89bc77-4tdwj:/# exit exit \[root@kmaster svc]# kubectl exec -ti nginx-deployment-57cc89bc77-czqx4 -- bash root@nginx-deployment-57cc89bc77-czqx4:/# echo 222 > /usr/share/nginx/html/index.html root@nginx-deployment-57cc89bc77-czqx4:/# exit exit \[root@kmaster svc]# kubectl exec -ti nginx-deployment-57cc89bc77-d7zgw -- bash root@nginx-deployment-57cc89bc77-d7zgw:/# echo 333 > /usr/share/nginx/html/index.html root@nginx-deployment-57cc89bc77-d7zgw:/# exit exit

\#创建svc

如果没有指定svc名字，默认使用deployment名字 port指定的是svc本身的端口（自定义），svc本身不提供业务服务，只是负载均衡器，而targetport 指的是后端pod本身开放的端口。

\[root@kmaster svc]# kubectl expose --name svc1 deployment nginx-deployment --port 5500 --target-port 80 --dry-run=client -o yaml > svc1.yaml \[root@kmaster svc]# cat svc1.yaml apiVersion: v1 kind: Service metadata: creationTimestamp: null labels: app: nginx name: svc1 spec: ports:

* port: 5500 protocol: TCP targetPort: 80 selector: app: nginx status: loadBalancer: {}

\[root@kmaster svc]# kubectl apply -f svc1.yaml service/svc1 created \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 10d svc1 ClusterIP 10.102.158.48 5500/TCP 3s web1 ClusterIP 10.99.64.14 80/TCP 23h

\#在某个node上执行访问测试 \[root@knode1 \~]# curl -s 10.102.158.48:5500 222 \[root@knode1 \~]# curl -s 10.102.158.48:5500 333 \[root@knode1 \~]# curl -s 10.102.158.48:5500 333 \[root@knode1 \~]# curl -s 10.102.158.48:5500 222 \[root@knode1 \~]# curl -s 10.102.158.48:5500 222 \[root@knode1 \~]# curl -s 10.102.158.48:5500 333

请问这里的selector是什么？ \[root@kmaster svc]# kubectl get svc -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR kubernetes ClusterIP 10.96.0.1 443/TCP 10d svc1 ClusterIP 10.102.158.48 5500/TCP 97s app=nginx web1 ClusterIP 10.99.64.14 80/TCP 23h app=web1

svc1 对应的 selector app=nginx ，这个标签是为了选择pod的。

\[root@kmaster svc]# kubectl get svc -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR kubernetes ClusterIP 10.96.0.1 443/TCP 10d svc1 ClusterIP 10.102.158.48 5500/TCP 11m app=nginx web1 ClusterIP 10.99.64.14 80/TCP 23h app=web1

\[root@kmaster svc]# kubectl get svc --show-labels NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE LABELS kubernetes ClusterIP 10.96.0.1 443/TCP 10d component=apiserver,provider=kubernetes svc1 ClusterIP 10.102.158.48 5500/TCP 9m40s app=nginx web1 ClusterIP 10.99.64.14 80/TCP 23h app=web1

\[root@kmaster svc]# kubectl get pod -l app=nginx 这个标签是由svc的selector来决定的，而不是svc的lable决定的。 NAME READY STATUS RESTARTS AGE nginx-deployment-57cc89bc77-4tdwj 1/1 Running 0 21m nginx-deployment-57cc89bc77-czqx4 1/1 Running 0 21m nginx-deployment-57cc89bc77-d7zgw 1/1 Running 0 21m

\[root@kmaster svc]# kubectl describe svc svc1 Name: svc1 Namespace: default Labels: app=nginx Annotations: Selector: app=nginx Type: ClusterIP IP Family Policy: SingleStack IP Families: IPv4 IP: 10.102.158.48 IPs: 10.102.158.48 Port: 5500/TCP TargetPort: 80/TCP Endpoints: 10.244.195.176:80,10.244.195.177:80,10.244.69.206:80 Session Affinity: None Events:

## 找出svc1关联的所有pod

\[root@kmaster svc]# kubectl get svc -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR kubernetes ClusterIP 10.96.0.1 443/TCP 10d svc1 ClusterIP 10.102.158.48 5500/TCP 15m app=nginx web1 ClusterIP 10.99.64.14 80/TCP 23h app=web1 \[root@kmaster svc]# kubectl get pod -l app=nginx NAME READY STATUS RESTARTS AGE nginx-deployment-57cc89bc77-4tdwj 1/1 Running 0 26m nginx-deployment-57cc89bc77-czqx4 1/1 Running 0 26m nginx-deployment-57cc89bc77-d7zgw 1/1 Running 0 26m

## 之前都是通过deployment来做的，svc可以自动定位到。如果手工创建呢？

svc是纯粹根据label标签来进行定位的。

首先查看当前的svc

\[root@kmaster svc]# kubectl describe svc kubernetes svc1 web1\
\[root@kmaster svc]# kubectl describe svc svc1 Name: svc1 Namespace: default Labels: app=nginx Annotations: Selector: app=nginx Type: ClusterIP IP Family Policy: SingleStack IP Families: IPv4 IP: 10.102.158.48 IPs: 10.102.158.48 Port: 5500/TCP TargetPort: 80/TCP Endpoints: 10.244.195.176:80,10.244.195.177:80,10.244.69.206:80 Session Affinity: None Events:

手工创建pod

\[root@kmaster svc]# cp pod1.yaml pod4.yaml \[root@kmaster svc]# vim pod4.yaml \[root@kmaster svc]# cat pod3.yaml apiVersion: apps/v1 kind: Deployment metadata: name: nginx labels: app: nginx spec: replicas: 3 selector: matchLabels: app: nginx template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx imagePullPolicy: IfNotPresent \[root@kmaster svc]# kubectl apply -f pod4.yaml pod/pod1 created

查看标签对应的pod \[root@kmaster svc]# kubectl get pod -l app=nginx NAME READY STATUS RESTARTS AGE nginx-deployment-57cc89bc77-4tdwj 1/1 Running 0 54m nginx-deployment-57cc89bc77-czqx4 1/1 Running 0 54m nginx-deployment-57cc89bc77-d7zgw 1/1 Running 1 (14m ago) 54m podtest 1/1 Running 0 14m

那么会在endpoint列表里面吗？ \[root@kmaster svc]# kubectl describe svc svc1 Name: svc1 Namespace: default Labels: app=nginx Annotations: Selector: app=nginx Type: ClusterIP IP Family Policy: SingleStack IP Families: IPv4 IP: 10.102.158.48 IPs: 10.102.158.48 Port: 5500/TCP TargetPort: 80/TCP Endpoints: 10.244.195.176:80,10.244.195.177:80,10.244.69.208:80 + 1 more... Session Affinity: None

如果看不全，可以通过这个命令 \[root@kmaster \~]# kubectl get endpoints NAME ENDPOINTS AGE kubernetes 192.168.146.139:6443 11d svc1 10.244.195.176:80,10.244.195.177:80,10.244.69.208:80 + 1 more... 12h web1 35h \[root@kmaster \~]# kubectl get ep NAME ENDPOINTS AGE kubernetes 192.168.146.139:6443 11d svc1 10.244.195.176:80,10.244.195.177:80,10.244.69.208:80 + 1 more... 12h web1 35h \[root@kmaster \~]# kubectl edit ep svc1

注意： 1.svc的IP也是集群内部可访问的。 2.svc的IP地址不会发生改变，除非删除重新创建。 3.svc是通过标签定位到后端的pod的。

所以，不需要担心pod的IP地址发生改变，因为外部访问，并不是直接访问pod的，而是访问svc的IP。 svc只开放了tcp的5500，并没有开放icmp。所以不要ping。 但是可以通过wget来测试。 \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 11d svc1 ClusterIP 10.102.158.48 5500/TCP 11h web1 ClusterIP 10.99.64.14 80/TCP 35h

\[root@kmaster \~]# \[root@kmaster \~]# wget 10.102.158.48:5500 --2023-03-08 09:00:24-- http://10.102.158.48:5500/ Connecting to 10.102.158.48:5500... connected. HTTP request sent, awaiting response... 200 OK Length: 4 \[text/html] Saving to: ‘index.html’

index.html 100%\[======================================================================>] 4 --.-KB/s in 0s

2023-03-08 09:00:24 (742 KB/s) - ‘index.html’ saved \[4/4]

或者通过测试页访问 \[root@kmaster \~]# curl 10.102.158.48:5500

Welcome to nginx!

## Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to [nginx.org](http://nginx.org/).\
Commercial support is available at [nginx.com](http://nginx.com/).

_Thank you for using nginx._

或者创建一个临时容器来测试 \[root@kmaster \~]# kubectl run nginx --image nginx --image-pull-policy IfNotPresent --rm -ti -- bash If you don't see a command prompt, try pressing enter. root@nginx:/# wget 10.102.158.48:5500 bash: wget: command not found

root@nginx:/# curl 10.102.158.48:5500

Welcome to nginx!

## Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to [nginx.org](http://nginx.org/).\
Commercial support is available at [nginx.com](http://nginx.com/).

_Thank you for using nginx._

### 服务的发现

svc的IP地址只是集群内部可见的，master或node，或者集群里面的pod。

服务的发现说什么？看下图

### 客户端访问 | | -------------- 一台主机 wordpress svc| | | wordpress pod| | |--> wordpress pod 如何发现 mysql svc 的IP地址 mysql svc | | | mysql pod |

服务的发现有3种方式： 1.ClusterIP 2.变量方式 3.DNS方式（建议 一般是给无头服务去使用）

### 1.ClusterIP

\[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 11d svc1 ClusterIP 10.102.158.48 5500/TCP 13h web1 ClusterIP 10.99.64.14 80/TCP 37h

创建svc，每个svc都会有自己的clusterIP 首先查看clusterIP，然后在pod里面通过该IP访问即可。上面说过。

实验： 分别下载wordpress和mysql镜像 crictl pull mysql crictl pull wordpress

01.创建mysql Pod \[root@kmaster \~]# kubectl run mysql --image mysql --env="MYSQL\_ROOT\_PASSWORD=redhat" --env="MYSQL\_DATABASE=wordpress" --dry-run=client -o yaml > db.yaml \[root@kmaster \~]# vim db.yaml \[root@kmaster \~]# cat db.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: mysql name: mysql spec: containers:

* env:
  * name: MYSQL\_ROOT\_PASSWORD value: redhat
  * name: MYSQL\_DATABASE value: wordpress image: mysql imagePullPolicy: IfNotPresent name: mysql resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {} \[root@kmaster \~]# kubectl apply -f db.yaml pod/mysql created

02.创建mysql svc

关联mysql Pod

\[root@kmaster \~]# kubectl expose pod mysql --name db --port 3306 --target-port 3306 service/db exposed \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE db ClusterIP 10.101.35.49 3306/TCP 3s kubernetes ClusterIP 10.96.0.1 443/TCP 11d svc1 ClusterIP 10.102.158.48 5500/TCP 14h svc2 ClusterIP 10.96.72.69 5500/TCP 13m web1 ClusterIP 10.99.64.14 80/TCP 37h \[root@kmaster \~]# kubectl get svc -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR db ClusterIP 10.101.35.49 3306/TCP 16s run=mysql kubernetes ClusterIP 10.96.0.1 443/TCP 11d svc1 ClusterIP 10.102.158.48 5500/TCP 14h app=nginx svc2 ClusterIP 10.96.72.69 5500/TCP 13m app=nginx web1 ClusterIP 10.99.64.14 80/TCP 37h app=web1

0.3 创建wordpress Pod \[root@kmaster \~]# kubectl run blog --image wordpress --env="WORDPRESS\_DB\_HOST=10.109.108.210" --env="WORDPRESS\_DB\_USER=root" --env="WORDPRESS\_DB\_PASSWORD=redhat" --env="WORDPRESS\_DB\_NAME=wordpress" --dry-run=client -o yaml > blog.yaml

## 通过指定ClusterIP方式

\[root@kmaster \~]# vim blog.yaml \[root@kmaster \~]# cat blog.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: blog name: blog spec: containers:

* env:
  * name: WORDPRESS\_DB\_HOST value: 10.101.35.49
  * name: WORDPRESS\_DB\_USER value: root
  * name: WORDPRESS\_DB\_PASSWORD value: redhat
  * name: WORDPRESS\_DB\_NAME value: wordpress image: wordpress name: blog resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {}

## .,19s/MYSQL/WORDPRESS/g

## 当前行开始到19行，替换

\[root@kmaster \~]# kubectl apply -f blog.yaml pod/blog created

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE blog 1/1 Running 0 5s mysql 1/1 Running 0 15m

04.创建wordpress svc \[root@kmaster \~]# kubectl expose pod blog --name blog --port 80 --target-port 80 --type NodePort service/blog exposed \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE blog NodePort 10.108.58.66 80:30604/TCP 7s

\#NodePort后面讲 打开浏览器，输入集群内任意一个主机名，并带上端口 30604 进行访问 http://192.168.146.140:30604/ 就可以看到安装界面。并且点击下一步，直接跳过了数据库配置。 这种方式，称之为叫ClusterIP方式。

### 2.变量方式

看好时间轴及先后顺序，如下

```
 pod1      pod2       pod3
```

svc1 svc2 svc3 --------------------------------->新时间代表未来

当我们每创建一个pod，此pod里面会自动的创建一些变量的。包含了之前创建过的svc的相关变量。 pod1创建后，里面包含svc1的相关变量，没有svc2和svc3 pod2创建后，里面包含svc2的相关变量，没有svc3

哪些变量呢？比如我们创建一个临时pod \[root@kmaster \~]# kubectl run nginx --image nginx --image-pull-policy Never --rm -ti -- bash If you don't see a command prompt, try pressing enter. root@nginx:/# env |grep SVC1

SVC1\_SERVICE\_HOST=10.102.158.48 SVC1\_PORT\_5500\_TCP\_ADDR=10.102.158.48 SVC1\_PORT=tcp://10.102.158.48:5500 SVC1\_PORT\_5500\_TCP\_PROTO=tcp SVC1\_PORT\_5500\_TCP\_PORT=5500 SVC1\_SERVICE\_PORT=5500 SVC1\_PORT\_5500\_TCP=tcp://10.102.158.48:5500

default: svc1 pod hehe: svc1 meme: svc1

包含了SVC1的一些变量。 如果现在再次创建第二个SVC2呢？

\[root@kmaster \~]# kubectl get svc svc1 -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR svc1 ClusterIP 10.102.158.48 5500/TCP 14h app=nginx

\[root@kmaster \~]# kubectl expose deployment nginx-deployment --name svc2 --port 5500 --target-port 80 --selector app=nginx service/svc2 exposed

\[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 11d svc1 ClusterIP 10.102.158.48 5500/TCP 14h svc2 ClusterIP 10.96.72.69 5500/TCP 22s web1 ClusterIP 10.99.64.14 80/TCP 37h

之前的临时pod会包含SVC2的变量吗？看下 root@nginx:/# env |grep SVC2 root@nginx:/# 没有。

这时候我们再次创建第二个临时pod \[root@kmaster \~]# kubectl run nginx2 --image nginx --image-pull-policy IfNotPresent --rm -ti -- bash If you don't see a command prompt, try pressing enter.

root@nginx2:/# env |grep SVC1 --有svc1变量 SVC1\_SERVICE\_HOST=10.102.158.48 SVC1\_PORT\_5500\_TCP\_ADDR=10.102.158.48 SVC1\_PORT=tcp://10.102.158.48:5500 SVC1\_PORT\_5500\_TCP\_PROTO=tcp SVC1\_PORT\_5500\_TCP\_PORT=5500 SVC1\_SERVICE\_PORT=5500 SVC1\_PORT\_5500\_TCP=tcp://10.102.158.48:5500

root@nginx2:/# env |grep SVC2 --有svc2变量 SVC2\_PORT\_5500\_TCP=tcp://10.96.72.69:5500 SVC2\_PORT\_5500\_TCP\_PROTO=tcp SVC2\_SERVICE\_PORT=5500 SVC2\_PORT\_5500\_TCP\_PORT=5500 SVC2\_PORT\_5500\_TCP\_ADDR=10.96.72.69 SVC2\_SERVICE\_HOST=10.96.72.69 SVC2\_PORT=tcp://10.96.72.69:5500 root@nginx2:/#

懂了这个变量之后，我们采用变量方式来创建blog

### 客户端访问 | |

### wordpress svc| | | wordpress pod| | |--> wordpress pod 如何发现 mysql svc 的IP地址 mysql svc | | | mysql pod |

原来在wrodpress pod 对应的yaml文件里面需要指定 SVC 对应的ClusterIP地址，现在不需要指定IP了， 因为创建pod的时候，会加载现有SVC中的环境变量，其中一个是 svc名称\_SERVICE\_HOST ，我们只需要 指定这个变量即可。例如：

\[root@kmaster svc]# kubectl exec -ti blog -- bash

root@blog:/var/www/html# env |grep SVC\* DBSVC\_PORT=tcp://10.107.243.88:3306 DBSVC\_PORT\_3306\_TCP\_ADDR=10.107.243.88 DBSVC\_PORT\_3306\_TCP=tcp://10.107.243.88:3306 DBSVC\_PORT\_3306\_TCP\_PORT=3306 DBSVC\_SERVICE\_PORT=3306 DBSVC\_SERVICE\_HOST=10.107.243.88 DBSVC\_PORT\_3306\_TCP\_PROTO=tcp

现在db的svc叫 DBSVC，所以变量为 DBSVC\_SERVICE\_HOST，修改yaml文件： \[root@kmaster svc]# vim blog.yaml \[root@kmaster svc]# cat blog.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: blog name: blog spec: containers:

* env:
  * name: WORDPRESS\_DB\_HOST value: $(DBSVC\_SERVICE\_HOST)
  * name: WORDPRESS\_DB\_USER value: root
  * name: WORDPRESS\_DB\_PASSWORD value: redhat
  * name: WORDPRESS\_DB\_NAME value: wordpress image: wordpress name: blog resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster svc]# kubectl delete -f blog.yaml pod "blog" deleted \[root@kmaster svc]# kubectl apply -f blog.yaml pod/blog created \[root@kmaster svc]# kubectl get pod NAME READY STATUS RESTARTS AGE blog 1/1 Running 0 9s mysql 1/1 Running 0 43m

接下来，重新登录下，看安装wordpress 能否跳过数据库对接动作。

\[root@kmaster svc]# kubectl get svc -o wide NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE SELECTOR blogsvc NodePort 10.109.29.93 80:30030/TCP 23m run=blog

http://192.168.100.148:30030/ ok，没有任何问题。这就是变量方式发现服务。

这种方式也有局限性，或许变量必须有先后顺序，且只能在同一个命名空间里面才能获得对应的svc变量。 一旦svc重建，地址和名称就发生变化了，上层应用也需要重新对接，很是麻烦。

### 3.DNS方式（建议）

\[root@kmaster \~]# kubectl get pod -n kube-system NAME READY STATUS RESTARTS AGE calico-kube-controllers-74677b4c5f-wg6xw 1/1 Running 4 (104m ago) 29d calico-node-76c4c 1/1 Running 3 (104m ago) 29d calico-node-dfdpt 1/1 Running 3 (104m ago) 29d calico-node-m684l 1/1 Running 3 (104m ago) 29d coredns-c676cc86f-hcpsm 1/1 Running 3 (104m ago) 29d coredns-c676cc86f-kp9gp 1/1 Running 3 (104m ago) 29d

在 kube-system 命名空间（或项目）中，有一个DNS服务器 Pod。

当我们在其他命名空间里面 创建 svc 的时候，svc会去dns上自动注册。对于DNS，就知道了svc的ip地址。

在同一个命名空间中，创建svc，创建pod，那么pod可以直接通过 svc 的服务名来访问。不需要找svc对应的ClusterIP地址。

比如上面有个svc1 ，我们手工创建一个临时容器。

\[root@kmaster svc]# cat svc1.yaml apiVersion: v1 kind: Service metadata: creationTimestamp: null labels: app: nginx name: svc1 spec: ports:

* port: 80 protocol: TCP targetPort: 80 selector: app: nginx status: loadBalancer: {}

\[root@kmaster svc]# kubectl apply -f svc1.yaml service/svc1 created

\[root@kmaster \~]# kubectl run busybox1 --image busybox --image-pull-policy IfNotPresent --rm -ti -- sh #busybox和alpine 默认没有curl命令 / # wget svc1 Connecting to svc1 (10.105.57.91:80) saving to 'index.html' index.html 100% |\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*| 4 0:00:00 ETA 'index.html' saved / #

或者用aipine \[root@kmaster \~]# kubectl run alpine1 --image alpine --image-pull-policy IfNotPresent --rm -ti -- sh If you don't see a command prompt, try pressing enter. / # wget svc1 Connecting to svc1 (10.105.57.91:80) saving to 'index.html' index.html 100% |\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*| 4 0:00:00 ETA 'index.html' saved / #

如果遇到以下错误。 \[root@kmaster \~]# kubectl run alpine --image alpine --image-pull-policy IfNotPresent --rm -ti -- sh If you don't see a command prompt, try pressing enter. / # ls / bin etc lib mnt proc run srv tmp var dev home media opt root sbin sys usr / # curl -s svc1 sh: curl: not found / # wget svc1 Connecting to svc1 (10.98.192.61:80) --这里一定注意，使用服务器默认连接80端口，但是上面创建的是 5500 端口 wget: can't connect to remote host (10.98.192.61): Connection refused

***

如果创建pod的时候看到这个错误 Warning Failed 11s (x2 over 12s) kubelet\
Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "bash": executable file not found in $PATH: unknown

或者

Warning Failed 4s (x3 over 22s) kubelet Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "/bin/bash": stat /bin/bash: no such file or directory: unknown

### 大概率是因为不支持 bash，换成 sh

可以看到，直接通过服务名就可以访问到了。为什么嗯？ 我们平常访问阿里，输入的是 aliyun.com，这个名称，中间会有个过程。转发给DNS。

\----默认命名空间----- | DNS pod服务器 | | DNS pod服务器 | DNS服务器 | DNS SVC | -----|--------------- | | | | -----|-------------- | | | svc1<------pod | | | ---自定义命名空间----

pod 在找svc1名称的时候，会转发给DNS pod服务器，说这个名称对应IP地址是什么，DNS进行解析后返回给pod。 于是pod就知道找svc1了。

查看记录dns文件，记录的这个ip地址是那里的？

/ # cat /etc/resolv.conf search sales.svc.cluster.local svc.cluster.local cluster.local localdomain nameserver 10.96.0.10 options ndots:5

在kube-system命名空间中有个svc，叫 kube-dns，记录的就是它。 \[root@kmaster svc]# kubectl get svc -n kube-system NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kube-dns ClusterIP 10.96.0.10 53/UDP,53/TCP,9153/TCP 29d metrics-server ClusterIP 10.110.81.164 443/TCP 29d

如果切换到其他命名空间呢？ \[root@kmaster \~]# kubens default Context "kubernetes-admin@kubernetes" modified. Active namespace is "default". \[root@kmaster \~]# kubectl run alpine1 --image alpine --image-pull-policy IfNotPresent --rm -ti -- sh If you don't see a command prompt, try pressing enter. / # wget svc1 wget: bad address 'svc1'

不可以了。因为虽然kube-system命名空间是整个集群公用的，但是你用B空间的pod连接A空间的 svc，默认B空间中是没有这个svc的。 如果想要连接，可以使用 svc名称.命名空间 格式来定义。例如：

/ # wget svc1.sales Connecting to svc1.sales (10.105.57.91:80) saving to 'index.html' index.html 100% |\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*| 4 0:00:00 ETA 'index.html' saved

接下来，我们删除wordpress ，修改yaml文件

\[root@kmaster svc]# kubens sales Context "kubernetes-admin@kubernetes" modified. Active namespace is "sales".

\[root@kmaster svc]# kubectl delete -f blog.yaml pod "blog" deleted

\[root@kmaster svc]# vim blog.yaml \[root@kmaster svc]# cat blog.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: blog name: blog spec: containers:

* env:
  * name: WORDPRESS\_DB\_HOST value: dbsvc ---> 直接输入svc对应的名称即可 kubectl get svc
  * name: WORDPRESS\_DB\_USER value: root
  * name: WORDPRESS\_DB\_PASSWORD value: redhat
  * name: WORDPRESS\_DB\_NAME value: wordpress image: wordpress name: blog resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster svc]# kubectl apply -f blog.yaml pod/blog created

\[root@kmaster svc]# kubectl get pod NAME READY STATUS RESTARTS AGE blog 1/1 Running 0 27s mysql 1/1 Running 0 99m

再次刷新网页 http://192.168.100.148:30030/ 连接成功且无需配置数据库。

为什么物理机可以连接node主机的IP地址呢？接下来讲解服务的发布。

### 服务发布

svc的clusterip只是集群内部可见的，我们创建集群的目的，是为了给外界主机访问的。 可这个ip无法从外部直接访问的，这时候，可以把该服务发布出去，让外界主机可以访问到。 要发布，有几种方式： 1.NodePort 2.LoadBalance(依靠公网地址池，成本非常昂贵) 3.ClusterIP（这个不算是服务发布） 4.Ingress

### 1.NodePort

### 默认情况下，外界主机是无法连通svc的clusterip地址的。

### | | | svc 端口映射-->| 把svc使用NodePort方式映射到主机上（端口），不仅仅是master，集群内所有的主机都有映射。 | |之后通过主机:端口 方式就可以访问到 svc，最后svc通过kube-proxy（iptables规则）进行转发到后端pod上。

通过NodePort映射出去有两种方法： 1.在创建服务的时候，直接使用 type 关键字来指定 2.可以在线修改

方法1：使用type \[root@kmaster svc]# kubectl apply -f pod3.yaml deployment.apps/nginx-deployment created

### vim pod3.yaml

apiVersion: apps/v1 kind: Deployment metadata: name: nginx-deployment1 labels: app: nginx spec: replicas: 3 selector: matchLabels: app: nginx template: metadata: labels: app: nginx spec: containers: - name: nginx image: nginx imagePullPolicy: IfNotPresent

\[root@kmaster svc]# kubectl get po NAME READY STATUS RESTARTS AGE nginx-deployment-57cc89bc77-2grl2 1/1 Running 0 2s nginx-deployment-57cc89bc77-57849 1/1 Running 0 2s nginx-deployment-57cc89bc77-fckwf 1/1 Running 0 2s

\[root@kmaster svc]# \[root@kmaster svc]# kubectl expose --name svc1 deployment nginx-deployment --port 80 --target-port 80 --type= pod1.yaml pod2.yaml pod3.yaml podtest.yaml svc1.yaml\
\[root@kmaster svc]# kubectl expose --name svc1 deployment nginx-deployment --port 80 --target-port 80 --type=NodePort service/svc1 exposed \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE svc1 NodePort 10.109.140.131 80:30607/TCP 4s

会随机找到一个 30000以上的端口来指定 注意：新版本（21有，24.3及之后没了）的 K8s 在使用 nodeport 类型的 Servive 在 node 上看不到监听的端口了 测试22版本和23版本。

之前版本可以通过 netstat 命令查看到对应的端口，但是新版本查不到，可以通过iptables命令查看下。 \[root@kmaster svc]# netstat -ntulp |grep 30607

\[root@kmaster svc]# iptables -S -t nat |grep 30607 -A KUBE-NODEPORTS -p tcp -m comment --comment "default/svc1" -m tcp --dport 30607 -j KUBE-EXT-DZERXHZGH3HKTTEJ

之后，我们可以访问集群内任意一个主机ip:30607 来访问nginx服务。

http://192.168.146.139:30607/ http://192.168.146.140:30607/ http://192.168.146.141:30607/

Port：svc本身开放的端口 targetPort：svc转发给后端pod，pod上的端口 NodePort：物理机的端口

现在把端口30607改为 30888

\[root@kmaster svc]# kubectl edit svc svc1 service/svc1 edited

直接切换到最后面。把原端口改为30888，保存并退出。 ports:

* nodePort: 30888 port: 80 protocol: TCP targetPort: 80 selector: app: nginx sessionAffinity: None type: NodePort status: loadBalancer: {}

\[root@kmaster svc]# iptables -S -t nat |grep 30607 \[root@kmaster svc]# iptables -S -t nat |grep 30888 -A KUBE-NODEPORTS -p tcp -m comment --comment "default/svc1" -m tcp --dport 30888 -j KUBE-EXT-DZERXHZGH3HKTTE

这时候再次访问，就要更换端口30888了。

方法2：在线修改ClusterIP为NodePort \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE blog NodePort 10.108.58.66 80:30604/TCP 8h db ClusterIP 10.101.35.49 3306/TCP 8h

\[root@kmaster svc]# kubectl edit svc db service/db edited

直接定位到最后一行

ports:

* port: 3306 protocol: TCP targetPort: 3306 selector: run: mysql sessionAffinity: None type: NodePort status: loadBalancer: {}

\[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE blog NodePort 10.108.58.66 80:30604/TCP 8h db NodePort 10.101.35.49 3306:30690/TCP 8h

如何改回来呢？

\[root@kmaster svc]# kubectl edit svc db service/db edited

直接定位到最后一行

ports:

* port: 3306 --删除上面一行 nodeport protocol: TCP targetPort: 3306 selector: run: mysql sessionAffinity: None type: ClusterIP ---改

\[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE blog NodePort 10.108.58.66 80:30604/TCP 8h db ClusterIP 10.101.35.49 3306/TCP 8h

这时候你会问，这端口很不方便，访问不方便，还要记住端口。 对于业务来说，集群内部主机，不会暴露在互联网上，前面还要有负载均衡器，防火墙等，可以把防火墙或LB的端口80公开出去，然后80转发到后端指定的端口上面去。 通过DNS指定到防火墙或LB上，使用默认80端口即可。 集群使用的都是私有地址。

```
       互联网
          |
          |DNS指向防火墙
          |       
       防火墙或LB（公开80）
        |          |
        |          |
```

master node1 node2

### 2.LoadBalance

注意：搭建集群，每个节点都要有一个公网ip，代价高，直接暴露在互联网上。 在某节点上有个svc1，指定模式为lb，就会从地址池里面分配一个ip地址。 外网用户直接访问该公网ip即可访问。

```
          地址池（公网ip地址池，网络运营商申请）
           |
```

loadbalance |地址池为节点分配公网ip地址 | |\
master(端口lb) node1 node2 | | svc1

最终公网ip会出现在 external-ip 这个字段上（默认none） \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE blog NodePort 10.108.58.66 80:30604/TCP 8h

实验：

01.安装模拟插件

https://metallb.universe.tf/installation/

kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml 网络原因，先把文件拷贝过来

\[root@kmaster svc]# kubectl get ns NAME STATUS AGE calico-apiserver Active 11d calico-system Active 11d default Active 11d kube-node-lease Active 11d kube-public Active 11d kube-system Active 11d tigera-operator Active 11d \[root@kmaster svc]# ls metallb-native.yaml pod1.yaml pod2.yaml pod3.yaml podtest.yaml svc1.yaml

\#用到的镜像 \[root@kmaster svc]# grep image metallb-native.yaml -n 1712: image: quay.io/metallb/controller:v0.13.9 1805: image: quay.io/metallb/speaker:v0.13.9

可以提前手工下载，并修改yaml文件的imagepullpolicy

\[root@kmaster svc]# kubectl apply -f metallb-native.yaml namespace/metallb-system created customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io created customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created customresourcedefinition.apiextensions.k8s.io/communities.metallb.io created customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created serviceaccount/controller created serviceaccount/speaker created role.rbac.authorization.k8s.io/controller created role.rbac.authorization.k8s.io/pod-lister created clusterrole.rbac.authorization.k8s.io/metallb-system:controller created clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created rolebinding.rbac.authorization.k8s.io/controller created rolebinding.rbac.authorization.k8s.io/pod-lister created clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created secret/webhook-server-cert created service/webhook-service created deployment.apps/controller created daemonset.apps/speaker created validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration created

\[root@kmaster svc]# kubectl get ns NAME STATUS AGE calico-apiserver Active 11d calico-system Active 11d default Active 11d kube-node-lease Active 11d kube-public Active 11d kube-system Active 11d metallb-system Active 4s tigera-operator Active 11d

等待一会，集群内所有节点下载完毕 \[root@knode2 \~]# crictl img |grep me quay.io/metallb/controller v0.13.9 26952499c3023 27.8MB quay.io/metallb/speaker v0.13.9 697605b359357 50.1MB

\[root@knode1 \~]# crictl img |grep me quay.io/metallb/speaker v0.13.9 697605b359357 50.1MB

\[root@kmaster svc]# crictl img |grep me quay.io/metallb/speaker v0.13.9 697605b359357 50.1MB

\[root@kmaster svc]# kubectl get ns NAME STATUS AGE calico-apiserver Active 11d calico-system Active 11d default Active 11d kube-node-lease Active 11d kube-public Active 11d kube-system Active 11d metallb-system Active 6m38s tigera-operator Active 11d

\[root@kmaster svc]# kubectl get pod -n metallb-system NAME READY STATUS RESTARTS AGE controller-68bf958bf9-56scw 1/1 Running 0 6m45s speaker-hhtv9 1/1 Running 0 6m44s speaker-wf69r 1/1 Running 0 6m44s speaker-xw7z6 1/1 Running 0 6m44s

02.创建地址池 https://metallb.universe.tf/configuration/

当前master节点ip地址为：192.168.146.139

安装小工具查看网络段，计算ip子网 \[root@kmaster svc]# yum install -y epel-release \[root@kmaster svc]# yum install -y sipcalc \[root@kmaster svc]# sipcalc 192.168.146.139/24 -\[ipv4 : 192.168.146.139/24] - 0

\[CIDR] Host address - 192.168.146.139 Host address (decimal) - 3232273035 Host address (hex) - C0A8928B Network address - 192.168.146.0 Network mask - 255.255.255.0 Network mask (bits) - 24 Network mask (hex) - FFFFFF00 Broadcast address - 192.168.146.255 Cisco wildcard - 0.0.0.255 Addresses in network - 256 Network range - 192.168.146.0 - 192.168.146.255 Usable range - 192.168.146.1 - 192.168.146.254

## 创建地址池

\[root@kmaster svc]# vim pools.yaml \[root@kmaster svc]# cat pools.yaml apiVersion: metallb.io/v1beta1 kind: IPAddressPool metadata: name: first-pool namespace: metallb-system spec: addresses:

* 192.168.146.210-192.168.146.220

## 创建实例并绑定地址池

\[root@kmaster svc]# vim l2.yaml \[root@kmaster svc]# cat l2.yaml apiVersion: metallb.io/v1beta1 kind: L2Advertisement metadata: name: example namespace: metallb-system spec: ipAddressPools:

* first-pool

\[root@kmaster svc]# kubectl apply -f pools.yaml ipaddresspool.metallb.io/first-pool created

\[root@kmaster svc]# kubectl apply -f l2.yaml l2advertisement.metallb.io/example created

\[root@kmaster svc]# kubectl get ipaddresspools.metallb.io -n metallb-system NAME AUTO ASSIGN AVOID BUGGY IPS ADDRESSES first-pool true false \["192.168.146.200-192.168.146.220"]

03.创建svc

\[root@kmaster svc]# kubectl expose --name svc666 deployment nginx-deployment --port 80 --target-port 80 --type LoadBalancer service/svc666 exposed \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 12d svc2 NodePort 10.97.241.127 80:30408/TCP 48m svc666 LoadBalancer 10.110.85.222 192.168.146.200 80:31541/TCP 7s

最后通过 192.168.146.200 访问nginx服务。注意：通过公网IP直接访问，不需要加端口号！！！ ip地址池可以有多个，但实例只能有一个。

### 4.Ingress（重点/推荐）本质上不是服务的发布类型【CCE 路由】也是一种访问k8s集群的方式

代理和反向代理的概念

### 华为云 | 代理服务器 proxy 一般用于出去

\| pc电脑

反向代理：一般用于进来的数据包

用户（www1.baidu.com/www2.baidu.com 解析地址都是一样的）

反向代理服务器 nginx

server1 server2

根据用户不同需求，反向代理会将数据包转发到不同的server上面去。 如下图：箭头往下 ingress-nginx

```
反向代理 nginx控制器
          |         
```

### www1.xx.com www2.xx.com ww3.xx.com ---虚拟主机，定义规则：访问www1，转发到service1

svcn1 svcn2 svcn3 ---clusterip

n1(111) n2 (222) n3（333） /index.html 333 /abc/index.html 444

为什么要使用反向代理？ NodePort/LB

### 映射物理机端口1 端口2 端口3 ------|----|-----|------|----- | | | | | | | svc1 svc2 svc3 svc4 | | |

假如创建了很多svc，都要映射到物理机的端口上面去（NodePort类型），这样物理机开放的端口很多，安全隐患比较大。 如果使用了ingress类型就好办了。

把负载均衡器nginx 虚拟主机，映射一个端口出去，外部主机访问nginx，如果输入的是www1，那么切换到service1上面去，其他类似。 就不需要把所有的服务端口映射出去。

步骤： 1.安装nginx（反向代理/负载均衡--NodePort类型发布出去） 2.创建三个svc 3.创建三个pod（111/222/333）

定义规则： 客户端访问www1 ，转发到svc1上面，访问www2，转发svc2

实验： 01.创建svc及pod \[root@kmaster \~]# kubectl delete svc svc{666,888,999} service "svc666" deleted service "svc888" deleted service "svc999" deleted \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 12d

\[root@kmaster \~]# kubectl run n1 --image nginx --image-pull-policy Never \[root@kmaster \~]# kubectl run n2 --image nginx --image-pull-policy Never \[root@kmaster \~]# kubectl run n3 --image nginx --image-pull-policy Never \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE n1 1/1 Running 0 19s n2 1/1 Running 0 9s n3 1/1 Running 0 4s

\[root@kmaster \~]# kubectl exec -ti n1 -- bash root@n1:/# echo 111 > /usr/share/nginx/html/index.html root@n1:/# exit exit \[root@kmaster \~]# kubectl exec -ti n2 -- bash root@n2:/# echo 222 > /usr/share/nginx/html/index.html root@n2:/# exit exit \[root@kmaster \~]# kubectl exec -ti n3 -- bash root@n3:/# echo 333 > /usr/share/nginx/html/index.html root@n3:/# mkdir /usr/share/nginx/html/abc root@n3:/# echo 444 > /usr/share/nginx/html/abc/index.html root@n3:/# exit exit

\[root@kmaster \~]# kubectl expose --name svc1 pod n1 --port 80 --target-port 80 service/svc1 exposed \[root@kmaster \~]# kubectl expose --name svc2 pod n2 --port 80 --target-port 80 service/svc2 exposed \[root@kmaster \~]# kubectl expose --name svc3 pod n3 --port 80 --target-port 80 service/svc3 exposed \[root@kmaster \~]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 12d svc1 ClusterIP 10.103.170.252 80/TCP 18s svc2 ClusterIP 10.97.249.226 80/TCP 12s svc3 ClusterIP 10.106.58.200 80/TCP 6s

02.配置反向代理

https://github.com/kubernetes/ingress-nginx https://kubernetes.github.io/ingress-nginx/deploy/ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml

将代码复制重命名上传。 \[root@kmaster \~]# ls anaconda-ks.cfg deploy.yaml \[root@kmaster \~]# grep image deploy.yaml --看需要安装哪些包，这些包应该可以在线联网安装，也可以提前手工下载 image: registry.k8s.io/ingress-nginx/controller:v1.6.4@sha256:15be4666c53052484dd2992efacf2f50ea77a78ae8aa21ccd91af6baaa7ea22f imagePullPolicy: IfNotPresent image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f imagePullPolicy: IfNotPresent image: registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343@sha256:39c5b2e3310dc4264d638ad28d9d1d96c4cbb2b2dcfb52368fe4e3c63f61e10f imagePullPolicy: IfNotPresent

因为镜像无法pull下来，所以申请香港主机 yum install -y yum-utils vim bash-completion net-tools wget systemctl stop firewalld systemctl disable firewalld yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo yum list docker-ce --showduplicates | sort -r yum install -y docker-ce systemctl start docker systemctl enable docker docker -v systemctl is-active docker systemctl is-enabled docker

为方便后期使用，安装完docker后，直接做成私有镜像。 进行下载，之后推送到阿里云中 \[root@ecs-docker \~]# docker pull registry.k8s.io/ingress-nginx/controller:v1.6.4 \[root@ecs-docker \~]# docker pull registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343

推送到阿里云 \[root@ecs-docker \~]# docker login --username=clisdodo@126.com registry.cn-hangzhou.aliyuncs.com Password: WARNING! Your password will be stored unencrypted in /root/.docker/config.json. Configure a credential helper to remove this warning. See https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded \[root@ecs-docker \~]# docker tag registry.k8s.io/ingress-nginx/controller:v1.6.4 registry.cn-hangzhou.aliyuncs.com/cloudcs/controller:v1.6.4 \[root@ecs-docker \~]# docker tag registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20220916-gd32f8c343 registry.cn-hangzhou.aliyuncs.com/cloudcs/kube-webhook-certgen:v20220916-gd32f8c343

\[root@ecs-docker \~]# docker push registry.cn-hangzhou.aliyuncs.com/cloudcs/kube-webhook-certgen:v20220916-gd32f8c343 \[root@ecs-docker \~]# docker push registry.cn-hangzhou.aliyuncs.com/cloudcs/controller:v1.6.4

然后，从阿里云下载（所有节点） \[root@kmaster \~]# crictl pull registry.cn-hangzhou.aliyuncs.com/cloudcs/kube-webhook-certgen:v20220916-gd32f8c343 Image is up to date for sha256:520347519a8caefcdff1c480be13cea37a66bccf517302949b569a654b0656b5 \[root@kmaster \~]# crictl pull registry.cn-hangzhou.aliyuncs.com/cloudcs/controller:v1.6.4 Image is up to date for sha256:7744eedd958ffb7011ea5dda4b9010de8e69a9f114ba3312c149bb7943ddbcd6

修改deploy.yaml文件 \[root@kmaster \~]# vim deploy.yaml 将里面的镜像路径修改成阿里云私有镜像仓库地址

\[root@kmaster \~]# kubectl apply -f deploy.yaml namespace/ingress-nginx created serviceaccount/ingress-nginx created serviceaccount/ingress-nginx-admission created role.rbac.authorization.k8s.io/ingress-nginx created role.rbac.authorization.k8s.io/ingress-nginx-admission created clusterrole.rbac.authorization.k8s.io/ingress-nginx created clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created rolebinding.rbac.authorization.k8s.io/ingress-nginx created rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created configmap/ingress-nginx-controller created service/ingress-nginx-controller created service/ingress-nginx-controller-admission created deployment.apps/ingress-nginx-controller created job.batch/ingress-nginx-admission-create created job.batch/ingress-nginx-admission-patch created ingressclass.networking.k8s.io/nginx created validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created

然后会看到一个命名空间及下面的pod \[root@kmaster \~]# kubectl get pod -n ingress-nginx NAME READY STATUS RESTARTS AGE ingress-nginx-admission-create-6gvmk 0/1 Completed 0 28s 上面两个是定时任务，正常状态为完成 ingress-nginx-admission-patch-s6pvw 0/1 Completed 0 28s ingress-nginx-controller-699577fcff-fs22p 1/1 Running 0 28s

创建好之后，默认会有两个svc \[root@kmaster \~]# kubectl get svc -n ingress-nginx NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE ingress-nginx-controller LoadBalancer 10.104.249.134 192.168.146.230 80:32335/TCP,443:31105/TCP 80m ingress-nginx-controller-admission ClusterIP 10.109.233.128 443/TCP 80m

查看deployment \[root@kmaster \~]# kubectl get deployments.apps -n ingress-nginx NAME READY UP-TO-DATE AVAILABLE AGE ingress-nginx-controller 1/1 1 1 81m

\---下面这段可以不要--不需要使用NodePort对外发布，使用它自己的LB即可---- 因为我们当前环境存在LB地址池，所以会自动获取公网IP，如果没有，建议手工以NodePort方式对外发布。 把nginx 的 svc发布出去 \[root@kmaster \~]# kubectl expose --name inss deployment ingress-nginx-controller --type NodePort -n ingress-nginx service/inss exposed \[root@kmaster \~]# kubectl get svc -n ingress-nginx NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE ingress-nginx-controller LoadBalancer 10.107.204.147 80:32101/TCP,443:31797/TCP 93m ingress-nginx-controller-admission ClusterIP 10.98.116.186 443/TCP 93m insss NodePort 10.97.120.222 80:31365/TCP,443:31682/TCP,8443:30099/TCP 88m

### 通过物理机的31365端口发布出去了。

03.配置规则

https://kubernetes.io/docs/concepts/services-networking/ingress/

\[root@kmaster svc]# kubectl get ingress No resources found in default namespace. \[root@kmaster svc]# kubectl get ingress -n ingress-nginx No resources found in ingress-nginx namespace.

现在是没有任何规则的

\[root@kmaster svc]# vim inss.yaml \[root@kmaster svc]# cat inss.yaml apiVersion: networking.k8s.io/v1 kind: Ingress metadata: name: ingress-wildcard-host spec: rules:

* host: "www.news.com" http: paths:
  * pathType: Prefix path: "/" backend: service: name: svc1 port: number: 80
* host: "www.edu.com" http: paths:
  * pathType: Prefix path: "/" backend: service: name: svc2 port: number: 80
* host: "www.mails.com" http: paths:
  * pathType: Prefix path: "/" backend: service: name: svc3 port: number: 80

\[root@kmaster svc]#

## 注意这里还要创建class 类，否则报错。如何使用默认的ingress类

https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/#default-ingress-class

### \[root@kmaster \~]# kubectl logs pods/ingress-nginx-controller-699577fcff-sdfzd -n ingress-nginx

NGINX Ingress controller Release: v1.6.4 Build: 69e8833858fb6bda12a44990f1d5eaa7b13f4b75 Repository: https://github.com/kubernetes/ingress-nginx nginx version: nginx/1.21.6

***

W0309 13:11:55.254894 7 client\_config.go:618] Neither --kubeconfig nor --master was specified. Using the inClusterConfig. This might not work. I0309 13:11:55.254975 7 main.go:209] "Creating API client" host="https://10.96.0.1:443" I0309 13:11:55.257957 7 main.go:253] "Running in Kubernetes cluster" major="1" minor="26" git="v1.26.0" state="clean" commit="b46a3f887ca979b1a5d14fd39cb1af43e7e5d12d" platform="linux/amd64" I0309 13:11:55.349056 7 main.go:104] "SSL fake certificate created" file="/etc/ingress-controller/ssl/default-fake-certificate.pem" I0309 13:11:55.656442 7 ssl.go:533] "loading tls certificate" path="/usr/local/certificates/cert" key="/usr/local/certificates/key" I0309 13:11:55.683116 7 nginx.go:261] "Starting NGINX Ingress controller" I0309 13:11:55.741677 7 event.go:285] Event(v1.ObjectReference{Kind:"ConfigMap", Namespace:"ingress-nginx", Name:"ingress-nginx-controller", UID:"9868b2ad-a76f-4883-9c2a-5ea181e36959", APIVersion:"v1", ResourceVersion:"4487", FieldPath:""}): type: 'Normal' reason: 'CREATE' ConfigMap ingress-nginx/ingress-nginx-controller I0309 13:11:56.886201 7 nginx.go:304] "Starting NGINX process" I0309 13:11:56.887267 7 leaderelection.go:248] attempting to acquire leader lease ingress-nginx/ingress-nginx-leader... I0309 13:11:56.887926 7 nginx.go:324] "Starting validation webhook" address=":8443" certPath="/usr/local/certificates/cert" keyPath="/usr/local/certificates/key" I0309 13:11:56.888140 7 controller.go:188] "Configuration changes detected, backend reload required" I0309 13:11:56.950072 7 leaderelection.go:258] successfully acquired lease ingress-nginx/ingress-nginx-leader I0309 13:11:56.950255 7 status.go:84] "New leader elected" identity="ingress-nginx-controller-699577fcff-sdfzd" I0309 13:11:57.048436 7 controller.go:205] "Backend successfully reloaded" I0309 13:11:57.048502 7 controller.go:216] "Initial sync, sleeping for 1 second" I0309 13:11:57.048884 7 event.go:285] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-699577fcff-sdfzd", UID:"9766518b-b5a2-4baa-8f18-4b2ba8616970", APIVersion:"v1", ResourceVersion:"4552", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration W0309 13:30:27.464155 7 controller.go:278] ignoring ingress ingress-wildcard-host in default based on annotation : ingress does not contain a valid IngressClass I0309 13:30:27.464201 7 main.go:100] "successfully validated configuration, accepting" ingress="default/ingress-wildcard-host" I0309 13:30:27.466612 7 store.go:429] "Ignoring ingress because of error while validating ingress class" ingress="default/ingress-wildcard-host" error="ingress does not contain a valid IngressClass" I0309 13:49:08.929631 7 store.go:399] "Ignoring ingress because of error while validating ingress class" ingress="default/ingress-wildcard-host" error="ingress does not contain a valid IngressClass" W0309 13:50:29.008340 7 controller.go:278] ignoring ingress ingress-wildcard-host in default based on annotation : ingress does not contain a valid IngressClass I0309 13:50:29.008377 7 main.go:100] "successfully validated configuration, accepting" ingress="default/ingress-wildcard-host" I0309 13:50:29.014798 7 store.go:429] "Ignoring ingress because of error while validating ingress class" ingress="default/ingress-wildcard-host" error="ingress does not contain a valid IngressClass"

你可以将一个特定的 IngressClass 标记为集群默认 Ingress 类。 将一个 IngressClass 资源的 ingressclass.kubernetes.io/is-default-class 注解设置为 true 将确保新的未指定 ingressClassName 字段的 Ingress 能够分配为这个默认的 IngressClass.

默认有个nginx的class类。 \[root@kmaster \~]# kubectl get ingressclasses.networking.k8s.io --请问这个nginx是什么时候有的呢？验证？ NAME CONTROLLER PARAMETERS AGE nginx k8s.io/ingress-nginx 71m

\[root@kmaster \~]# kubectl edit ingressclasses.networking.k8s.io nginx ingressclass.networking.k8s.io/nginx edited

添加ingressclass一行 annotations: ingressclass.kubernetes.io/is-default-class: "true" kubectl.kubernetes.io/last-applied-configuration: |

修改完毕后，需要重新创建ingress \[root@kmaster \~]# kubectl delete -f insss.yaml ingress.networking.k8s.io "ingress-wildcard-host" deleted

\[root@kmaster \~]# kubectl apply -f insss.yaml

\[root@kmaster svc]# kubectl get pod NAME READY STATUS RESTARTS AGE n1 1/1 Running 0 167m n2 1/1 Running 0 167m n3 1/1 Running 0 167m

\[root@kmaster \~]# kubectl get ingress NAME CLASS HOSTS ADDRESS PORTS AGE ingress-wildcard-host nginx www.meme1.com,www.meme2.com 80 2m41s

\[root@kmaster \~]# kubectl describe ingress ingress-wildcard-host Name: ingress-wildcard-host Labels: Namespace: default Address:\
Ingress Class: nginx Default backend: Rules: Host Path Backends

***

www.meme1.com\
/ svc1:80 (10.244.195.133:80) www.meme2.com\
/ svc2:80 (10.244.195.134:80) Annotations: Events: Type Reason Age From Message

***

Normal Sync 3m19s nginx-ingress-controller Scheduled for sync --同步成功

负载均衡是在哪里运行的？ \[root@kmaster svc]# kubectl get pod -n ingress-nginx -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES ingress-nginx-admission-create-hj6wq 0/1 Completed 0 104m 10.244.195.131 knode1 ingress-nginx-admission-patch-ktmq4 0/1 Completed 0 104m 10.244.195.132 knode1 ingress-nginx-controller-699577fcff-zvjzh 1/1 Running 0 104m 10.244.69.221 knode2

04.测试 随便找台主机。不改物理机，修改hosts \[root@gui \~]# echo '192.168.146.141 www.meme1.com' >> /etc/hosts 添加 ingress-nginx-controller LB 地址（公网ip） \[root@gui \~]# echo '192.168.146.141 www.meme2.com' >> /etc/hosts \[root@gui \~]# cat /etc/hosts 127.0.0.1 localhost localhost.localdomain localhost4 localhost4.localdomain4 ::1 localhost localhost.localdomain localhost6 localhost6.localdomain6 192.168.146.141 www.meme1.com 192.168.146.141 www.meme2.com

解析的地址都是一样的。 在测试主机上打开浏览器。 如果不起作用怎么办？查看ingress-nginx ns 下面 nginx-controller 的日志。 \[root@kmaster \~]# kubectl logs pods/ingress-nginx-controller-699577fcff-sdfzd -n ingress-nginx 或许会报错class类错误。 正确的日志是这样： I0309 14:41:08.575676 7 controller.go:188] "Configuration changes detected, backend reload required" I0309 14:41:08.608440 7 controller.go:205] "Backend successfully reloaded" I0309 14:41:08.608834 7 event.go:285] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress-nginx", Name:"ingress-nginx-controller-699577fcff-sdfzd", UID:"9766518b-b5a2-4baa-8f18-4b2ba8616970", APIVersion:"v1", ResourceVersion:"4552", FieldPath:""}): type: 'Normal' reason: 'RELOAD' NGINX reload triggered due to a change in configuration I0309 14:41:15.978466 7 admission.go:149] processed ingress via admission controller {testedIngressLength:1 testedIngressTime:0.031s renderingIngressLength:1 renderingIngressTime:0s admissionTime:29.6kBs testedConfigurationSize:0.031} I0309 14:41:15.978490 7 main.go:100] "successfully validated configuration, accepting" ingress="default/ingress-wildcard-host" I0309 14:41:15.985141 7 store.go:433] "Found valid IngressClass" ingress="default/ingress-wildcard-host" ingressclass="nginx" I0309 14:41:15.985448 7 controller.go:188] "Configuration changes detected, backend reload required" I0309 14:41:15.986566 7 event.go:285] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"ingress-wildcard-host", UID:"fd2171aa-7aec-4403-882d-c36f73dc0e6b", APIVersion:"networking.k8s.io/v1", ResourceVersion:"16073", FieldPath:""}): type: 'Normal' reason: 'Sync' Scheduled for sync I0309 14:41:16.055059 7 controller.go:205] "Backend successfully reloaded"

***

给两个pod分别创建svc，类型LB \[root@kmaster svc]# kubectl expose pod web201 --name web201 --port 80 --target-port 80 --type LoadBalancer service/web201 exposed \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 22d svc01 NodePort 10.97.82.42 80:30542/TCP 3h38m svc666 LoadBalancer 10.99.217.13 192.168.100.210 80:31979/TCP 163m svcn1 ClusterIP 10.97.11.246 80/TCP 122m svcn2 ClusterIP 10.111.2.47 80/TCP 122m svcn3 ClusterIP 10.106.124.46 80/TCP 122m web NodePort 10.107.120.41 80:32418/TCP 22m web201 LoadBalancer 10.107.66.15 192.168.100.212 80:30801/TCP 4s \[root@kmaster svc]# kubectl expose pod web202 --name web202 --port 80 --target-port 80 --type LoadBalancer service/web202 exposed \[root@kmaster svc]# kubectl get svc NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE kubernetes ClusterIP 10.96.0.1 443/TCP 22d svc01 NodePort 10.97.82.42 80:30542/TCP 3h38m svc666 LoadBalancer 10.99.217.13 192.168.100.210 80:31979/TCP 163m svcn1 ClusterIP 10.97.11.246 80/TCP 122m svcn2 ClusterIP 10.111.2.47 80/TCP 122m svcn3 ClusterIP 10.106.124.46 80/TCP 122m web NodePort 10.107.120.41 80:32418/TCP 22m web201 LoadBalancer 10.107.66.15 192.168.100.212 80:30801/TCP 17s web202 LoadBalancer 10.110.72.130 192.168.100.213 80:30498/TCP 2s

这时候，两个pod，两个网站，分别通过两个不同的端口来访问，一个公网ip就要占用一个端口。显然需求不合理。 虽然配合DNS可以进行同一个域名，进行网站流量分摊，但前提是网站内容是一样的，目的是为了高可用。 我们的需求是，网站不一样。这时候不同ip映射不同域名，一台主机上就多了很多端口，不安全。

现在的需求是，用同一个主机ip同一个端口，通过不同域名规则，来进行不同网站的访问。如何实现？通过ingress。
