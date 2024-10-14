# 12 helm

HELM

在linux里面安装rpm包的时候，可以通过yum/dnf来安装，自动解决所有包依赖关系。

helm作用是把许多定义（pod、svc、pvc、pv。。。），比如 svc，deployment等一次性全部定义好，放在源里统一管理。

对于helm来说，要安装软件，需要提前编辑 yaml 文件。 它可以帮助我们进行一键式安装应用。 网上存在很多打包好的应用，互联网上有很多源，类似于yum源，源里面有很多包

helm包

源1 源2 包1 包2.tar 包3 包4 | | | 包可以下载本地，也是通过helm下载到本地。下载好之后手工解压。下载的tar叫做包。 tar--解压--得到文件夹（叫chart，里面包含了创建应用的所有的属性）---------> k8s 环境 这时候，本地安装了helm之后，根据chart要求进行部署安装，helm会自动的部署到k8s环境里面。

另外，也可以在线指定源，不需要下载，自动部署。建议下载过来。

helm版本

v2和v3 版本

v2版本有些复杂。 v3版本简化了很多操作，没有了tiller端，只有单纯的helm客户端。

安装过程 https://helm.sh/docs/intro/install/

下载软件 https://github.com/helm/helm/releases/tag/v3.13.1

helm-v3.11.2-linux-amd64.tar.gz

解压 \[root@kmaster1 \~]# tar -zxvf helm-v3.11.2-linux-amd64.tar.gz linux-amd64/ linux-amd64/helm linux-amd64/LICENSE linux-amd64/README.md \[root@kmaster1 \~]# ls anaconda-ks.cfg bridge-utils-1.5-9.el7.x86\_64.rpm helm-v3.11.2-linux-amd64.tar.gz linux-amd64 Stream8\_docker.sh \[root@kmaster1 \~]# cd linux-amd64/ \[root@kmaster1 linux-amd64]# ls helm LICENSE README.md \[root@kmaster1 linux-amd64]# mv helm /usr/local/bin/helm \[root@kmaster1 linux-amd64]# \[root@kmaster1 linux-amd64]# \[root@kmaster1 linux-amd64]# helm help

配置环境

helm completion bash > \~/.helm helm completion bash > \~/.helmrc echo "source \~/.helmrc" >> \~/.bashrc source \~/.bashrc

配置仓库，阿里云无法使用

微软的chart仓库 http://mirror.azure.cn/kubernetes/charts/ 这个仓库强烈推荐，基本上官网有的chart这里都有。

\[root@kmaster1 linux-amd64]# helm repo add weiruan http://mirror.azure.cn/kubernetes/charts/ "weiruan" has been added to your repositories \[root@kmaster1 linux-amd64]# helm repo list NAME URL\
weiruan http://mirror.azure.cn/kubernetes/charts/

删除源 \[root@kmaster1 linux-amd64]# helm repo add (add a chart repository) remove (remove one or more chart repositories) index (generate an index file given a directory containing packaged charts) update (update information of available charts locally from chart repositories) list (list chart repositories)\
\[root@kmaster1 linux-amd64]# helm repo remove weiruan "weiruan" has been removed from your repositories

\[root@kmaster1 linux-amd64]# helm search repo mysql NAME CHART VERSION APP VERSION DESCRIPTION\
weiruan/mysql 1.6.9 5.7.30 DEPRECATED - Fast, reliable, scalable, and easy... weiruan/mysqldump 2.6.2 2.4.1 DEPRECATED! - A Helm chart to help backup MySQL... weiruan/prometheus-mysql-exporter 0.7.1 v0.11.0 DEPRECATED A Helm chart for prometheus mysql ex... weiruan/percona 1.2.3 5.7.26 DEPRECATED - free, fully compatible, enhanced, ... weiruan/percona-xtradb-cluster 1.0.8 5.7.19 DEPRECATED - free, fully compatible, enhanced, ... weiruan/phpmyadmin 4.3.5 5.0.1 DEPRECATED phpMyAdmin is an mysql administratio... weiruan/gcloud-sqlproxy 0.6.1 1.11 DEPRECATED Google Cloud SQL Proxy\
weiruan/mariadb 7.3.14 10.3.22 DEPRECATED Fast, reliable, scalable, and easy t...

