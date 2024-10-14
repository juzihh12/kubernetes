# 11 Authentication

1.远程登录 2.角色及权限 3.图形化管理

## 1 远程登录

### 1.1 kubeconfig

在master上都是以kubeconfig方式登录的。并不是说有一个文件叫kubeconfig 默认使用的配置文件 ls .kube/config 这个配置文件，而这个配置文件是通过这个文件 /etc/kubernetes/admin.conf

如果在node上执行命令，也需要配置该文件，否则报错如下： \[root@knode1 \~]# kubectl get node E0411 16:18:03.370177 34252 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp \[::1]:8080: connect: connection refused E0411 16:18:03.370447 34252 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp \[::1]:8080: connect: connection refused E0411 16:18:03.372113 34252 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp \[::1]:8080: connect: connection refused E0411 16:18:03.374987 34252 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp \[::1]:8080: connect: connection refused E0411 16:18:03.375890 34252 memcache.go:238] couldn't get current server API group list: Get "http://localhost:8080/api?timeout=32s": dial tcp \[::1]:8080: connect: connection refused The connection to the server localhost:8080 was refused - did you specify the right host or port?

当然，可以在master拷贝一份到knode1 \[root@kmaster kubernetes]# pwd /etc/kubernetes \[root@kmaster kubernetes]# scp admin.conf knode1:\~

在node1上执行命令的时候，直接指定该文件运行。 \[root@knode1 \~]# kubectl get node --kubeconfig=admin.conf NAME STATUS ROLES AGE VERSION kmaster Ready control-plane 19d v1.26.0 knode1 Ready 19d v1.26.0 knode2 Ready 19d v1.26.0

如果不想指定配置文件，可以直接配置环境变量。 \[root@knode1 \~]# echo 'export KUBECONFIG=admin.conf' >> /etc/profile \[root@knode1 \~]# source /etc/profile \[root@knode1 \~]# kubectl get no NAME STATUS ROLES AGE VERSION kmaster Ready control-plane 19d v1.26.0 knode1 Ready 19d v1.26.0 knode2 Ready 19d v1.26.0

这样就不需要指定配置文件了。 注意：该admin.conf文件是直接从master上拷贝过来的，所以默认具备admin管理员最大的权限的。 生产环境不建议这样操作，因为权限过大，无法保证安全性。

能否去单独创建用户授权呢？比如tom natasha

#### 1.1.1 生成私钥

\[root@kmaster \~]# mkdir /ca \[root@kmaster \~]# cd /ca/ \[root@kmaster ca]# openssl genrsa -out client.key 2048 Generating RSA private key, 2048 bit long modulus (2 primes) .......................................................+++++ .............+++++ e is 65537 (0x010001)

#### 1.1.2 生成tom用户证书请求文件

\[root@kmaster ca]# openssl req -new -key client.key -subj "/CN=tom" -out client.csr \[root@kmaster ca]# ls client.csr client.key

k8s本身使用的一些ca证书，是在/etc/kubernetes/pki/目录下 \[root@kmaster ca]# ls /etc/kubernetes/pki/ apiserver.crt apiserver.key ca.crt front-proxy-ca.crt front-proxy-client.key apiserver-etcd-client.crt apiserver-kubelet-client.crt ca.key front-proxy-ca.key sa.key apiserver-etcd-client.key apiserver-kubelet-client.key etcd front-proxy-client.crt sa.pub \[root@kmaster ca]#

#### 1.1.3 为tom用户颁发证书

tom如何将请求发给k8s的ca进行证书颁发呢？使用kubernetes自带的ca为tom颁发证书。 \[root@kmaster ca]# openssl x509 -req -in client.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out client.crt -days 3650 Signature ok subject=CN = tom Getting CA Private Key

\[root@kmaster ca]# ls client.crt client.csr client.key

拷贝ca证书到当前目录 \[root@kmaster ca]# cp /etc/kubernetes/pki/ca.crt . \[root@kmaster ca]# ll total 16 -rw-r--r-- 1 root root 1099 Apr 16 10:58 ca.crt -rw-r--r-- 1 root root 985 Apr 11 16:45 client.crt -rw-r--r-- 1 root root 883 Apr 11 16:27 client.csr -rw------- 1 root root 1675 Apr 11 16:24 client.key

#### 1.1.4 创建测试命名空间及pod

\[root@kmaster ca]# kubens hehe Context "kubernetes-admin@kubernetes" modified. Active namespace is "hehe". \[root@kmaster ca]# kubectl get pod No resources found in hehe namespace. \[root@kmaster ca]# kubectl run pod1 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml > pod1.yaml \[root@kmaster ca]# kubectl apply -f pod1.yaml pod/pod1 created \[root@kmaster ca]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 1s

#### 1.1.5 创建角色和授权

