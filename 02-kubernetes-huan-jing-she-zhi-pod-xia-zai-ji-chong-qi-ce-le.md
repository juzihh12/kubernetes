# 02 Kubernetes环境设置 pod下载及重启策略

## 1 环境配置

### 1.1 命令行自动补全

\[root@kmaster \~]# vim /etc/profile \[root@kmaster \~]# cat /etc/profile

## /etc/profile

source <(kubectl completion bash) \[root@kmaster \~]# source /etc/profile

### 1.2 设置 metrics-server

https://github.com/kubernetes-sigs/metrics-server

更改两行参数       - args:         - --kubelet-insecure-tls         - --cert-dir=/tmp         - --secure-port=4443         - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname         - --kubelet-use-node-status-port         - --metric-resolution=15s         image: registry.cn-hangzhou.aliyuncs.com/cloudcs/metrics-server:v0.6.2

\[root@kmaster \~]# kubectl apply -f components.yaml serviceaccount/metrics-server created clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created clusterrole.rbac.authorization.k8s.io/system:metrics-server created rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created service/metrics-server created deployment.apps/metrics-server created apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created

\[root@kmaster \~]# kubectl top node NAME CPU(cores) CPU% MEMORY(bytes) MEMORY%\
kmaster 181m 9% 1479Mi 18%\
knode1 93m 4% 818Mi 10%\
knode2 72m 3% 781Mi 9%

### 1.3 了解namespace及相关操作

\[root@kmaster \~]# kubectl get pod -n kube-system NAME READY STATUS RESTARTS AGE coredns-5bbd96d687-4zl6m 1/1 Running 1 (4h49m ago) 21h coredns-5bbd96d687-w9295 1/1 Running 1 (4h49m ago) 21h etcd-kmaster 1/1 Running 1 (4h49m ago) 21h kube-apiserver-kmaster 1/1 Running 1 (4h49m ago) 21h kube-controller-manager-kmaster 1/1 Running 1 (4h49m ago) 21h kube-proxy-87qn9 1/1 Running 1 (4h49m ago) 21h kube-proxy-bpqng 1/1 Running 1 (4h49m ago) 21h kube-proxy-x88s7 1/1 Running 1 (4h49m ago) 21h kube-scheduler-kmaster 1/1 Running 1 (4h49m ago) 21h metrics-server-5fc67cc865-5sp2h 1/1 Running 0 8m22s \[root@kmaster \~]# kubectl get node NAME STATUS ROLES AGE VERSION kmaster Ready control-plane 21h v1.26.0 knode1 Ready 21h v1.26.0 knode2 Ready 21h v1.26.0 \[root@kmaster \~]# \[root@kmaster \~]# \[root@kmaster \~]# kubectl get pod No resources found in default namespace. \[root@kmaster \~]# kubectl config get-contexts CURRENT NAME CLUSTER AUTHINFO NAMESPACE

* ```
      kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
  ```

\[root@kmaster \~]# kubectl config set-context --current --namespace kube-system Context "kubernetes-admin@kubernetes" modified. \[root@kmaster \~]# kubectl config get-contexts CURRENT NAME CLUSTER AUTHINFO NAMESPACE

* ```
      kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   kube-system
  ```

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE coredns-5bbd96d687-4zl6m 1/1 Running 1 (4h53m ago) 21h coredns-5bbd96d687-w9295 1/1 Running 1 (4h53m ago) 21h etcd-kmaster 1/1 Running 1 (4h53m ago) 21h kube-apiserver-kmaster 1/1 Running 1 (4h53m ago) 21h kube-controller-manager-kmaster 1/1 Running 1 (4h53m ago) 21h kube-proxy-87qn9 1/1 Running 1 (4h53m ago) 21h kube-proxy-bpqng 1/1 Running 1 (4h53m ago) 21h kube-proxy-x88s7 1/1 Running 1 (4h53m ago) 21h kube-scheduler-kmaster 1/1 Running 1 (4h53m ago) 21h metrics-server-5fc67cc865-5sp2h 1/1 Running 0 12m

\[root@kmaster \~]# mv kubens /bin/ \[root@kmaster \~]# chmod +x /bin/kubens \[root@kmaster \~]# kubens \[root@kmaster \~]# kubens default

## 2 POD管理

### 2.1 pod如何创建

在k8s集群里面，k8s调度的最小单位 pod，pod里面跑容器（containerd）。

如何创建一个pod：1.命令行 2.yaml文件。推荐后者 命令行： \[root@kmaster \~]# kubectl create ns pod323 namespace/pod323 created \[root@kmaster \~]# kubens pod323 Context "kubernetes-admin@kubernetes" modified.

\[root@kmaster \~]# kubectl run pod1 --image nginx pod/pod1 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 0/1 ContainerCreating 0 9s \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 54s \[root@kmaster \~]# \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod1 1/1 Running 0 83s 10.244.195.133 knode1

\[root@kmaster \~]# kubectl describe pod pod1

\[root@kmaster \~]# kubectl run pod5 --image nginx --image-pull-policy IfNotPresent

