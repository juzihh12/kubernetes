# 01 Kubernetes架构及安装

### 先决条件：

### 1.节点网络必须可以连通外网

### 2.虚拟cpu数量至少2个（the number of available CPUs 1 is less than the required 2）

环境规划 kmaster ip : 192.168.100.180 knode1 ip : 192.168.100.181 knode2 ip : 192.168.100.182

IPADDR=192.168.100.181 NETMASK=255.255.255.0 GATEWAY=192.168.100.2 DNS1=192.168.100.2

### 01 Docker 和 Containerd 关系

docker run/start/stop/restart/rm -f

Docker Engine 核心组件 Containerd，后将其开源并捐赠给CNCF(云原生计算基金会)，作为独立的项目运营。 也就是说 Docker Engine 里面是包含 containerd 的，安装 Docker 后，就会有 containerd。 containerd 也可以进行单独安装，无需安装 Docker。

yum install -y containerd

### 02 为什么要说明他们的关系-引入dockershim

kubectl ---> kubelet --(CRI)--> dockershim ---> Docker Engine ---> Containerd 运行时 ---> container... kubectl ---> kubelet --(CRI)--> CRI-dockerd ---> Docker Engine ---> Containerd 运行时 ---> container... kubectl ---> kubelet --(CRI)--> CRI-Containerd ---> Containerd 运行时 ---> container...

早期，Kubernetes 集成了 Docker Engine 这个容器运行时，之后 K8s 为了兼容更多容器运行时（比如 containerd/CRI-O）， 创建了一个CRI（Container Runtime Interface）容器运行时接口标准，制定标准的目的是为了实现编排器（如 Kubernetes）和其他不同的容器运行时之间交互操作。 但是，Docker Engine 本身就是一个完整的技术栈，没有实现（CRI）接口，也不可能为了迎合k8s编排工具指定的 CRI 标准而改变底层架构，于是，K8s引入了一个临时解决方案--dockershim。 dockershim 这个项目是k8s维护的项目，目的是为了解决 Docker 本身无法适配 CRI 而提出的。 也就是通过dockershim，可以让k8s的代理kubelet通过 CRI 来调用 Docker Engine 进行容器的操作。

### 03 那为何在v1.24版本中删除 dockershim

k8s 在 1.7 的版本中就已经将 containerd 和 kubelet 集成使用了。既然最终我们都是要调用 containerd 这个容器运行时。 那干嘛非得多一个dockershim？而且dockershim本身就是作为一个临时过渡的产品来使用的。如果剔除dockershim 转而直接使用 containerd，调用链岂不是更短？ 执行效率岂不是更高？而且维护dockershim已经成为kubernetes维护者的沉重负担。 于是 Kubernetes 官方发布公告，宣布自 v1.20 起放弃对 Docker 的支持，v1.20 的版本中，会收到一个 docker 的弃用警告， 在未来 v1.22 版本之前是不会删除的，这意味着到 2021 年底的 v1.23 版本，还有 1 年的时间来寻找合适的 CRI 运行时来确保顺利的过渡，比如 containerd 和 CRI-O。

大瓜爆出，众说纷纭，其中有一种声音：Docker将被抛弃、Docker或将被淘汰。其实不对！Docker没有被淘汰，也没有被弃用，只是在1.24.0中删除了dockershim而已。

docker pull kubectl run .... v1.26 v1.25.4

对于上层使用者而言，如果没有使用cri-dockerd将Docker Engine 和 k8s 集成，那么docker命令是查不到k8s（crictl）的镜像，反过来也一样。

### 04 命令之间的区别

docker（docker）：docker 命令行工具，无需单独安装。 ctr（containerd）：containerd 命令行工具，无需单独安装，集成containerd，多用于测试或开发。 nerdctl（containerd）：containerd 客户端工具，使用效果与 docker 命令的语法一致。需要单独安装。 crictl（kubernetes）：遵循 CRI 接口规范的一个命令行工具，通常用它来检查和管理​​kubelet​​节点上的容器运行时和镜像。没有tag和push。

### 05 安装

### 操作系统安装、模板制作及搭建请参考：

G003-OS-LIN-RHEL-01 红帽 8.4 安装 （https://blog.51cto.com/cloudcs/5245337） G018-LIN-ASK-SOL-01 制作 Linux 虚拟机模板 （https://blog.51cto.com/cloudcs/5258769） G030-CON-CKA-K8S-01 CentOS Stream 8 搭建 kubernetes 集群 v1.26.0 （https://blog.51cto.com/cloudcs/6042006）

