# 05 Kubernetes 存储管理

存储volume docker默认情况下，数据保存在容器层，一旦删除容器，数据也随之删除。 在k8s环境里，pod运行容器，之前也没有指定存储，当删除pod的时候，之前在pod里面写入的数据就没有了。 如何需要数据永久存储怎么办？三个方面：1.本地存储 2.网络存储 3.持久化存储(重点) persistent volume （PV） PVC claim

1.本地存储 emptyDir hostPath

1.1 emptyDir 对于emptyDir来说，会在pod所在的物理机上生成一个随机目录。pod的容器会挂载到这个随机目录上。当pod容器删除后，随机目录也会随之删除。适用于多个容器临时共享数据。

\[root@kmaster \~]# kubectl create ns vol namespace/vol created \[root@kmaster \~]# kubens vol Context "kubernetes-admin@kubernetes" modified. Active namespace is "vol". \[root@kmaster \~]# kubens

\[root@kmaster \~]# kubectl run podvol1 --image nginx --dry-run=client -o yaml > podvol1.yaml \[root@kmaster \~]# cat podvol1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: podvol1 name: podvol1 spec: containers:

* image: nginx name: podvol1 resources: {} dnsPolicy: ClusterFirst restartPolicy: Always status: {} \[root@kmaster \~]# vim podvol1.yaml \[root@kmaster \~]# cat podvol1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: podvol1 name: podvol1 spec: volumes:
* name: v1 emptyDir: {}
* name: v2 emptyDir: {} containers:
* image: nginx name: podvol1 imagePullPolicy: IfNotPresent resources: {} volumeMounts:
  * name: v1 mountPath: /abc1 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

在物理机上随机定义一个卷 v1/v2，把物理机的卷挂载到容器里面的/abc1目录里。

\[root@kmaster \~]# kubectl apply -f podvol1.yaml pod/podvol1 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE podvol1 1/1 Running 0 2s