\[root@kmaster1 linux-amd64]# helm pull weiruan/mysql \[root@kmaster1 linux-amd64]# ls LICENSE mysql-1.6.9.tgz README.md

\[root@kmaster1 linux-amd64]# ls LICENSE mysql-1.6.9.tgz README.md \[root@kmaster1 linux-amd64]# tar -zxvf mysql-1.6.9.tgz mysql/Chart.yaml mysql/values.yaml mysql/templates/NOTES.txt mysql/templates/\_helpers.tpl mysql/templates/configurationFiles-configmap.yaml mysql/templates/deployment.yaml mysql/templates/initializationFiles-configmap.yaml mysql/templates/pvc.yaml mysql/templates/secrets.yaml mysql/templates/serviceaccount.yaml mysql/templates/servicemonitor.yaml mysql/templates/svc.yaml mysql/templates/tests/test-configmap.yaml mysql/templates/tests/test.yaml mysql/.helmignore mysql/README.md \[root@kmaster1 linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md \[root@kmaster1 linux-amd64]# rm -rf mysql-1.6.9.tgz \[root@kmaster1 linux-amd64]# helm package mysql/ Successfully packaged chart and saved it to: /root/linux-amd64/mysql-1.6.9.tgz \[root@kmaster1 linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md

这个目录就是一个chart，也可以在本地打包 \[root@kmaster mysql]# cd .. \[root@kmaster linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md \[root@kmaster linux-amd64]# rm -rf mysql-1.6.9.tgz \[root@kmaster linux-amd64]# helm package mysql/ Successfully packaged chart and saved it to: /root/linux-amd64/mysql-1.6.9.tgz \[root@kmaster linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md

这里文件夹就是一个mysql，打包后，它怎么知道具体版本的呢？这里有个chart的文件，里面记录了元数据信息。

\[root@kmaster linux-amd64]# cd mysql/ \[root@kmaster mysql]# ls Chart.yaml README.md templates values.yaml \[root@kmaster mysql]# cat Chart.yaml apiVersion: v1 appVersion: 5.7.30 deprecated: true description: DEPRECATED - Fast, reliable, scalable, and easy to use open-source relational database system. home: https://www.mysql.com/ icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png keywords:

* mysql
* database
* sql name: mysql sources:
* https://github.com/kubernetes/charts
* https://github.com/docker-library/mysql version: 1.6.9 \[root@kmaster mysql]#

然后根据values文件，部署pod 查看values文件内容

版本可以改为最新的 busybox: image: "busybox" tag: "latest"

testFramework关闭 testFramework: enabled: false

mysql的root密码

### Default: random 10 character string

mysqlRootPassword: memeda

是否使用持久卷，改为false

### Persist data to a persistent volume

persistence: enabled: false

关闭ssl ssl: enabled: false

那么这个yaml文件里面指定的这么多参数，它是如何自动创建的呢？比如持久卷如何创建？ \[root@kmaster mysql]# ls Chart.yaml README.md templates values.yaml \[root@kmaster mysql]# ls templates/ configurationFiles-configmap.yaml \_helpers.tpl NOTES.txt secrets.yaml servicemonitor.yaml tests deployment.yaml initializationFiles-configmap.yaml pvc.yaml serviceaccount.yaml svc.yaml

在templates目录里面，有很多yaml文件，比如pvc，svc，secrests等等。就是通过这个yaml来创建的。而这个文件里面记录的都是变量，根据变量去取值。

接下来我们通过helm来部署一个应用。 \[root@kmaster mysql]# helm ls NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION

可以在本地构建 helm install name chart目录 也可以在线构建 helm install name 源/名称

一般来说，使用公共的helm源，是不知道里面具体要设置什么值，建议最好down下来。

\[root@kmaster mysql]# cd .. \[root@kmaster linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md \[root@kmaster linux-amd64]# helm install db mysql mysql/ mysql-1.6.9.tgz\
\[root@kmaster linux-amd64]# helm install db mysql/ WARNING: This chart is deprecated NAME: db LAST DEPLOYED: Sat Apr 22 21:50:38 2023 NAMESPACE: default STATUS: deployed REVISION: 1 TEST SUITE: None NOTES: MySQL can be accessed via port 3306 on the following DNS name from within your cluster: db-mysql.default.svc.cluster.local