红帽 RHEL 8.0 发布，CentOS 紧随其后 CentOS 8.0 CentOS Stream 版本放在前面 发布； 根据上游stream版本发布 RHEL

Minimal

### 1.装包

yum install -y yum-utils vim bash-completion net-tools wget

### 2.设置/etc/hosts

echo "$hostip $hostname" >> /etc/hosts

echo "192.168.100.100 kmaster" >> /etc/hosts

脚本里面没有写 逻辑判断和传递值，只是命令的集合。

### 3.关闭防火墙和SELinux

systemctl stop firewalld systemctl disable firewalld setenforce 0 sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

### 4.关闭swap

swapoff -a sed -i "s/^.\*swap/#&/g" /etc/fstab

### 5.安装docker-ce

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo echo 'list docker-ce versions' yum list docker-ce --showduplicates | sort -r yum install -y docker-ce 默认安装最新版 yum install -y docker-ce-xx.xx.x docker-ce-cli-xx.xx.x systemctl start docker systemctl enable docker

v1.26 没有dockershim yum install -y containerd

### 6.iptables

cat < /etc/sysctl.d/k8s.conf net.bridge.bridge-nf-call-ip6tables = 1 net.bridge.bridge-nf-call-iptables = 1 net.ipv4.ip\_forward = 1 EOF sysctl -p /etc/sysctl.d/k8s.conf

### 7.设置cgroup(systemd/cgroupfs)

containerd config default > /etc/containerd/config.toml sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/google\_containers/pause#g" /etc/containerd/config.toml sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml systemctl restart containerd

### 8.kubernetes.repo

cat < /etc/yum.repos.d/kubernetes.repo \[kubernetes] name=Kubernetes baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86\_64/ enabled=1 gpgcheck=1 repo\_gpgcheck=1 gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg EOF

### 9.crictl 运行端点

cat < /etc/crictl.yaml runtime-endpoint: unix:///run/containerd/containerd.sock image-endpoint: unix:///run/containerd/containerd.sock timeout: 5 debug: false EOF

crictl images 查镜像直接报错 默认会调用 dockershim docker 1.24 剔除 dockershim

### 10.安装kube工具

yum install -y kubelet-1.26.0 kubeadm-1.26.0 kubectl-1.26.0 --disableexcludes=kubernetes systemctl enable --now kubelet

以上1-10操作在所有节点上操作。

### 11.集群初始化（仅在master节点）

kubeadm init --image-repository registry.aliyuncs.com/google\_containers --kubernetes-version=v1.26.0 --pod-network-cidr=10.244.0.0/16

镜像--创建容器（pod）

### 12.将节点加入集群

参考博客 https://blog.51cto.com/cloudcs/6042006

mkdir -p $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster. Run "kubectl apply -f \[podnetwork].yaml" with one of the options listed at: https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.100.205:6443 --token i10ers.ukzqi8t6y2w9ds1p\
\--discovery-token-ca-cert-hash sha256:81469d1f980f8751fa9feecce02d56f497858ea97c7ec06011143b134237b06c

将最后一段复制到所有的节点上执行，加入集群

### 13.安装calico网络

https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart

参考博客 https://blog.51cto.com/cloudcs/6042006

v1.27.0 版本k8s 通过脚本搭建流程，可以参考以下链接 https://blog.51cto.com/cloudcs/6773818

如果执行custom脚本报错如下： \[root@kmaster \~]# kubectl delete -f custom-resources.yaml apiserver.operator.tigera.io "default" deleted error: resource mapping not found for name: "default" namespace: "" from "custom-resources.yaml": no matches for kind "Installation" in version "operator.tigera.io/v1"

请确认是否没有用create 执行tigera-operator，不可以使用 apply 来执行。 kubectl create -f tigera-operator.yam

CNI Plugin CNI网络插件，Calico通过CNI网络插件与kubelet关联，从而实现Pod网络。 Calico Node Calico节点代理是运行在每个节点上的代理程序，负责管理节点路由信息、策略规则和创建Calico虚拟网络设备。 Calico Controller Calico网络策略控制器。允许创建"NetworkPolicy"资源对象，并根据资源对象里面对网络策略定义，在对应节点主机上创建针对于Pod流出或流入流量的IPtables规则。 Calico Typha（可选的扩展组件） Typha是Calico的一个扩展组件，用于Calico通过Typha直接与Etcd通信，而不是通过kube-apiserver。通常当K8S的规模超过50个节点的时候推荐启用它，以降低kube-apiserver的负载。每个Pod/calico-typha可承载100\~200个Calico节点的连接请求，最多不要超过200个。