\[root@kmaster ca]# kubectl create role r1 --verb get --verb watch --verb list --resource pods -n hehe role.rbac.authorization.k8s.io/r1 created 创建角色，使其对hehe命名空间具有get/watch/list权限 查看： \[root@kmaster ca]# kubectl get role NAME CREATED AT r1 2023-04-16T03:12:58Z 详细查看： \[root@kmaster ca]# kubectl describe role r1 Name: r1 Labels: Annotations: PolicyRule: Resources Non-Resource URLs Resource Names Verbs

***

pods \[] \[] \[get watch list]

#### 1.1.6 将角色绑定给tom用户

\[root@kmaster ca]# kubectl create rolebinding r1binding --role r1 --user tom -n hehe rolebinding.rbac.authorization.k8s.io/r1binding created \[root@kmaster ca]# kubectl get rolebindings.rbac.authorization.k8s.io NAME ROLE AGE r1binding Role/r1 7s \[root@kmaster ca]# kubectl describe rolebindings.rbac.authorization.k8s.io Name: r1binding Labels: Annotations: Role: Kind: Role Name: r1 Subjects: Kind Name Namespace

***

User tom

#### 1.1.7 编辑kuberconfig文件

https://kubernetes.io/zh-cn/docs/concepts/configuration/organize-cluster-access-kubeconfig/

\[root@kmaster ca]# vim kubeabc \[root@kmaster ca]# cat kubeabc apiVersion: v1 kind: Config

clusters:

* cluster: name: dev1

users:

* name: tom

contexts:

* context: name: context1 namespace: hehe current-context: "context1"

设置集群密钥信息（嵌入集群密钥） \[root@kmaster ca]# kubectl config --kubeconfig=tomconfig set-cluster dev1 --server=https://192.168.100.205:6443 --certificate-authority=ca.crt --embed-certs=true Cluster "dev1" set.

设置tom用户密钥信息 \[root@kmaster ca]# kubectl config --kubeconfig=tomconfig set-credentials tom --client-certificate=client.crt --client-key=client.key --embed-certs=true User "tom" set.

设置上下文信息 \[root@kmaster ca]# kubectl config --kubeconfig=tomconfig set-context context1 --cluster=dev1 --namespace=ns825 --user=tom Context "context1" modified.

#### 1.1.8 测试

拷贝至客户端并测试 \[root@kmaster ca]# scp kubeabc knode2:\~ root@knode2's password: kubeabc

\[root@knode2 \~]# kubectl get pod --kubeconfig=kubeabc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 52m

这样，客户端就以tom身份登录到集群，进行管理操作了。 \[root@knode2 \~]# export KUBECONFIG=kubeabc \[root@knode2 \~]# kubectl get pod

### 1.2 静态token方式

https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#static-token-file

\[root@kmaster ca]# vim /etc/kubernetes/manifests/kube-apiserver.yaml

containers:

* command:
  * kube-apiserver
  * \--advertise-address=192.168.100.180
  * \--token-auth-file=/etc/kubernetes/pki/static.csv /etc/kubernetes/pki/static.csv 该文件就是保存用户账号和密码的。建议不要更换目录，因为k8s本身是对该目录有足够权限的。

创建csv文件 当 API 服务器的命令行设置了 --token-auth-file=SOMEFILE 选项时，会从文件中读取持有者令牌。 目前，令牌会长期有效，并且在不重启 API 服务器的情况下无法更改令牌列表。

令牌文件是一个 CSV 文件，包含至少 3 个列：令牌、用户名和用户的 UID。 其余列被视为可选的组名。 如果要设置的组名不止一个，则对应的列必须用双引号括起来，例如：token,user,uid,"group1,group2,group3"

\[root@kmaster \~]# openssl rand -hex 10 6de81897de48a0006ec1 \[root@kmaster \~]# cd /etc/kubernetes/pki/ \[root@kmaster pki]# ls apiserver.crt apiserver-etcd-client.key apiserver-kubelet-client.crt ca.crt etcd front-proxy-ca.key front-proxy-client.key sa.pub apiserver-etcd-client.crt apiserver.key apiserver-kubelet-client.key ca.key front-proxy-ca.crt front-proxy-client.crt sa.key \[root@kmaster pki]# vim abc.csv \[root@kmaster pki]# vim abc.csv \[root@kmaster pki]# cat abc.csv 6de81897de48a0006ec1,tom,3 \[root@kmaster pki]# pwd /etc/kubernetes/pki \[root@kmaster pki]# vim /etc/kubernetes/manifests/kube-apiserver.yaml \[root@kmaster pki]# vim /etc/kubernetes/manifests/kube-apiserver.yaml \[root@kmaster pki]# systemctl restart kubelet.service