To get your root password run:

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default db-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
```

To connect to your database:

1.  Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
2.  Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y
3. Connect using the mysql cli, then provide your password: $ mysql -h db-mysql -p

To connect to your database directly from outside the K8s cluster: MYSQL\_HOST=127.0.0.1 MYSQL\_PORT=3306

```
# Execute the following command to route the connection:
kubectl port-forward svc/db-mysql 3306

mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

\[root@kmaster linux-amd64]# helm ls NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION db default 1 2023-04-22 21:50:38.157284162 +0800 CST deployed mysql-1.6.9 5.7.30

\[root@kmaster linux-amd64]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES db-mysql-86f5ff66dc-zt8lb 1/1 Running 0 69s 10.244.195.133 knode1

安装客户端，尝试连接 \[root@kmaster linux-amd64]# yum install -y mariadb \[root@kmaster linux-amd64]# mysql -uroot -pmemeda -h 10.244.195.133 Welcome to the MariaDB monitor. Commands end with ; or \g. Your MySQL connection id is 22 Server version: 5.7.30 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL \[(none)]> show databases; +--------------------+ | Database | +--------------------+ | information\_schema | | mysql | | performance\_schema | | sys | +--------------------+ 4 rows in set (0.001 sec)

MySQL \[(none)]>

删除helm \[root@kmaster linux-amd64]# helm ls NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION db default 1 2023-04-22 21:50:38.157284162 +0800 CST deployed mysql-1.6.9 5.7.30\
\[root@kmaster linux-amd64]# helm delete db release "db" uninstalled \[root@kmaster linux-amd64]# helm ls NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION \[root@kmaster linux-amd64]# kubectl get pod No resources found in default namespace.

使用helm安装软件包的时候，都是连接到互联网上去，但是生产环境有可能无法联网，这时候我们就可以选择搭建自己的私有仓库。比如就拿mysql为例。 用一个web服务器（为了方便演示，直接使用容器来做），创建一个新的目录，专门保存包资源。

\[root@knode2 \~]# docker images REPOSITORY TAG IMAGE ID CREATED SIZE \[root@knode2 \~]# docker pull nginx Using default tag: latest latest: Pulling from library/nginx a2abf6c4d29d: Pull complete a9edb18cadd1: Pull complete 589b7251471a: Pull complete 186b1aaa4aa6: Pull complete b4df32aa5a72: Pull complete a0bcbecc962e: Pull complete Digest: sha256:0d17b565c37bcbd895e9d92315a05c1c3c9a29f762b011a10c54a66cd53c9b31 Status: Downloaded newer image for nginx:latest docker.io/library/nginx:latest \[root@knode2 \~]# docker images REPOSITORY TAG IMAGE ID CREATED SIZE nginx latest 605c77e624dd 15 months ago 141MB

\[root@knode2 \~]# docker run -tid --name web-chart --restart always -p 80:80 -v /charts:/usr/share/nginx/html/charts nginx 8ab605382848ab548fbcd98b0406560d5f3b9e4561e6c74774556ee475d951dd

注意：当我们访问node2主机ip的时候，默认加载的网站跟目录为 /usr/share/nginx/html，如果要访问charts目录，则是主机ip/charts 首先将chart目录进行打包 \[root@kmaster linux-amd64]# ls LICENSE mysql mysql-1.6.9.tgz README.md \[root@kmaster linux-amd64]# mkdir aaa \[root@kmaster linux-amd64]# helm package mysql Successfully packaged chart and saved it to: /root/linux-amd64/mysql-1.6.9.tgz \[root@kmaster linux-amd64]# ls aaa LICENSE mysql mysql-1.6.9.tgz README.md \[root@kmaster linux-amd64]# cp mysql-1.6.9.tgz aaa/ \[root@kmaster linux-amd64]# ls aaa/ mysql-1.6.9.tgz

为mysql包创建索引信息 \[root@kmaster linux-amd64]# helm repo index aaa/ --url http://192.168.146.141/charts \[root@kmaster linux-amd64]# ls aaa/ index.yaml mysql-1.6.9.tgz

\[root@kmaster linux-amd64]# cat aaa/index.yaml apiVersion: v1 entries: mysql:

