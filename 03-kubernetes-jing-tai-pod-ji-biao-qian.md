# 03 Kubernetes 静态pod及标签

kubernetes 1.3版本引入了init container 初始化容器特性。主要用于在启动应用容器（app container）前来启动一个或多个初始化容器，作为应用容器的一个基础。

init 1 --> init 2 --> 所有的初始化容器加载运行完成后，才能运行 应用容器 --> app container

initContainers:

* name: init-myservice image: busybox:1.28 command:

\[root@kmaster \~]# kubectl explain pods.spec.containers

\[root@kmaster 327]# cat initpod2.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: initpod2 name: initpod2 spec: initContainers:

* image: alpine imagePullPolicy: IfNotPresent name: alpine command: \["sh","-c","echo 123 > /initest/test.txt"] volumeMounts:
  * name: testdir mountPath: /initest dnsPolicy: ClusterFirst restartPolicy: Always containers:
* image: nginx imagePullPolicy: IfNotPresent name: initpod2 resources: {} volumeMounts:
  * name: testdir mountPath: /test volumes:
* name: testdir emptyDir: {}

\[root@kmaster 327]# \[root@kmaster 327]# \[root@kmaster 327]# kubectl get pod NAME READY STATUS RESTARTS AGE initpod1 1/1 Running 0 22m initpod2 1/1 Running 0 113s \[root@kmaster 327]# \[root@kmaster 327]# kubectl exec -ti initpod2 -- bash Defaulted container "initpod2" out of: initpod2, alpine (init) root@initpod2:/# root@initpod2:/# ls /test/ test.txt root@initpod2:/# cat /test/test.txt 123 root@initpod2:/#

指定在哪个节点上运行pod，是通过利用标签来实现的。 标签的格式 xx=yy aa=bb aa/bb=cc aa-bb=cc aa.bb-cc/dd=ee

静态pod，注意不要在master上操作，因为master上跑的是集群核心静态pod，在knode1或knode2上去做。 1.创建一个目录 \[root@knode1 \~]# mkdir /etc/kubernetes/test

以下实验，版本不一样，效果不一样 v1.27.0版本 直接在master上，对应的静态pod所在目录里面，去编辑yaml文件，不用执行，即可看到pod运行。删除文件，pod随之删除。

2.修改配置文件 \[root@knode1 \~]# cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf \~# Note: This dropin only works with kubeadm and kubelet v1.11+ \[Service] Environment="KUBELET\_KUBECONFIG\_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf" Environment="KUBELET\_CONFIG\_ARGS=--config=/var/lib/kubelet/config.yaml --pod-manifest-path=/etc/kubernetes/test"

在环境变量参数里面加上 --pod-manifest-path=/etc/kubernetes/test ，指向创建的目录

3.在test目录里面编写yaml文件

注意：在master上这个目录 \[root@kmaster manifests]# pwd /etc/kubernetes/manifests

kubectl run pod1 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml > pod1.yaml \[root@knode1 test]# cat pod1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod1 name: pod1 spec: containers:

* image: nginx imagePullPolicy: IfNotPresent name: pod1 resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {} \[root@knode1 test]# \[root@knode1 test]# ls pod1.yaml

去master上查询，就会看到一个静态pod \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1-knode1 0/1 ContainerCreating 0 7s

一旦移走了yaml文件，静态pod也将被删除

\[root@knode1 test]# mv pod1.yaml /tmp/ \[root@knode1 test]# ls

\[root@kmaster \~]# kubectl get pod -o wide No resources found in default namespace.

\[root@knode1 test]# mv /tmp/pod1.yaml .

定义标签 1.为主机定义标签 \[root@kmaster \~]# kubectl label nodes knode1 aaa=knode1 node/knode1 labeled \[root@kmaster \~]# kubectl get nodes knode1 --show-labels NAME STATUS ROLES AGE VERSION LABELS knode1 Ready 5d21h v1.26.0 aaa=knode1,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=knode1,kubernetes.io/os=linux

2.为pod定义标签 \[root@kmaster \~]# kubectl get pods/pod1 --show-labels NAME READY STATUS RESTARTS AGE LABELS pod1 1/1 Running 0 69s run=pod1 \[root@kmaster \~]# vim pod1.yaml \[root@kmaster \~]# kubectl label pods/pod1 bbb=pod1 pod/pod1 labeled \[root@kmaster \~]# kubectl get pods/pod1 --show-labels NAME READY STATUS RESTARTS AGE LABELS pod1 1/1 Running 0 2m8s bbb=pod1,run=pod1

3.取消标签 \[root@kmaster \~]# kubectl get pods/pod1 --show-labels NAME READY STATUS RESTARTS AGE LABELS pod1 1/1 Running 0 2m8s bbb=pod1,run=pod1 \[root@kmaster \~]# kubectl label pods/pod1 bbb- pod/pod1 unlabeled \[root@kmaster \~]# kubectl label pod pod1 bbb- label "bbb" not found. pod/pod1 not labeled \[root@kmaster \~]# kubectl get pods/pod1 --show-labels NAME READY STATUS RESTARTS AGE LABELS pod1 1/1 Running 0 4m15s run=pod1 \[root@kmaster \~]# \[root@kmaster \~]# kubectl label nodes knode1 aaa- node/knode1 unlabeled \[root@kmaster \~]# kubectl get pods/knode1 --show-labels Error from server (NotFound): pods "knode1" not found \[root@kmaster \~]# kubectl get nodes knode1 --show-labels NAME STATUS ROLES AGE VERSION LABELS knode1 Ready 5d21h v1.26.0 beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=knode1,kubernetes.io/os=linux \[root@kmaster \~]#