\[root@knode2 \~]# kubectl options \[root@knode2 \~]# kubectl --server='https://192.168.100.205:6443' --token='6de81897de48a0006ec1' get pods -n default E0119 11:32:12.213220 7649 memcache.go:238] couldn't get current server API group list: Get "https://192.168.146.139:6443/api?timeout=32s": x509: certificate signed by unknown authority E0119 11:32:12.215085 7649 memcache.go:238] couldn't get current server API group list: Get "https://192.168.146.139:6443/api?timeout=32s": x509: certificate signed by unknown authority E0119 11:32:12.216838 7649 memcache.go:238] couldn't get current server API group list: Get "https://192.168.146.139:6443/api?timeout=32s": x509: certificate signed by unknown authority E0119 11:32:12.218482 7649 memcache.go:238] couldn't get current server API group list: Get "https://192.168.146.139:6443/api?timeout=32s": x509: certificate signed by unknown authority E0119 11:32:12.220062 7649 memcache.go:238] couldn't get current server API group list: Get "https://192.168.146.139:6443/api?timeout=32s": x509: certificate signed by unknown authority Unable to connect to the server: x509: certificate signed by unknown authority

提示x509加密认证

\[root@knode2 \~]# kubectl --server='https://192.168.100.205:6443' --token='87b6d6f693f07cbcb0e2' --insecure-skip-tls-verify=true get pods -n default Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "default"

提示该用户是没有权限的。登录过来没有权限，那如何进行授权呢？

## 2 角色授权

https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/

containers:

* command:
  * kube-apiserver
  * \--token-auth-file=/etc/kubernetes/pki/abc.csv
  * \--advertise-address=192.168.146.139
  * \--allow-privileged=true
  * \--authorization-mode=Node,RBAC

注意这一行 - --authorization-mode=Node,RBAC 除了这两个，还有以下： --authorization-mode=ABAC 基于属性的访问控制（ABAC）模式允许你使用本地文件配置策略。 --authorization-mode=RBAC 基于角色的访问控制（RBAC）模式允许你使用 Kubernetes API 创建和存储策略。把权限放在角色包里，把角色授权用户。 --authorization-mode=Webhook WebHook 是一种 HTTP 回调模式，允许你使用远程 REST 端点管理鉴权。 --authorization-mode=Node 节点鉴权是一种特殊用途的鉴权模式，专门对 kubelet 发出的 API 请求执行鉴权。 --authorization-mode=AlwaysDeny 该标志阻止所有请求。仅将此标志用于测试。 --authorization-mode=AlwaysAllow 此标志允许所有请求。仅在你不需要 API 请求的鉴权时才使用此标志。

我们接下来，把api里面默认用的授权方式改成 AlwaysAllow ，重启api服务，再次尝试tom用户是否可以直接登录。

\[root@kmaster pki]# vim /etc/kubernetes/manifests/kube-apiserver.yaml

containers:

* command:
  * kube-apiserver
  * \--token-auth-file=/etc/kubernetes/pki/abc.csv
  * \--advertise-address=192.168.146.139
  * \--allow-privileged=true #- --authorization-mode=Node,RBAC
  * \--authorization-mode=AlwaysAllow

\[root@kmaster pki]# systemctl restart kubelet.service \[root@kmaster pki]# kubectl get nodes NAME STATUS ROLES AGE VERSION kmaster Ready control-plane 53d v1.26.0 knode1 Ready 53d v1.26.0 knode2 Ready 53d v1.26.0

再次在node2上执行命令查询 \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true get pods -n default NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 19m

这时候会发现，即使，集群没有针对tom进行授权，他也可以直接访问到该pod。 \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true get pods -n kube-system NAME READY STATUS RESTARTS AGE coredns-5bbd96d687-ddrf4 1/1 Running 1 (33m ago) 53d coredns-5bbd96d687-jgfbv 1/1 Running 1 (33m ago) 53d etcd-kmaster 1/1 Running 1 (33m ago) 53d kube-apiserver-kmaster 0/1 Running 1 (31s ago) 5m1s kube-controller-manager-kmaster 1/1 Running 3 (49s ago) 53d kube-proxy-7k2sk 1/1 Running 1 (33m ago) 53d kube-proxy-dtplw 1/1 Running 1 (33m ago) 53d kube-proxy-jcwrd 1/1 Running 1 (33m ago) 53d kube-scheduler-kmaster 1/1 Running 3 (49s ago) 53d

反之，如果api参数被改为 AlwaysDeny，那么不管该用户是否授权，一律不可以访问。当然，除了master可以，因为默认master是通过本地主机进行访问的。不受该参数影响。

### 2.1 role 与 rolebinding

是基于命名空间来授权的

get

list ——————> 全部给到 role1 角色（基于命名空间） ---> 给到tom用户 abc create

查看集群中的角色 \[root@kmaster pki]# kubectl get clusterrole

查看当前集群中管理员角色，看他的权限。 \[root@kmaster pki]# kubectl describe clusterrole admin