镜像的下载策略 Always：它每次都会联网检查最新的镜像，不管你本地有没有，都会到互联网上（动作：会有联网检查这个动作） Never：它只会使用本地镜像，从不下载 IfNotPresent：它如果检测本地没有镜像，才会联网下载，如果有，联网检测版本，但是如果联不通外网，直接用本地。

yaml文件 kubectl run pod1 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml -- sleep 3600 > pod2.yaml

\[root@kmaster pod323]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod2 1/1 Running 0 39m 10.244.195.137 knode1 podmeme 2/2 Running 0 46s 10.244.69.204 knode2

一个pod里面启动多个容器 \[root@kmaster \~]# cat pod11.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod11 name: pod11 spec: containers:

* args:
  * sleep
  * "3600" image: nginx imagePullPolicy: IfNotPresent name: r1 resources: {}
* image: nginx imagePullPolicy: IfNotPresent name: r2 resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod1 1/1 Running 0 53m 10.244.195.133 knode1 pod10 1/1 Running 0 9m44s 10.244.69.203 knode2 pod11 2/2 Running 0 90s 10.244.195.141 knode1

\[root@kmaster pod323]# kubectl exec -ti podmeme -- bash Defaulted container "pod11" out of: pod11, pod22 \[root@podmeme /]# touch 123.txt \[root@podmeme /]# ls 123.txt bin dev etc home lib lib64 lost+found media mnt opt proc root run sbin srv sys tmp usr var \[root@podmeme /]# exit exit \[root@kmaster pod323]# kubectl exec -ti podmeme -- bash Defaulted container "pod11" out of: pod11, pod22 \[root@podmeme /]# ls 123.txt bin dev etc home lib lib64 lost+found media mnt opt proc root run sbin srv sys tmp usr var \[root@podmeme /]# exit exit \[root@kmaster pod323]# kubectl exec -ti podmeme -c pod22 -- bash \[root@podmeme /]# ls bin dev etc home lib lib64 lost+found media mnt opt proc root run sbin srv sys tmp usr var

### 2.2 pod生命周期及重启策略

容器运行的是进程，这个进程是由镜像定义好的。mysql--mysqld centos--bash kubectl run pod4 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml -- sh -c 'sleep 10' > pod4.yaml

restartPolicy: Always: 一直，正常退出的，非正常退出的，错误的，都重启 Never: 从未，不管是正常的，还是不正常的，都不重启 OnFailure: 遇到了错误重启，正常退出不重启

比如你定义错了命令，这时候创建该pod，并观察错误过程，会发现它在不断尝试重启。 \[root@kmaster pod323]# cat pod4.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod4 name: pod4 spec: containers:

* args:
  * sh
  * \-c
  * sleaaaep 10 image: nginx imagePullPolicy: IfNotPresent name: pod4 resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster pod323]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod4 0/1 CrashLoopBackOff 4 (24s ago) 107s 10.244.195.139 knode1

Never遇到错误从来不重启 \[root@kmaster pod323]# vim pod4.yaml \[root@kmaster pod323]# cat pod4.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod4 name: pod4 spec: containers:

* args:
  * sh
  * \-c
  * sleaaaep 10 image: nginx imagePullPolicy: IfNotPresent name: pod4 resources: {} dnsPolicy: ClusterFirst restartPolicy: Never status: {}

\[root@kmaster pod323]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod4 0/1 Error 0 37s 10.244.195.140 knode1

Onfailure 遇到（命令）错误才会重启，正常退出是不会重启的。 \[root@kmaster pod323]# vim pod4.yaml \[root@kmaster pod323]# cat pod4.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod4 name: pod4 spec: containers:

* args:
  * sh
  * \-c
  * sleaaaep 10 image: nginx imagePullPolicy: IfNotPresent name: pod4 resources: {} dnsPolicy: ClusterFirst restartPolicy: OnFailure status: {}

\[root@kmaster pod323]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod4 0/1 Error 4 (51s ago) 95s 10.244.195.141 knode1

onfailure如果是正常退出呢？没有遇到错误，不会重启。 \[root@kmaster pod323]# cat pod4.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod4 name: pod4 spec: containers:

* args:
  * sh
  * \-c
  * sleep 10 image: nginx imagePullPolicy: IfNotPresent name: pod4 resources: {} dnsPolicy: ClusterFirst restartPolicy: OnFailure status: {} \[root@kmaster pod323]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES pod4 0/1 Completed 0 96s 10.244.195.142 knode1

下周内容预告： 1.初始化容器 2.静态pod 3.pod调度策略 4.cordon/drain/污点

总结： Always 命令正确，正常退出，会重启吗？会反复重启 Always 命令错误，非正常退出，会重启吗？会反复重启

Never 命令正确，正常退出，会重启吗？不会重启 Never 命令错误，非正常退出，会重启吗？不会重启

OnFailure 命令正确，正常退出，会重启吗？不会重启 OnFailure 命令错误，非正常退出，会重启吗？会反复重启