\[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES podvol1 1/1 Running 0 17s 10.244.69.207 knode2

在pod里面abc1已经有了 \[root@kmaster \~]# kubectl exec -ti podvol1 -- bash root@podvol1:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.7G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup /dev/mapper/cs-root xfs 62G 4.7G 57G 8% /abc1 shm tmpfs 64M 0 64M 0% /dev/shm tmpfs tmpfs 7.7G 12K 7.7G 1% /run/secrets/kubernetes.io/serviceaccount tmpfs tmpfs 3.9G 0 3.9G 0% /proc/acpi tmpfs tmpfs 3.9G 0 3.9G 0% /proc/scsi tmpfs tmpfs 3.9G 0 3.9G 0% /sys/firmware

那么abc1到底挂载到物理机的哪个目录上了？查询knode2容器。 \[root@knode2 \~]# crictl ps CONTAINER IMAGE CREATED STATE NAME ATTEMPT POD ID POD 060155d83ec2a a99a39d070bfd 2 minutes ago Running podvol1 0 2112584fe3399 podvol1 fc963212f6dbc 08616d26b8e74 4 days ago Running calico-node 0 ac21f0ebfa5c6 calico-node-mshzf fbd066f5b1dd1 2c2bc18642790 4 days ago Running kube-proxy 0 31b7f9bd4e741 kube-proxy-v5nn6

\[root@knode2 ~~]# crictl inspect 060155d83ec2a "mounts": \[ { "containerPath": "/abc1", "hostPath": "/var/lib/kubelet/pods/54b39465-9bcc-46d0-b55b-a897da005897/volumes/kubernetes.io~~empty-dir/v1", "propagation": "PROPAGATION\_PRIVATE", "readonly": false, "selinuxRelabel": false },

进去查看 \[root@knode2 ~~]# ls /var/lib/kubelet/pods/54b39465-9bcc-46d0-b55b-a897da005897/volumes/kubernetes.io~~empty-dir/v1

然后在pod里面写入数据 root@podvol1:/# touch /abc1/abc.txt root@podvol1:/# exit exit

查看目录 \[root@knode2 ~~]# ls /var/lib/kubelet/pods/54b39465-9bcc-46d0-b55b-a897da005897/volumes/kubernetes.io~~empty-dir/v1 abc.txt

因为存储指定的是本地存储emptyDir，所以是临时的，因此pod删除后，该目录也会随之删除。 \[root@kmaster \~]# kubectl delete -f podvol1.yaml pod "podvol1" deleted

\[root@knode2 ~~]# ls /var/lib/kubelet/pods/54b39465-9bcc-46d0-b55b-a897da005897/volumes/kubernetes.io~~empty-dir/v1 ls: cannot access '/var/lib/kubelet/pods/54b39465-9bcc-46d0-b55b-a897da005897/volumes/kubernetes.io\~empty-dir/v1': No such file or directory

***

一个pod两个容器共享目录

\[root@kmaster \~]# kubectl delete -f podvol1.yaml pod "podvol1" deleted \[root@kmaster \~]# vim podvol1.yaml \[root@kmaster \~]# cat podvol1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: podvol1 name: podvol1 spec: volumes:

* name: v1 emptyDir: {}
* name: v2 emptyDir: {} containers:
* image: nginx name: vol1 imagePullPolicy: IfNotPresent resources: {} volumeMounts:
  * name: v1 mountPath: /vol1
* image: nginx name: vol2 imagePullPolicy: IfNotPresent command: \["sh","-c","sleep 10000"] volumeMounts:
  * name: v1 mountPath: /vol2 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f podvol1.yaml pod/podvol1 created \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES podvol1 2/2 Running 0 6s 10.244.69.208 knode2 \[root@kmaster \~]# kubectl exec -ti podvol1 -c vol1 -- bash root@podvol1:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.7G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup /dev/mapper/cs-root xfs 62G 4.7G 57G 8% /vol1 shm tmpfs 64M 0 64M 0% /dev/shm tmpfs tmpfs 7.7G 12K 7.7G 1% /run/secrets/kubernetes.io/serviceaccount tmpfs tmpfs 3.9G 0 3.9G 0% /proc/acpi tmpfs tmpfs 3.9G 0 3.9G 0% /proc/scsi tmpfs tmpfs 3.9G 0 3.9G 0% /sys/firmware

在容器vol1里面写数据 root@podvol1:/# ls /vol1 root@podvol1:/# touch /vol1/memeda root@podvol1:/#

容器vol2里面直接可以看到 \[root@kmaster \~]# kubectl exec -ti podvol1 -c vol2 -- bash root@podvol1:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.7G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup /dev/mapper/cs-root xfs 62G 4.7G 57G 8% /vol2 shm tmpfs 64M 0 64M 0% /dev/shm tmpfs tmpfs 7.7G 12K 7.7G 1% /run/secrets/kubernetes.io/serviceaccount tmpfs tmpfs 3.9G 0 3.9G 0% /proc/acpi tmpfs tmpfs 3.9G 0 3.9G 0% /proc/scsi tmpfs tmpfs 3.9G 0 3.9G 0% /sys/firmware root@podvol1:/# ls /vol2/ memeda

1.2 hostPath 挂载卷的方式是一样的，只不过不是随机的，是我们自定义的目录，把自定义的目录挂载到容器里面。 \[root@kmaster \~]# kubectl delete -f podvol1.yaml pod "podvol1" deleted

\[root@kmaster \~]# vim podvol1.yaml \[root@kmaster \~]# cat podvol1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: podvol1 name: podvol1 spec: volumes:

* name: v1 emptyDir: {}
* name: v2 hostPath: path: /data containers:
* image: nginx name: vol1 imagePullPolicy: IfNotPresent resources: {} volumeMounts:
  * name: v2 mountPath: /vol1 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f podvol1.yaml pod/podvol1 created \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES podvol1 1/1 Running 0 5s 10.244.69.209 knode2

尝试在pod里面写数据 \[root@kmaster \~]# kubectl exec -ti podvol1 -- bash root@podvol1:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.7G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup /dev/mapper/cs-root xfs 62G 4.7G 57G 8% /vol1 shm tmpfs 64M 0 64M 0% /dev/shm tmpfs tmpfs 7.7G 12K 7.7G 1% /run/secrets/kubernetes.io/serviceaccount tmpfs tmpfs 3.9G 0 3.9G 0% /proc/acpi tmpfs tmpfs 3.9G 0 3.9G 0% /proc/scsi tmpfs tmpfs 3.9G 0 3.9G 0% /sys/firmware root@podvol1:/# touch /vol1/abc.txt root@podvol1:/# exit exit

查看knode2对应的路径 \[root@knode2 \~]# ls /data/ abc.txt

删除pod \[root@kmaster \~]# kubectl delete -f podvol1.yaml pod "podvol1" deleted

knode2对用的路径文件依然存在 \[root@knode2 \~]# ls /data/ abc.txt

这样自定义的目录，可以永久保留，好似没有问题。 如果上面的pod是在knode2上运行的，目录也是在knode2上的永久目录，但是未来该pod万一调度到了knode1上面了，就找不到数据了。保存的记录就没有了。没有办法各个节点之间进行同步。 如何解决这个问题呢？

2.网络存储 2.1 NFS Network FileSystem 网络存储支持很多种类型 nfs/ceph/iscsi等都可以作为后端存储来使用。举例NFS。 克隆一台NFS服务器。 yum install -y yum-utils vim bash-completion net-tools wget nfs-utils \[root@nfs \~]# mkdir /nfsdata \[root@nfs \~]# systemctl start nfs-server.service \[root@nfs \~]# systemctl enable nfs-server.service Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service.

\[root@nfs \~]# systemctl stop firewalld.service \[root@nfs \~]# systemctl disable firewalld.service Removed /etc/systemd/system/multi-user.target.wants/firewalld.service. Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

\[root@nfs \~]# setenforce 0 \[root@nfs \~]# vim /etc/selinux/config

\[root@nfs \~]# vim /etc/exports \[root@nfs \~]# cat /etc/exports /nfsdata \*(rw,async,no\_root\_squash)

exportfs -arv //不用重启nfs服务，配置文件就会生效 no\_root\_squash：登入 NFS 主机使用分享目录的使用者，如果是 root 的话，那么对于这个分享的目录来说，他就具有 root 的权限！这个项目『极不安全』，不建议使用。以root身份写。 exportfs命令 -a 全部挂载或者全部卸载 -r 重新挂载 -u 卸载某一个目录 -v 显示共享目录

虽然未来的pod要连接nfs，但是真正连接nfs的是pod所在的物理主机。所以作为物理主机（客户端）也要安装nfs客户端。 \[root@knode1 \~]# yum install -y nfs-utils \[root@knode2 \~]# yum install -y nfs-utils

控制节点、网络节点、计算节点、存储节点

客户端尝试挂载 \[root@knode1 \~]# mount 192.168.100.143:/nfsdata /mnt \[root@knode1 \~]# df -Th 192.168.100.143:/nfsdata nfs4 62G 2.2G 60G 4% /mnt \[root@knode1 \~]# umount /mnt

\[root@knode2 \~]# mount 192.168.100.143:/nfsdata /mnt \[root@knode2 \~]# df -Th |grep nfsdata 192.168.100.143:/nfsdata nfs4 62G 2.2G 60G 4% /mnt \[root@knode2 \~]# umount /mnt

编写yaml文件 \[root@kmaster \~]# vim podvol1.yaml \[root@kmaster \~]# cat podvol1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: podvol1 name: podvol1 spec: volumes:

* name: v1 emptyDir: {}
* name: v2 hostPath: path: /data
* name: v3 nfs: server: 192.168.100.143 path: /nfsdata containers:
* image: nginx name: vol1 imagePullPolicy: IfNotPresent resources: {} volumeMounts:
  * name: v3 mountPath: /vol1 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f podvol1.yaml pod/podvol1 created \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES podvol1 1/1 Running 0 7s 10.244.69.210 knode2 \[root@kmaster \~]# kubectl exec -ti podvol1 -- bash root@podvol1:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.8G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup 192.168.100.143:/nfsdata nfs4 62G 2.2G 60G 4% /vol1

在knode2上查看物理节点是否有挂载 \[root@knode2 ~~]# df -Th |grep nfsdata 192.168.100.143:/nfsdata nfs4 62G 2.2G 60G 4% /var/lib/kubelet/pods/fa6819e4-e093-4008-ac36-baea9c9687ae/volumes/kubernetes.io~~nfs/v3

在pod中写数据 \[root@kmaster \~]# kubectl exec -ti podvol1 -- bash root@podvol1:/# touch /vol1/abc.txt root@podvol1:/# ls /vol1/ abc.txt

在NFS服务器查看 \[root@nfs \~]# ls /nfsdata/ abc.txt

2.2 iscsi作为存储(存储知识) 关闭防火墙及selinux \[root@nfs \~]# systemctl stop firewalld \[root@nfs \~]# systemctl disable firewalld Removed /etc/systemd/system/multi-user.target.wants/firewalld.service. Removed /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service. \[root@nfs \~]# getenforce Enforcing \[root@nfs \~]# setenforce 0 \[root@nfs \~]# vim /etc/selinux/config

搭建iscsi服务器，创建一个新分区，略。（nvme0n2p1） \[root@nfs \~]# partprobe /dev/nvme0n2 \[root@nfs \~]# lsblk NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT sr0 11:0 1 10.9G 0 rom\
nvme0n1 259:0 0 100G 0 disk ├─nvme0n1p1 259:1 0 1G 0 part /boot └─nvme0n1p2 259:2 0 99G 0 part ├─cs-root 253:0 0 61.2G 0 lvm / ├─cs-swap 253:1 0 7.9G 0 lvm \[SWAP] └─cs-home 253:2 0 29.9G 0 lvm /home nvme0n2 259:3 0 10G 0 disk └─nvme0n2p1 259:5 0 10G 0 part

安装target包 \[root@nfs \~]# yum install -y target\*

\[root@nfs \~]# systemctl start target ; systemctl enable target Created symlink /etc/systemd/system/multi-user.target.wants/target.service → /usr/lib/systemd/system/target.service.

\[root@nfs \~]# targetcli Warning: Could not load preferences file /root/.targetcli/prefs.bin. targetcli shell version 2.1.53 Copyright 2011-2013 by Datera, Inc and others. For help on commands, type 'help'.

/> ls / o- / ......................................................................................................................... \[...] o- backstores .............................................................................................................. \[...] | o- block .................................................................................................. \[Storage Objects: 0] | o- fileio ................................................................................................. \[Storage Objects: 0] | o- pscsi .................................................................................................. \[Storage Objects: 0] | o- ramdisk ................................................................................................ \[Storage Objects: 0] o- iscsi ............................................................................................................ \[Targets: 0] o- loopback ......................................................................................................... \[Targets: 0]

/> /backstores/block create block1 /dev/nvme0n2p1 Created block storage object block1 using /dev/nvme0n2p1. /> ls / o- / ......................................................................................................................... \[...] o- backstores .............................................................................................................. \[...] | o- block .................................................................................................. \[Storage Objects: 1] | | o- block1 .................................................................. \[/dev/nvme0n2p1 (10.0GiB) write-thru deactivated] | | o- alua ................................................................................................... \[ALUA Groups: 1] | | o- default\_tg\_pt\_gp ....................................................................... \[ALUA state: Active/optimized] | o- fileio ................................................................................................. \[Storage Objects: 0] | o- pscsi .................................................................................................. \[Storage Objects: 0] | o- ramdisk ................................................................................................ \[Storage Objects: 0] o- iscsi ............................................................................................................ \[Targets: 0] o- loopback ......................................................................................................... \[Targets: 0]

/> /iscsi create iqn.2021-12.com.cloudcs:memeda Created target iqn.2021-12.com.cloudcs:memeda. Created TPG 1. Global pref auto\_add\_default\_portal=true Created default portal listening on all IPs (0.0.0.0), port 3260. /> ls / o- / ......................................................................................................................... \[...] o- backstores .............................................................................................................. \[...] | o- block .................................................................................................. \[Storage Objects: 1] | | o- block1 .................................................................. \[/dev/nvme0n2p1 (10.0GiB) write-thru deactivated] | | o- alua ................................................................................................... \[ALUA Groups: 1] | | o- default\_tg\_pt\_gp ....................................................................... \[ALUA state: Active/optimized] | o- fileio ................................................................................................. \[Storage Objects: 0] | o- pscsi .................................................................................................. \[Storage Objects: 0] | o- ramdisk ................................................................................................ \[Storage Objects: 0] o- iscsi ............................................................................................................ \[Targets: 1] | o- iqn.2021-12.com.cloudcs:memeda .................................................................................... \[TPGs: 1] | o- tpg1 ............................................................................................... \[no-gen-acls, no-auth] | o- acls .......................................................................................................... \[ACLs: 0] | o- luns .......................................................................................................... \[LUNs: 0] | o- portals .................................................................................................... \[Portals: 1] | o- 0.0.0.0:3260 ..................................................................................................... \[OK] o- loopback ......................................................................................................... \[Targets: 0]

/> cd /iscsi/iqn.2021-12.com.cloudcs:memeda/tpg1/ /iscsi/iqn.20...s:memeda/tpg1> ls o- tpg1 ..................................................................................................... \[no-gen-acls, no-auth] o- acls ................................................................................................................ \[ACLs: 0] o- luns ................................................................................................................ \[LUNs: 0] o- portals .......................................................................................................... \[Portals: 1] o- 0.0.0.0:3260 ........................................................................................................... \[OK] /iscsi/iqn.20...s:memeda/tpg1> luns/ create /backstores/block/block1 Created LUN 0. /iscsi/iqn.20...s:memeda/tpg1> ls / o- / ......................................................................................................................... \[...] o- backstores .............................................................................................................. \[...] | o- block .................................................................................................. \[Storage Objects: 1] | | o- block1 .................................................................... \[/dev/nvme0n2p1 (10.0GiB) write-thru activated] | | o- alua ................................................................................................... \[ALUA Groups: 1] | | o- default\_tg\_pt\_gp ....................................................................... \[ALUA state: Active/optimized] | o- fileio ................................................................................................. \[Storage Objects: 0] | o- pscsi .................................................................................................. \[Storage Objects: 0] | o- ramdisk ................................................................................................ \[Storage Objects: 0] o- iscsi ............................................................................................................ \[Targets: 1] | o- iqn.2021-12.com.cloudcs:memeda .................................................................................... \[TPGs: 1] | o- tpg1 ............................................................................................... \[no-gen-acls, no-auth] | o- acls .......................................................................................................... \[ACLs: 0] | o- luns .......................................................................................................... \[LUNs: 1] | | o- lun0 ............................................................... \[block/block1 (/dev/nvme0n2p1) (default\_tg\_pt\_gp)] | o- portals .................................................................................................... \[Portals: 1] | o- 0.0.0.0:3260 ..................................................................................................... \[OK] o- loopback ......................................................................................................... \[Targets: 0]

/iscsi/iqn.20...s:memeda/tpg1> acls/ create iqn.2021-12.com.cloudcs:acl Created Node ACL for iqn.2021-12.com.cloudcs:acl Created mapped LUN 0. /iscsi/iqn.20...s:memeda/tpg1> ls / o- / ......................................................................................................................... \[...] o- backstores .............................................................................................................. \[...] | o- block .................................................................................................. \[Storage Objects: 1] | | o- block1 .................................................................... \[/dev/nvme0n2p1 (10.0GiB) write-thru activated] | | o- alua ................................................................................................... \[ALUA Groups: 1] | | o- default\_tg\_pt\_gp ....................................................................... \[ALUA state: Active/optimized] | o- fileio ................................................................................................. \[Storage Objects: 0] | o- pscsi .................................................................................................. \[Storage Objects: 0] | o- ramdisk ................................................................................................ \[Storage Objects: 0] o- iscsi ............................................................................................................ \[Targets: 1] | o- iqn.2021-12.com.cloudcs:memeda .................................................................................... \[TPGs: 1] | o- tpg1 ............................................................................................... \[no-gen-acls, no-auth] | o- acls .......................................................................................................... \[ACLs: 1] | | o- iqn.2021-12.com.cloudcs:acl .......................................................................... \[Mapped LUNs: 1] | | o- mapped\_lun0 ................................................................................ \[lun0 block/block1 (rw)] | o- luns .......................................................................................................... \[LUNs: 1] | | o- lun0 ............................................................... \[block/block1 (/dev/nvme0n2p1) (default\_tg\_pt\_gp)] | o- portals .................................................................................................... \[Portals: 1] | o- 0.0.0.0:3260 ..................................................................................................... \[OK] o- loopback ......................................................................................................... \[Targets: 0] /iscsi/iqn.20...s:memeda/tpg1> exit Global pref auto\_save\_on\_exit=true Configuration saved to /etc/target/saveconfig.json

配置两个work节点为iscsi客户端 \[root@knode1 \~]# yum install -y iscsi\* \[root@knode2 \~]# yum install -y iscsi\*

\[root@knode1 \~]# vim /etc/iscsi/initiatorname.iscsi \[root@knode1 \~]# cat /etc/iscsi/initiatorname.iscsi InitiatorName=iqn.2021-12.com.cloudcs:acl （注意：这里输入的是acls的值）

\[root@knode1 \~]# systemctl restart iscsid \[root@knode1 \~]# systemctl enable iscsid Created symlink /etc/systemd/system/multi-user.target.wants/iscsid.service → /usr/lib/systemd/system/iscsid.service.

\[root@knode2 \~]# vim /etc/iscsi/initiatorname.iscsi \[root@knode2 \~]# cat /etc/iscsi/initiatorname.iscsi InitiatorName=iqn.2021-12.com.cloudcs:acl

\[root@knode2 \~]# systemctl restart iscsid \[root@knode2 \~]# systemctl enable iscsid Created symlink /etc/systemd/system/multi-user.target.wants/iscsid.service → /usr/lib/systemd/system/iscsid.service.

生成yaml文件 \[root@kmaster \~]# kubectl run ipod --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml > ipod.yaml \[root@kmaster \~]# vim ipod.yaml \[root@kmaster \~]# cat ipod.yaml

apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: ipod name: ipod spec: containers:

* image: nginx imagePullPolicy: IfNotPresent name: ipod volumeMounts:
  * mountPath: "/mnt/ivol" name: ivol resources: {} volumes:
* name: ivol iscsi: targetPortal: 192.168.100.161:3260 iqn: iqn.2022-12.com.cloudcs:memeda lun: 0 fsType: xfs readOnly: false dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f ipod.yaml pod/ipod created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE ipod 0/1 ContainerCreating 0 6s \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE ipod 1/1 Running 0 9s

注意：如果一致处于ContainerCreating状态，可能是防火墙未关闭。

\[root@kmaster \~]# kubectl exec -ti ipod -- bash

root@ipod:/# lsblk NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT sda 8:0 0 10G 0 disk /mnt/ivol sr0 11:0 1 10.9G 0 rom\
nvme0n1 259:0 0 100G 0 disk |-nvme0n1p1 259:1 0 1G 0 part \`-nvme0n1p2 259:2 0 99G 0 part

root@ipod:/# df -Th Filesystem Type Size Used Avail Use% Mounted on overlay overlay 62G 4.8G 57G 8% / tmpfs tmpfs 64M 0 64M 0% /dev tmpfs tmpfs 3.9G 0 3.9G 0% /sys/fs/cgroup /dev/sda xfs 10G 104M 9.9G 2% /mnt/ivol

该pod是在哪个节点上运行的？ \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES ipod 1/1 Running 0 2m23s 10.244.69.212 knode2

在knode2上查看 \[root@knode2 \~]# iscsiadm -m session tcp: \[1] 192.168.100.143:3260,1 iqn.2021-12.com.cloudcs:memeda (non-flash)

\[root@knode2 ~~]# lsblk NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT sda 8:0 0 10G 0 disk /var/lib/kubelet/pods/6dedd682-5540-4e7e-bc38-c356e280c5d9/volumes/kubernetes.io~~iscsi/ivol

同理，如果ceph或glusterfs，worker节点要配置成对应的客户端。

这种方式看起来很理想，但是配置起来稍有负载，尤其配置iscsi的时候，操作麻烦，而且不够安全，为什么说不安全？ 多个客户端多个用户连接同一个存储里面的数据，会导致目录重复，后者有没有可能会把整个存储或者某个数据删除掉呢？就会带来一些安全隐患。

3.持久化存储（存储重点） 持久化存储有两个类型：PersistentVolume/PersistentVolumeClaim

所谓的持久化存储，它只是一种k8s里面针对存储管理的一种机制。后端该对接什么存储，就对接什么存储（NFS/AWS/Ceph。。。。）

master上有很多命名空间 N1和N2，不同用户连接到不同的命名空间里面（后面讲）。用户管理pod，而专员管理存储，分开管理。 下面是一个存储服务器，共享多个目录，由专门的管理员管理。管理员会在集群中创建 PersistentVolume（PV），这个PV是全局可见的。该PV会和存储服务器中的某个目录关联。 用户要做的就是创建自己的 PVC，PVC是基于命名空间进行隔离的。而PV是全局可见的。之后把PVC和PV关联在一起。

\[root@nfs \~]# mkdir /nfsdata \[root@nfs \~]# systemctl start nfs-server.service \[root@nfs \~]# systemctl enable nfs-server.service Created symlink /etc/systemd/system/multi-user.target.wants/nfs-server.service → /usr/lib/systemd/system/nfs-server.service. \[root@nfs \~]# systemctl stop firewalld.service \[root@nfs \~]# systemctl disable firewalld.service

\[root@nfs \~]# vim /etc/exports \[root@nfs \~]# cat /etc/exports /nfsdata \*(rw,async,no\_root\_squash) \[root@nfs \~]# exportfs -arv exporting \*:/nfsdata

使用持久卷管理机制的流程：首先创建PV，之后再创建PVC，会自动关联

编辑yaml模板，可参考官方文档 https://kubernetes.io/docs/concepts/storage/persistent-volumes/ https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

\[root@kmaster \~]# vim pv1.yaml \[root@kmaster \~]# cat pv1.yaml apiVersion: v1 kind: PersistentVolume metadata: name: pv01 spec: capacity: storage: 5Gi volumeMode: Filesystem accessModes: - ReadWriteOnce storageClassName: manual nfs: path: "/data" server: 192.168.100.161

这样pv和目录就关联在一起了。 \[root@kmaster \~]# kubectl apply -f pv1.yaml persistentvolume/pv01 created \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Retain Available manual 3s

claim为空的，因为还没有任何关联。pv是全局可见的，比如我现在所在的命名空间是 vol，切换到默认空间依然可以看到。 \[root@kmaster \~]# kubens default Context "kubernetes-admin@kubernetes" modified. Active namespace is "default". \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Retain Available manual 3s

\[root@kmaster \~]# kubens vol Context "kubernetes-admin@kubernetes" modified. Active namespace is "vol".

查看pv具体属性 \[root@kmaster \~]# kubectl describe pv pv01

下面创建pvc 复制官方 \[root@kmaster \~]# vim pvc1.yaml \[root@kmaster \~]# cat pvc1.yaml apiVersion: v1 kind: PersistentVolumeClaim metadata: name: pvc01 spec: storageClassName: manual accessModes: - ReadWriteOnce resources: requests: storage: 5Gi

pvc创建完成之后，完成自动关联。 \[root@kmaster \~]# kubectl get pvc NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE mypvc01 Bound pv01 5Gi RWO manual 3s

它到底是通过什么关联的呢？一个是accessModes，这个参数在pv和pvc里面必须一致。然后大小，pvc指定的大小必须小于等于pv的大小才能关联成功。 而且一个pv只能和一个pvc进行关联。比如切换命名空间，再次运行pvc，查看状态。就会一直处于pending状态。 \[root@kmaster \~]# kubens ns1211 Context "kubernetes-admin@kubernetes" modified. Active namespace is "ns1211". \[root@kmaster \~]# kubectl apply -f pvc1.yaml persistentvolumeclaim/mypvc01 created \[root@kmaster \~]# kubectl get pvc NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE mypvc01 Pending manual 5s

那你这不是扯吗？也就是谁操作的快，谁就能关联成功，而且不可控，因为命名空间相对独立，互相看不到。咋整？ 这时候除了accessModes和大小，还有一个storageClassName可以控制。该参数随便自定义名称。但优先级最高。 也就是pv如果定义了该参数，pvc没有该参数，即使满足了accessModes和大小也不会成功。 例如： \[root@kmaster \~]# vim pv1.yaml \[root@kmaster \~]# cat pv1.yaml apiVersion: v1 kind: PersistentVolume metadata: name: pv01 spec: capacity: storage: 5Gi volumeMode: Filesystem accessModes: - ReadWriteOnce storageClassName: abc nfs: path: "/nfsdata" server: 192.168.100.143 \[root@kmaster \~]# vim pvc1.yaml \[root@kmaster \~]# cat pvc1.yaml apiVersion: v1 kind: PersistentVolumeClaim metadata: name: mypvc01 spec: accessModes: - ReadWriteOnce resources: requests: storage: 5Gi

\[root@kmaster \~]# kubectl apply -f pv1.yaml persistentvolume/pv01 created \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Retain Available abc 4s \[root@kmaster \~]# kubectl apply -f pvc1.yaml persistentvolumeclaim/mypvc01 created \[root@kmaster \~]# kubectl get pvc NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE mypvc01 Pending 3s

创建pod \[root@kmaster \~]# vim mypod.yaml \[root@kmaster \~]# cat mypod.yaml apiVersion: v1 kind: Pod metadata: name: mypod spec: containers: - name: myfrontend image: nginx imagePullPolicy: IfNotPresent volumeMounts: - mountPath: "/var/www/html" name: mypd volumes: - name: mypd persistentVolumeClaim: claimName: mypvc01 \[root@kmaster \~]# kubectl apply -f mypod.yaml pod/mypod created \[root@kmaster \~]# kubectl get pods NAME READY STATUS RESTARTS AGE mypod 1/1 Running 0 11s

\[root@kmaster \~]# kubectl get pod No resources found in vol namespace.

\[root@kmaster \~]# kubectl apply -f mypod.yaml pod/mypod created \[root@kmaster \~]# kubectl get pods NAME READY STATUS RESTARTS AGE mypod 1/1 Running 0 11s

\[root@kmaster \~]# kubectl exec -ti mypod -- bash root@mypod:/# df -Th |grep 192 192.168.100.143:/nfsdata nfs4 62G 2.2G 60G 4% /var/www/html root@mypod:/# touch /var/www/html/abc.txt 如果写数据提示permission denied ，说明没有权限。注意 /etc/exports里面的权限要加上 no\_root\_squash。

写完数据后，到共享存储上可以查看到信息。 \[root@nfs \~]# ls /nfsdata/ abc.txt

回收策略：默认retain（NFS支持recycle，理论上recycle删除pvc，也会删除nfs数据，同时pv状态变为available）

静态pvc 创建PV（关联后端存储），创建PVC（关联PV），创建POD（关联pvc），删除？ POD删除，存储数据依然存在。 PVC删除，存储数据依然存在。 PV删除，存储数据依然存在。

动态pvc POD删除，存储数据不会删除。 PVC删除，让存储数据和PV自动删除。

Retain -- manual reclamation Recycle -- basic scrub (rm -rf /thevolume/\*) 基础擦洗 已废弃

reclaimPolicy两种常用取值：Delete、Retain；

Delete：表示删除PVC的时候，PV也会一起删除，同时也删除PV所指向的实际存储空间；

Retain：表示删除PVC的时候，PV不会一起删除，而是变成Released状态等待管理员手动清理；

Delete： 优点：实现数据卷的全生命周期管理，应用删除PVC会自动删除后端云盘。能有效避免出现大量闲置云盘没有删除的情况。 缺点：删除PVC时候一起把后端云盘一起删除，如果不小心误删pvc，会出现后端数据丢失；

Retain： 优点：后端云盘需要手动清理，所以出现误删的可能性比较小； 缺点：没有实现数据卷全生命周期管理，常常会造成pvc、pv删除后，后端云盘闲置忘清理，长此以往导致大量磁盘浪费。

查看当前的回收策略。 \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Retain Bound vol/mypvc01 abc 56s

retain是不回收数据，删除pvc后，pv不可用，并长期保持released状态。 recycle会删除数据，

\[root@kmaster \~]# kubectl delete -f pvc1.yaml persistentvolumeclaim "mypvc01" deleted \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Retain Released vol/mypvc01 abc 3m23s

查看存储（产出pvc和pv后，数据依然存在。） \[root@nfs \~]# ls /nfsdata/ abc.txt

如果想要使用pv和pvc，必须全部删除，重新创建。

改成recycle \[root@kmaster \~]# vim pv1.yaml \[root@kmaster \~]# cat pv1.yaml apiVersion: v1 kind: PersistentVolume metadata: name: pv01 spec: capacity: storage: 5Gi volumeMode: Filesystem accessModes: - ReadWriteOnce storageClassName: abc persistentVolumeReclaimPolicy: Recycle nfs: path: "/nfsdata" server: 192.168.100.143

\[root@kmaster \~]# kubectl apply -f pv1.yaml persistentvolume/pv01 created

\[root@kmaster \~]# kubectl apply -f pvc1.yaml persistentvolumeclaim/mypvc01 created

\[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pv01 5Gi RWO Recycle Bound vol/mypvc01 abc 7s

\[root@kmaster \~]# kubectl get pvc NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE mypvc01 Bound pv01 5Gi RWO abc 6s

\[root@kmaster \~]# kubectl apply -f mypod.yaml pod/mypod created

\[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE mypod 1/1 Running 0 4s

这时候删除pvc，查看pv状态一直是release，之后就会变为failed。（正常逻辑是删除后，会自动变为available，可能是系统本身问题） \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pvhehe 5Gi RWO Recycle Failed vol/pvchehe hehe 3m19s

\[root@kmaster \~]# kubectl edit pv pvhehe 将claimRef下面参数删除即可。（清除UUID即可）

claimRef: apiVersion: v1 kind: PersistentVolumeClaim name: pvchehe namespace: vol resourceVersion: "421224" uid: ce1929ba-2bb8-412d-be5b-7240de2b273d

\[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pvhehe 5Gi RWO Recycle Available hehe 4m2s

现在查看pv的status就变为了 available。

然后我们回到master节点，删除pvc，这时候pv的状态就会为Available，同时再回到nfs节点中，/nfs中我们创建的文件就没了。 当然，凡事有意外，比如我这删了pvc以后，pv状态成Failed了，这个时候nfs中文件是不会被删除的，但这是意外情况，正常情况为available才对 可以看到，提示镜像pull失败了，因为我这集群是没有外网的， 因为没联网，所以没法下载回收镜像busybox:1.27，所以导致pv回收失败，这个玩意没法指定镜像获取方式， 所以没得玩，你如果在有外网的机子上测试，是不会出现这种意外报错的。

\-----------------这种方法不行------------------------ 现实不会自动变为available，查询pv的时候显示错误，下载镜像错误。 \[root@kmaster \~]# kubectl describe pv pvhehe Name: pvhehe Labels: Annotations: pv.kubernetes.io/bound-by-controller: yes Finalizers: \[kubernetes.io/pv-protection] StorageClass: aaa Status: Released Claim: vol/pvchehe Reclaim Policy: Recycle Access Modes: RWO VolumeMode: Filesystem Capacity: 5Gi Node Affinity: Message:\
Source: Type: NFS (an NFS mount that lasts the lifetime of a pod) Server: 192.168.100.143 Path: /data3 ReadOnly: false Events: Type Reason Age From Message

***

Normal RecyclerPod 2m1s persistentvolume-controller Recycler pod: Successfully assigned default/recycler-for-pvhehe to knode2 Normal RecyclerPod 2m persistentvolume-controller Recycler pod: Pulling image "registry.k8s.io/debian-base:v2.0.0" Warning RecyclerPod 97s persistentvolume-controller Recycler pod: Failed to pull image "registry.k8s.io/debian-base:v2.0.0": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/debian-base:v2.0.0": failed to resolve reference "registry.k8s.io/debian-base:v2.0.0": failed to do request: Head "https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/debian-base/manifests/v2.0.0": dial tcp 108.177.97.82:443: connect: connection refused Warning RecyclerPod 97s persistentvolume-controller Recycler pod: Error: ErrImagePull Normal RecyclerPod 96s persistentvolume-controller Recycler pod: Back-off pulling image "registry.k8s.io/debian-base:v2.0.0" Warning RecyclerPod 96s persistentvolume-controller Recycler pod: Error: ImagePullBackOff Warning RecyclerPod 15s persistentvolume-controller Recycler pod: Failed to pull image "registry.k8s.io/debian-base:v2.0.0": rpc error: code = Unknown desc = failed to pull and unpack image "registry.k8s.io/debian-base:v2.0.0": failed to resolve reference "registry.k8s.io/debian-base:v2.0.0": failed to do request: Head "https://asia-east1-docker.pkg.dev/v2/k8s-artifacts-prod/images/debian-base/manifests/v2.0.0": dial tcp 64.233.188.82:443: connect: connection refused

在华为云购买了香港主机，通过k8s.gcr.io/debian-base:v2.0.0下载对应的镜像 \[root@ecs-ex \~]# docker pull k8s.gcr.io/debian-base:v2.0.0 v2.0.0: Pulling from debian-base 597de8ba0c30: Pull complete Digest: sha256:ebda8587ec0f49eb88ee3a608ef018484908cbc5aa32556a0d78356088c185d4 Status: Downloaded newer image for k8s.gcr.io/debian-base:v2.0.0 k8s.gcr.io/debian-base:v2.0.0

### 然后传到阿里云

retain，保留静态卷供应，1-创建PV 2-创建PVC 3-创建pod（定义pvc），删除PVC，删除PV，删除底层数据 delete，必须采用动态卷供应：删除pvc，会自动删除pv和底层数据。1-创建PVC即可（PV会自动创建）

利用nfs配置动态卷供应流程：(整个过程是不需要创建pv的)

* 配置 NFS 服务器
* 获取 NFS Subdir External Provisioner 文件，地址：https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/deploy
* 如果集群启用了 RBAC，设置授权。
* 配置 NFS subdir external provisioner
* 创建 Storage Class
* 创建 PVC 和 Pod 进行测试

NFS Subdir External Provisioner是一个Kubernetes的外部卷（Persistent Volume）插件， 它允许在Kubernetes集群中动态地创建和管理基于NFS共享的子目录卷。

* 配置 NFS 服务器

***

nfs创建/vdisk \[root@nfs \~]# mkdir /vdisk \[root@nfs \~]# vim /etc/exports \[root@nfs \~]# exportfs -arv exporting \*:/vdisk \[root@nfs \~]# cat /etc/exports /vdisk \*(rw,async,no\_root\_squash)

* 获取 NFS Subdir External Provisioner 文件

***

\[root@kmaster \~]# yum install -y git \[root@kmaster volume]# git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git Cloning into 'nfs-subdir-external-provisioner'... fatal: unable to access 'https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git/': Failed to connect to github.com port 443: Connection refused

总是因为网络原因报错。去华为云购买香港主机，安装git，重新clone即可。 \[root@ecs-6db1 \~]# git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git Cloning into 'nfs-subdir-external-provisioner'... remote: Enumerating objects: 7356, done. remote: Counting objects: 100% (33/33), done. remote: Compressing objects: 100% (30/30), done. remote: Total 7356 (delta 9), reused 21 (delta 3), pack-reused 7323 Receiving objects: 100% (7356/7356), 7.68 MiB | 1.13 MiB/s, done. Resolving deltas: 100% (4072/4072), done. （保存在百度网盘 2023）

* 如果集群启用了 RBAC，设置授权。role based access control

***

上传到master，解压 \[root@kmaster deploy]# ls class.yaml deployment.yaml kustomization.yaml objects rbac.yaml test-claim.yaml test-pod.yaml \[root@kmaster deploy]# vim rbac.yaml \[root@kmaster deploy]# vim rbac.yaml \[root@kmaster deploy]# sed -i 's/namespace: default/namespace: ns823/g' rbac.yaml ---更换命名空间 \[root@kmaster deploy]# kubectl apply -f rbac.yaml serviceaccount/nfs-client-provisioner created clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

* 配置 NFS subdir external provisioner

***

\[root@kmaster deploy]# cat deployment.yaml apiVersion: apps/v1 kind: Deployment metadata: name: nfs-client-provisioner labels: app: nfs-client-provisioner

## replace with namespace where provisioner is deployed

namespace: vol spec: replicas: 1 strategy: type: Recreate selector: matchLabels: app: nfs-client-provisioner template: metadata: labels: app: nfs-client-provisioner spec: serviceAccountName: nfs-client-provisioner containers: - name: nfs-client-provisioner image: registry.cn-hangzhou.aliyuncs.com/cloudcs/nfs-subdir-external-provisioner:v4.0.2 imagePullPolicy: IfNotPresent volumeMounts: - name: nfs-client-root mountPath: /persistentvolumes env: - name: PROVISIONER\_NAME value: k8s-sigs.io/nfs-subdir-external-provisioner - name: NFS\_SERVER value: 192.168.100.143 - name: NFS\_PATH value: /vdisk volumes: - name: nfs-client-root nfs: server: 192.168.100.143 path: /vdisk

还有一个镜像，也是通过华为云香港主机下载，直接push到阿里云 image: k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2

所有节点提前下载该镜像 crictl pull registry.cn-hangzhou.aliyuncs.com/cloudcs/nfs-subdir-external-provisioner:v4.0.2

注意修改deployment.yaml 里面的image路径为 registry.cn-hangzhou.aliyuncs.com/cloudcs/nfs-subdir-external-provisioner:v4.0.2

\[root@kmaster deploy]# kubectl apply -f deployment.yaml deployment.apps/nfs-client-provisioner created \[root@kmaster deploy]# kubectl get pod NAME READY STATUS RESTARTS AGE nfs-client-provisioner-566d46b885-hqxtx 1/1 Running 0 3s

* 创建 Storage Class

***

\[root@kmaster deploy]# vim class.yaml \[root@kmaster deploy]# cat class.yaml apiVersion: storage.k8s.io/v1 kind: StorageClass metadata: name: nfs-client provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER\_NAME' parameters: archiveOnDelete: "false"

\[root@kmaster deploy]# kubectl apply -f class.yaml storageclass.storage.k8s.io/nfs-client created \[root@kmaster deploy]# kubectl get sc NAME PROVISIONER RECLAIMPOLICY VOLUMEBINDINGMODE ALLOWVOLUMEEXPANSION AGE fast kubernetes.io/gce-pd Delete Immediate false 20h nfs-client k8s-sigs.io/nfs-subdir-external-provisioner Delete Immediate false 4s

* 创建 PVC 和 Pod 进行测试

***

\[root@kmaster \~]# vim test.yaml \[root@kmaster \~]# cat test.yaml apiVersion: v1 kind: PersistentVolumeClaim metadata: name: pvchehe spec: accessModes: - ReadWriteOnce storageClassName: nfs-client resources: requests: storage: 5Gi

\[root@kmaster \~]# kubectl apply -f test.yaml persistentvolumeclaim/pvchehe created \[root@kmaster \~]# kubectl get pv NAME CAPACITY ACCESS MODES RECLAIM POLICY STATUS CLAIM STORAGECLASS REASON AGE pvc-d244652a-a8b2-4bef-8c24-375165a7bcaf 5Gi RWO Delete Bound vol/pvchehe nfs-client 3s \[root@kmaster \~]# \[root@kmaster \~]# kubectl get pvc NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE pvchehe Bound pvc-d244652a-a8b2-4bef-8c24-375165a7bcaf 5Gi RWO nfs-client 8s \[root@kmaster \~]# kubectl describe pv pvc-d244652a-a8b2-4bef-8c24-375165a7bcaf Name: pvc-d244652a-a8b2-4bef-8c24-375165a7bcaf Labels: Annotations: pv.kubernetes.io/provisioned-by: k8s-sigs.io/nfs-subdir-external-provisioner Finalizers: \[kubernetes.io/pv-protection] StorageClass: nfs-client Status: Bound Claim: vol/pvchehe Reclaim Policy: Delete Access Modes: RWO VolumeMode: Filesystem Capacity: 5Gi Node Affinity: Message:\
Source: Type: NFS (an NFS mount that lasts the lifetime of a pod) Server: 192.168.100.143 Path: /vdisk/vol-pvchehe-pvc-d244652a-a8b2-4bef-8c24-375165a7bcaf ReadOnly: false Events:

测试，删除pvc，默认pv被删除 \[root@kmaster \~]# kubectl delete -f test.yaml persistentvolumeclaim "pvchehe" deleted \[root@kmaster \~]# kubectl get pv No resources found

\[root@nfs \~]# ls /vdisk/ \[root@nfs \~]#

动态扩展，需要在存储类里面添加一行参数 只有当 PVC 的存储类中将 allowVolumeExpansion 设置为 true 时，你才可以扩充该 PVC 申领。 https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#expanding-persistent-volumes-claims

扩充 PVC 申领 特性状态： Kubernetes v1.24 \[stable] 现在，对扩充 PVC 申领的支持默认处于被启用状态。你可以扩充以下类型的卷：

azureFile（已弃用） csi flexVolume（已弃用） gcePersistentDisk（已弃用） rbd portworxVolume（已弃用）

\[root@kmaster deploy]# cat class.yaml apiVersion: storage.k8s.io/v1 kind: StorageClass metadata: name: nfs-client provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER\_NAME' parameters: archiveOnDelete: "false" allowVolumeExpansion: true

\[root@kmaster deploy]# kubectl apply -f class.yaml storageclass.storage.k8s.io/nfs-client created \[root@kmaster deploy]# kubectl get sc NAME PROVISIONER RECLAIMPOLICY VOLUMEBINDINGMODE ALLOWVOLUMEEXPANSION AGE nfs-client k8s-sigs.io/nfs-subdir-external-provisioner Delete Immediate true 2s

\[root@kmaster deploy]# kubectl edit pvc/testpvc-claim