角色又分为两种：role和clusterrole role：是基于命名空间的ns1，通过ns进行隔离，只会在当前ns1里面生效。 把角色绑定给用户abc，叫rolebinding。那么abc这个用户他在ns1里面只会具备role角色定义的权限。

实验： 创建一个ns，abc

\[root@kmaster pki]# kubectl create ns abc namespace/abc created \[root@kmaster pki]# kubectl get ns NAME STATUS AGE abc Active 19s calico-apiserver Active 53d calico-system Active 53d default Active 53d kube-node-lease Active 53d kube-public Active 53d kube-system Active 53d tigera-operator Active 53d

\[root@kmaster \~]# kubectl apply -f pod1.yaml -n abc pod/pod1 created \[root@kmaster \~]# kubectl get pod -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 6s

默认情况下，node2 tom用户是无法访问abc里面的pod的 \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true get pods -n abc Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "abc"

默认abc里面也没有任何角色 \[root@kmaster \~]# kubectl get role -n abc No resources found in abc namespace.

创建一个角色 \[root@kmaster \~]# kubectl create role pod-reader01 --verb get,list,watch,delete --resource pods --dry-run=client -o yaml > role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * "" resources:
  * pods verbs:
  * get
  * list
  * watch

\[root@kmaster \~]# kubectl apply -f role1.yaml -n abc role.rbac.authorization.k8s.io/pod-reader01 created \[root@kmaster \~]# kubectl get role -n abc NAME CREATED AT pod-reader01 2023-04-19T04:12:51Z

\[root@kmaster \~]# kubectl describe role pod-reader01 -n abc Name: pod-reader01 Labels: Annotations: PolicyRule: Resources Non-Resource URLs Resource Names Verbs

***

pods \[] \[] \[get list watch]

这时候该角色仅仅针对pods类型具有 get list 和watch操作。这时候node2的tom可以访问吗？不可以，还没有绑定。

绑定tom用户 \[root@kmaster \~]# kubectl get rolebindings.rbac.authorization.k8s.io -n abc No resources found in abc namespace.

注意带上 --user 和 --token ，因为是根据 token 认证的，所以必须带上，否则后面无法进行创建和删除 \[root@kmaster \~]# kubectl create rolebinding rb1-tom --role pod-reader01 --user tom --dry-run=client -o yaml > rb1.yaml \[root@kmaster \~]# cat rb1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: RoleBinding metadata: creationTimestamp: null name: rb1-tom roleRef: apiGroup: rbac.authorization.k8s.io kind: Role name: pod-reader01 subjects:

* apiGroup: rbac.authorization.k8s.io kind: User name: tom

\[root@kmaster \~]# kubectl apply -f rb1.yaml -n abc rolebinding.rbac.authorization.k8s.io/rb1-tom created \[root@kmaster \~]# kubectl get rolebindings.rbac.authorization.k8s.io -n abc NAME ROLE AGE rb1-tom Role/pod-reader01 3s

tom查看pod \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true get pods -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 12m

可以查看其他ns资源吗？不可以。因为角色是在abc命名空间设置的权限，而不是在kube-system里面。 \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true get pods -n kube-system Error from server (Forbidden): pods is forbidden: User "tom" cannot list resource "pods" in API group "" in the namespace "kube-system"

tom可以删除吗？不可以，因为授权的时候，没有授权delete \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='6de81897de48a0006ec1' --insecure-skip-tls-verify=true delete pods/pod1 -n abc Error from server (Forbidden): pods "pod1" is forbidden: User "tom" cannot delete resource "pods" in API group "" in the namespace "abc"

现在增加一个delete权限。 \[root@kmaster \~]# vim role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * "" resources:
  * pods verbs:
  * get
  * list
  * watch
  * delete \[root@kmaster \~]# kubectl apply -f role1.yaml role.rbac.authorization.k8s.io/pod-reader01 created

这里有问题了，给delete，create，依然无法创建或删除？？？因为上面创建binding的时候，没有添加token参数。 现在删除binding。重新创建 \[root@kmaster \~]# kubectl create rolebinding rb1-tom --role pod-reader01 --user='tom' --token='605b364ac03b532cbd44' --dry-run=client -o yaml > rb1.yaml \[root@kmaster \~]# kubectl apply -f rb1.yaml rolebinding.rbac.authorization.k8s.io/rb1-tom configured \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * "" resources:
  * pods verbs:
  * get
  * list
  * watch
  * delete
  * create \[root@kmaster \~]# kubectl apply -f role1.yaml role.rbac.authorization.k8s.io/pod-reader01 configured