* apiVersion: v1 appVersion: 5.7.30 created: "2022-04-22T22:10:06.970175025+08:00" deprecated: true description: DEPRECATED - Fast, reliable, scalable, and easy to use open-source relational database system. digest: 7c1e3d7c890cc2ed306d47ebd9f8b1d4b493e9caab296c5f29ff8af94fcd7373 home: https://www.mysql.com/ icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png keywords:
  * mysql
  * database
  * sql name: mysql sources:
  * https://github.com/kubernetes/charts
  * https://github.com/docker-library/mysql urls:
  * http://192.168.146.141/charts/mysql-1.6.9.tgz version: 1.6.9 generated: "2022-04-22T22:10:06.969209519+08:00"

将aaa目录下的索引文件及包资源拷贝到容器web服务器里面 \[root@kmaster linux-amd64]# scp aaa/\* 192.168.146.141:/charts root@192.168.146.141's password: index.yaml 100% 742 1.8MB/s 00:00\
mysql-1.6.9.tgz 100% 11KB 8.4MB/s 00:00\
\[root@kmaster linux-amd64]#

\[root@knode2 \~]# ls /charts/ index.yaml mysql-1.6.9.tgz

\[root@knode2 \~]# docker ps -a CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES 8ab605382848 nginx "/docker-entrypoint.…" 13 minutes ago Up 13 minutes 0.0.0.0:80->80/tcp, :::80->80/tcp web-chart \[root@knode2 \~]# docker exec -ti web-chart /bin/bash root@8ab605382848:/# ls /usr/share/nginx/html/charts/ index.yaml mysql-1.6.9.tgz

做好之后，在helm中添加新的本地仓库。 \[root@kmaster linux-amd64]# helm repo add myrepo http://192.168.146.141/charts "myrepo" has been added to your repositories \[root@kmaster linux-amd64]# helm repo list NAME URL\
weiruan http://mirror.azure.cn/kubernetes/charts/ myrepo http://192.168.146.141/charts

这样，私有仓库就搭建好了。查询下。 \[root@kmaster linux-amd64]# helm search repo mysql NAME CHART VERSION APP VERSION DESCRIPTION\
myrepo/mysql 1.6.9 5.7.30 DEPRECATED - Fast, reliable, scalable, and easy... weiruan/mysql 1.6.9 5.7.30 DEPRECATED - Fast, reliable, scalable, and easy... weiruan/mysqldump 2.6.2 2.4.1 DEPRECATED! - A Helm chart to help backup MySQL... weiruan/prometheus-mysql-exporter 0.7.1 v0.11.0 DEPRECATED A Helm chart for prometheus mysql ex... weiruan/percona 1.2.3 5.7.26 DEPRECATED - free, fully compatible, enhanced, ... weiruan/percona-xtradb-cluster 1.0.8 5.7.19 DEPRECATED - free, fully compatible, enhanced, ... weiruan/phpmyadmin 4.3.5 5.0.1 DEPRECATED phpMyAdmin is an mysql administratio... weiruan/gcloud-sqlproxy 0.6.1 1.11 DEPRECATED Google Cloud SQL Proxy\
weiruan/mariadb 7.3.14 10.3.22 DEPRECATED Fast, reliable, scalable, and easy t...

通过helm私有仓库在线安装mysql。 \[root@kmaster linux-amd64]# kubectl get pod No resources found in default namespace. \[root@kmaster linux-amd64]# hel helm help\
\[root@kmaster linux-amd64]# helm install db myrepo/mysql WARNING: This chart is deprecated NAME: db LAST DEPLOYED: Sat Apr 22 22:25:41 2023 NAMESPACE: default STATUS: deployed REVISION: 1 TEST SUITE: None NOTES: MySQL can be accessed via port 3306 on the following DNS name from within your cluster: db-mysql.default.svc.cluster.local

To get your root password run:

```
MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default db-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)
```

To connect to your database:

1.  Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
2.  Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y
3. Connect using the mysql cli, then provide your password: $ mysql -h db-mysql -p

To connect to your database directly from outside the K8s cluster: MYSQL\_HOST=127.0.0.1 MYSQL\_PORT=3306

```
# Execute the following command to route the connection:
kubectl port-forward svc/db-mysql 3306

mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

\[root@kmaster linux-amd64]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES db-mysql-86f5ff66dc-sjphg 0/1 Running 0 6s 10.244.195.134 knode1 \[root@kmaster linux-amd64]#