尝试删除 \[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='605b364ac03b532cbd44' --insecure-skip-tls-verify=true delete pod pod1 -n abc pod "pod1" deleted

尝试创建 \[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='605b364ac03b532cbd44' --insecure-skip-tls-verify=true get pods -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 2m51s \[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='605b364ac03b532cbd44' --insecure-skip-tls-verify=true get pods -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 2m52s

同样，也可以针对其他对象，比如deployment操作。

\[root@kmaster \~]# vim role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * "" resources:
  * pods
  * deployments verbs:
  * get
  * list
  * watch
  * delete
  * create

\[root@kmaster \~]# kubectl apply -f role1.yaml -n abc role.rbac.authorization.k8s.io/pod-reader01 configured \[root@kmaster \~]# kubectl get deployments.apps No resources found in default namespace.

\[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true get deploy -n abc Error from server (Forbidden): deployments.apps is forbidden: User "henry" cannot list resource "deployments" in API group "apps" in the namespace "abc"

你会发现，查询deploy是不可以的。因为deploy

pod--apiversion v1 services--apiversion v1 deployment--apps/v1 不同的这些资源类型，他们的apiversion是不一样的。查询集群中所有的apiversion \[root@kmaster \~]# kubectl api-versions \[root@kmaster \~]# kubectl api-resources -o wide 这里面version分为两种，一种带斜杠，一种不带斜杠。前者是两级结构，后者一级结构。

deployment对应的apiversion为两级结构，所以，要写两级，要把父级写入到apigroup里面，才能生效。而pod和svc都是一级结构，所以父级apigroup为空。

所以改一下： \[root@kmaster \~]# vim role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * ""
  * apps resources:
  * pods
  * deployments verbs:
  * get
  * list
  * watch
  * delete
  * create

再次apply，node2上尝试查询deployment。 \[root@kmaster \~]# kubectl apply -f role1.yaml -n abc role.rbac.authorization.k8s.io/pod-reader01 configured

\[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true get deploy -n abc No resources found in abc namespace.

deploy可以查看了，那可以创建吗？ \[root@kmaster \~]# kubectl create deployment web1 --image nginx --dry-run=client -o yaml > web1.yaml

\[root@knode2 \~]# cat web1.yaml apiVersion: apps/v1 kind: Deployment metadata: creationTimestamp: null labels: app: web1 name: web1 spec: replicas: 1 selector: matchLabels: app: web1 strategy: {} template: metadata: creationTimestamp: null labels: app: web1 spec: containers: - image: nginx imagePullPolicy: IfNotPresent name: nginx resources: {} status: {}

\[root@kmaster ~~]# scp web1.yaml 192.168.100.182:~~ root@192.168.100.182's password:

\[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true apply -f web1.yaml -n abc deployment.apps/web1 created \[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true get deploy -n abcNAME READY UP-TO-DATE AVAILABLE AGE web1 1/1 1 1 7s \[root@knode2 \~]# kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true get pod -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 40m web1-849556688-z99ks 1/1 Running 0 10s

再试下可以scale扩展吗？ \[root@knode2 \~]# alias kubectl='kubectl --server='https://192.168.100.180:6443' --token='7dd2cbf6e642e5cbd712' --insecure-skip-tls-verify=true' \[root@knode2 \~]# kubectl get pod -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 48m web1-849556688-z99ks 1/1 Running 0 8m13s

\[root@knode2 \~]# kubectl get deploy -n abc NAME READY UP-TO-DATE AVAILABLE AGE web1 1/1 1 1 8m48s

\[root@knode2 \~]# kubectl scale deploy web1 --replicas=3 -n abc Error from server (Forbidden): deployments.apps "web1" is forbidden: User "henry" cannot patch resource "deployments/scale" in API group "apps" in the namespace "abc"

这时候提示resources是无法修补资源的。把 deployments/scale 加入到 资源里面，重新应用，再次尝试下。 \[root@kmaster \~]# vim role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * ""
  * apps resources:
  * pods
  * deployments
  * deployments/scale

\[root@kmaster \~]# kubectl apply -f role1.yaml -n abc role.rbac.authorization.k8s.io/pod-reader01 configured \[root@kmaster \~]#

\[root@knode2 \~]# kubectl scale deploy web1 --replicas=3 -n abc Error from server (Forbidden): deployments.apps "web1" is forbidden: User "henry" cannot patch resource "deployments/scale" in API group "apps" in the namespace "abc"

这时候依然不行。为什么？因为role脚本里面没有给scale这个权限。所以当修改副本数的时候，实际上是更新update权限还有patch补丁权限。 \[root@kmaster \~]# vim role1.yaml \[root@kmaster \~]# cat role1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: Role metadata: creationTimestamp: null name: pod-reader01 rules:

* apiGroups:
  * ""
  * apps resources:
  * pods
  * deployments
  * deployments/scale verbs:
  * get
  * list
  * watch
  * delete
  * create
  * patch
  * update

\[root@kmaster \~]# kubectl apply -f role1.yaml -n abc role.rbac.authorization.k8s.io/pod-reader01 configured

再次执行scale \[root@knode2 \~]# kubectl scale deploy web1 --replicas=3 -n abc deployment.apps/web1 scaled \[root@knode2 \~]# kubectl get pod -n abc NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 56m web1-849556688-8xh5h 1/1 Running 0 16s web1-849556688-wzjfw 1/1 Running 0 16s web1-849556688-z99ks 1/1 Running 0 16m

发现这次可以了。这时候会考虑，如果根据的不同的对象，设置不同的权限。如何做？例如：

rules:

* apiGroups: \[""] resources: \["pods"] verbs: \["get", "list", "watch"]

rules:

* apiGroups: \["apps"] resources: \["deployments"] verbs: \["get", "list", "watch", "create", "update", "patch", "delete"]

以上的作用域是在ns级别。并不是整个集群的。

### 2.2 clusterrole 与 clusterrolebinding

clusterrole对于role来说，是全局的，整个集群命名空间都生效。且clusterrole可以进行clusterrolebinding，也可以进行rolebinding，通过 --namespace 来指定命名空间。比如 \[root@kmaster \~]# kubectl create rolebinding multins --clusterrole pod-reader --user henry --token='46c6cc3b55f372736657' --namespace=default --namespace=kube-system

是基于所有命名空间来授权的

get

list ——————> 全部给到 role1 角色（基于集群所有命名空间） ---> 给到tom用户

create

整个语法和刚才role是类似的。

创建角色 \[root@kmaster \~]# kubectl create clusterrole pod-reader --verb=get,list,watch --resource=pods --dry-run=client -o yaml > clusterrole1.yaml \[root@kmaster \~]# cat clusterrole1.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: ClusterRole metadata: creationTimestamp: null name: pod-reader rules:

* apiGroups:
  * "" resources:
  * pods verbs:
  * get
  * list
  * watch

\[root@kmaster \~]# kubectl apply -f clusterrole1.yaml clusterrole.rbac.authorization.k8s.io/pod-reader created \[root@kmaster \~]# kubectl get clusterrole |grep pod pod-reader 2022-04-20T08:52:22Z system:controller:horizontal-pod-autoscaler 2022-03-22T14:03:55Z system:controller:pod-garbage-collector 2022-03-22T14:03:55Z

查看clusterrole的属性 \[root@kmaster \~]# kubectl describe clusterrole pod-reader Name: pod-reader Labels: Annotations: PolicyRule: Resources Non-Resource URLs Resource Names Verbs

***

pods \[] \[] \[get list watch]

创建binding \[root@kmaster \~]# kubectl create clusterrolebinding cluster-pods --clusterrole pod-reader --user jerry --token='799144ecf0bd002709fb' --dry-run=client -o yaml > rbcluster.yaml \[root@kmaster \~]# cat rbcluster.yaml apiVersion: rbac.authorization.k8s.io/v1 kind: ClusterRoleBinding metadata: creationTimestamp: null name: cluster-pods roleRef: apiGroup: rbac.authorization.k8s.io kind: ClusterRole name: pod-reader subjects:

* apiGroup: rbac.authorization.k8s.io kind: User name: henry

\[root@kmaster \~]# kubectl apply -f rbcluster.yaml clusterrolebinding.rbac.authorization.k8s.io/cluster-pods created

\[root@kmaster \~]# kubectl get clusterrolebindings.rbac.authorization.k8s.io | grep cluster cluster-admin ClusterRole/cluster-admin 28d cluster-pods ClusterRole/pod-reader

\[root@kmaster \~]# kubectl describe clusterrolebindings.rbac.authorization.k8s.io cluster-pods Name: cluster-pods Labels: Annotations: Role: Kind: ClusterRole Name: pod-reader Subjects: Kind Name Namespace

***

User henry

这时候，直接通过henry用户来查询default命名空间。 \[root@knode2 \~]# kubectl --server='https://192.168.146.139:6443' --token='46c6cc3b55f372736657' --insecure-skip-tls-verify=true get pod -n default No resources found in default namespace.

\[root@knode2 \~]# alias kubectl='kubectl --server='https://192.168.146.139:6443' --token='46c6cc3b55f372736657' --insecure-skip-tls-verify=true' \[root@knode2 \~]# kubectl get pod -n default No resources found in default namespace. \[root@knode2 \~]# kubectl get pod -n kube-system NAME READY STATUS RESTARTS AGE coredns-5bbd96d687-ddrf4 1/1 Running 1 (17m ago) 55d coredns-5bbd96d687-jgfbv 1/1 Running 1 (17m ago) 55d etcd-kmaster 1/1 Running 1 (17m ago) 55d kube-apiserver-kmaster 1/1 Running 0 4m55s kube-controller-manager-kmaster 1/1 Running 2 (4m57s ago) 55d kube-proxy-7k2sk 1/1 Running 1 (17m ago) 55d kube-proxy-dtplw 1/1 Running 1 (17m ago) 55d kube-proxy-jcwrd 1/1 Running 1 (17m ago) 55d kube-scheduler-kmaster 1/1 Running 2 (4m57s ago) 55d

其他角色权限和role是一样的。

### 2.3 SA serviceaccount

更新：从1.24版本开始，创建sa不再自动生成secrets https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.24.md#urgent-upgrade-notes https://kubernetes.io/docs/concepts/configuration/secret/#service-account-token-secrets

RBAC分为2种账户：

user account 用户账号（UA）：用于远程登陆系统，也就是我们在刚刚实验所使用的账号 service account 服务账户（SA）：Service account是为了方便Pod里面的进程（而非人工）调用Kubernetes API或其他外部服务而设计的。例如使用web来管理k8s环境时，需要调用相关的程序（pod中）有对K8s管理，当时此程序需要有相关的权限，这时候就可以创建一个SA，赋予相关的权限，并且与管理K8s的环境程序进行绑定，那么这个程序就有了SA的对应权限，可以对K8s进行管理。

在k8s集中群，用户分为 useraccount和serviceaccount，前者用于做登录限制授权使用，后者用于给应用程序授权，不会让用户直接登录。 k8s集群中很多组件都是基于pod来运行的。比如，需要某个pod里面的应用程序来监控或操作集群资源，自动扩展、缩容，部署新的pod。 但是pod里面运行的是某个进程，该进程要去管理集群里面的资源，是否有权限吗？ 这个时候，就需要通过授权指定pod里面的进程，要以哪个sa（服务账号）来运行。该sa是否有足够的权限，如果没有，需要对sa进行授权

举例：创建一个pod 假设该pod里面的进程未来要管理集群资源。 \[root@kmaster \~]# cat web1.yaml apiVersion: apps/v1 kind: Deployment metadata: creationTimestamp: null labels: app: web1 name: web1 spec: replicas: 2 selector: matchLabels: app: web1 strategy: {} template: metadata: creationTimestamp: null labels: app: web1 spec: containers: - image: nginx imagePullPolicy: IfNotPresent name: nginx resources: {} status: {}

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 31m web1-849556688-j7wpx 1/1 Running 0 44m web1-849556688-jgwmj 1/1 Running 0 44m

\[root@kmaster \~]# kubectl get sa NAME SECRETS AGE default 0 55d \[root@kmaster \~]# kubectl create sa sa1 serviceaccount/sa1 created \[root@kmaster \~]# kubectl get sa NAME SECRETS AGE default 0 55d sa1 0 3s

这是的sa1是没有任何权限的。将集群管理员角色授权binding给sa1

\[root@kmaster \~]# kubectl create clusterrolebinding rbsa1 --clusterrole admin --serviceaccount default:sa1 clusterrolebinding.rbac.authorization.k8s.io/rbsa1 created

将服务账号，给到deploy pod（里面的进程）。查看下当前deploy里面的内容： \[root@kmaster \~]# kubectl edit deployments.apps web1

securityContext: {}

把sa1账号绑定给deploy \[root@kmaster \~]# kubectl set serviceaccount deploy web1 sa1 deployment.apps/web1 serviceaccount updated 再次查看 \[root@kmaster \~]# kubectl edit deployments.apps web1

```
  securityContext: {}
  serviceAccount: sa1
  serviceAccountName: sa1
```

这个时候，deploy里面的pod就具备了admin的权限了。

查询sa和关联的角色 kubectl get rolebindings,clusterrolebindings\
\--all-namespaces\
\-o custom-columns='KIND:kind,NAMESPACE:metadata.namespace,NAME:metadata.name,SERVICE\_ACCOUNTS:subjects\[?(@.kind=="ServiceAccount")].name'

## 3 图形化管理

dashboard：官方图形化管理工具 https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/

创建SA

\[root@kmaster \~]# kubectl create sa sa2 -n kube-system serviceaccount/sa2 created \[root@kmaster \~]# kubectl get sa -n kube-system

\[root@kmaster \~]# kubectl create clusterrolebinding rbsa2 --clusterrole admin --serviceaccount kube-system:sa2 -n kube-system clusterrolebinding.rbac.authorization.k8s.io/rbsa2 created

\[root@kmaster \~]# kubectl get sa sa2 -n kube-system NAME SECRETS AGE sa2 0 3m36s

创建secrets \[root@kmaster \~]# vim sa2-secret.yaml \[root@kmaster \~]# cat sa2-secret.yaml apiVersion: v1 kind: Secret metadata: name: secret-sa-sample annotations: kubernetes.io/service-account.name: "sa2" type: kubernetes.io/service-account-token

\[root@kmaster \~]# kubectl apply -f sa2-secret.yaml -n kube-system secret/secret-sa-sample created

为了方便操作，切换到kube-system \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE secret-sa-sample kubernetes.io/service-account-token 3 24s \[root@kmaster \~]# kubectl describe secrets secret-sa-sample Name: secret-sa-sample Namespace: kube-system Labels: Annotations: kubernetes.io/service-account.name: sa2 kubernetes.io/service-account.uid: c67402c3-8624-4627-961b-8a8a4fe9f2f6

Type: kubernetes.io/service-account-token

## Data

namespace: 11 bytes token: eyJhbGciOiJSUzI1NiIsImtpZCI6IkF4WWpqdzhfNDB6UWVjNXZRTzA0cEZsWGdJT2tQTl9Bc05qZF9PTWQwakUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJzZWNyZXQtc2Etc2FtcGxlIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6InNhMiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImM2NzQwMmMzLTg2MjQtNDYyNy05NjFiLThhOGE0ZmU5ZjJmNiIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTpzYTIifQ.5cgYpF4Et6RdaVLCalGhozQRAVWi6q6TcwTDD5e2pjdu6YNxiUd\_GV0dqkqMmy5AGxcZziucRzhBa3Jo48YMKnZ12lG3MInN8BiUJxeYLfYgSjf-EnnbX9n6EfFbYNjE9E2s6ZbfEJvzmo7eHKHPc6tDQ7WTcKU3t1bp89Umn33s8iToLWiHbCPx\_oXbK3Bp7-CxgwqXEy-BDRjynTJoLuYPIK\_VNtMR3Mj7LqktxtbjPLy5cIP6XYPapi4fe2tQPMpWrVRoN27j2HzhI13B9lofO2DnwZ21J77j16vvef4tqIMB1\_zdeenPBTa1p7fwTP8geVxJPPrNkWEeuHx7IA ca.crt: 1099 bytes

这个token就是admin角色的值。记好。 开始安装dashboard

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

\[root@kmaster \~]# vim recommended.yaml \[root@kmaster \~]# grep image recommended.yaml image: kubernetesui/dashboard:v2.7.0 imagePullPolicy: Always image: kubernetesui/metrics-scraper:v1.0.8

更换成阿里云的镜像地址

registry.cn-hangzhou.aliyuncs.com/cloudcs/dashboard:v2.7.0 registry.cn-hangzhou.aliyuncs.com/cloudcs/metrics-scraper:v1.0.8

\[root@kmaster \~]# grep image recommended.yaml image: registry.cn-hangzhou.aliyuncs.com/cloudcs/dashboard:v2.7.0 imagePullPolicy: IfNotPresent image: registry.cn-hangzhou.aliyuncs.com/cloudcs/metrics-scraper:v1.0.8

\[root@kmaster \~]# kubectl apply -f recommended.yaml namespace/kubernetes-dashboard created serviceaccount/kubernetes-dashboard created service/kubernetes-dashboard created secret/kubernetes-dashboard-certs created secret/kubernetes-dashboard-csrf created secret/kubernetes-dashboard-key-holder created configmap/kubernetes-dashboard-settings created role.rbac.authorization.k8s.io/kubernetes-dashboard created clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created deployment.apps/kubernetes-dashboard created service/dashboard-metrics-scraper created deployment.apps/dashboard-metrics-scraper created

\[root@kmaster \~]# kubectl get ns NAME STATUS AGE calico-apiserver Active 55d calico-system Active 55d default Active 55d kube-node-lease Active 55d kube-public Active 55d kube-system Active 55d kubernetes-dashboard Active 35s tigera-operator Active 55d \[root@kmaster \~]# kubectl get pods -n kubernetes-dashboard NAME READY STATUS RESTARTS AGE dashboard-metrics-scraper-67f9cdfb55-pzqj5 1/1 Running 0 60s kubernetes-dashboard-67d48649d5-sbbql 1/1 Running 0 60s \[root@kmaster \~]# kubectl get svc -n kubernetes-dashboard NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE dashboard-metrics-scraper ClusterIP 10.107.56.187 8000/TCP 74s kubernetes-dashboard ClusterIP 10.111.104.11 443/TCP 74s

\[root@kmaster \~]# kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard service/kubernetes-dashboard edited

sessionAffinity: None type: NodePort

\[root@kmaster \~]# kubectl get svc -n kubernetes-dashboard NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE dashboard-metrics-scraper ClusterIP 10.107.56.187 8000/TCP 3m18s kubernetes-dashboard NodePort 10.111.104.11 443:31485/TCP 3m18s

访问集群master 带上端口号（注意https） https://192.168.146.139:31485/#/login

可以选择config方式，也可以token方式登录（上面生成的）。 关于token如何创建，也可以参考官方链接： https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md

类似于其他的三方工具： https://www.rancher.com/ https://github.com/kubesphere/kubesphere/blob/master/README\_zh.md
