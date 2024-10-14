# 07 kubernetes Secret及Configmap

### secret及configmap

使用某些镜像例如mysql，是需要变量来传递密码的，也就是再编写yaml文件的时候，需要在参数里面指定明文密码。 这样就会导致一定的安全隐患。例如 \[root@kmaster \~]# kubectl run db --image mysql --image-pull-policy IfNotPresent --env="MYSQL\_ROOT\_PASSWORD=redhat" --dry-run=client -o yaml > db.yaml 查看yaml文件，可以看到所有变量参数的密码。

这时候为了安全起见，我需要单独的把密码保存到某个地方，两个东西，一个是secret，一个是configmap（简称cm）。 index.html网站文件 配置文件 参数文件

pod滚动升级--调用新的image镜像，原来通过镜像创建配置好的pod，那么经过更新之后，这些配置全部没了。

nginx 1.9 ---》pod--》开始配置，修改index.html，修改某些文件 nginx 1.25 --> pod --》之前的所有配置都没了，又要重新进行配置。

### secret

secret是基于命名空间的，相互独立无法看到。查看当前的命名空间，没有任何的secret \[root@kmaster \~]# kubectl get secrets No resources found in sales namespace.

创建一个secret，方法有很多：命令行方式、文件方式和变量方式。

1.命令行方式 \[root@kmaster \~]# kubectl create secret --help \[root@kmaster \~]# kubectl create secret generic --help |grep from kubectl create secret generic my-secret --from-literal=key1=supersecret --from-literal=key2=topsecret \[root@kmaster \~]# kubectl create secret generic mysec1 --from-literal=name1=redhat 当然，如果有第二个第三个可以继续写。 secret/mysec1 created

查看secret \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec1 Opaque 1 17s \[root@kmaster \~]# \[root@kmaster \~]# kubectl describe secrets mysec1 Name: mysec1 Namespace: sales Labels: Annotations:

Type: Opaque

## Data

name1: 6 bytes

这样，未来创建需要密码的pod，直接引用 mysec1 中的 name1 即可。

secret类型 1）默认为 Opaque，base64编码格式，用来存储密码和密钥，但是它可以通过base64 -decode解码得到原始数据，安全性较弱。 2）dockerconfigjson：存储私有docker registry的认证信息 3）service-account-token：用于被serviceaccount引用。sa创建的时候，k8s会创建对应的secret。如果pod使用了sc，对应的secret会自动挂载到pod目录。

通过解码来获得密码 \[root@kmaster \~]# kubectl get secrets mysec1 -o yaml apiVersion: v1 data: name1: cmVkaGF0 kind: Secret metadata: creationTimestamp: "2023-02-08T13:38:22Z" name: mysec1 namespace: sales resourceVersion: "142963" uid: 93cd941a-cc5d-492b-a6cc-db47234964f4 type: Opaque

name1: cmVkaGF0 注意这个值，是被base64进行了编码。 \[root@kmaster \~]# echo -n redhat | base64 cmVkaGF0 解码 \[root@kmaster \~]# echo -n cmVkaGF0 |base64 -d redhat

2.文件方式

\[root@kmaster \~]# echo -n redhat > name1 \[root@kmaster \~]# cat name1 redhat \[root@kmaster \~]# kubectl create secret generic mysec2 --from-file=./name1 secret/mysec2 created \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec1 Opaque 1 77m mysec2 Opaque 1 8s

\[root@kmaster \~]# kubectl get secrets mysec2 -o yaml apiVersion: v1 data: name1: cmVkaGF0 kind: Secret metadata: creationTimestamp: "2023-02-08T14:56:11Z" name: mysec2 namespace: sales resourceVersion: "149530" uid: f24ab95a-c2ef-4b1b-b0ea-eaf917382562 type: Opaque

3.变量方式 \[root@kmaster \~]# vim abc.txt \[root@kmaster \~]# cat abc.txt name1=redhat1 name2=redhat2 \[root@kmaster \~]# kubectl create secret generic mysec3 --from-env-file=abc.txt secret/mysec3 created \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec1 Opaque 1 82m mysec2 Opaque 1 4m42s mysec3 Opaque 2 2s \[root@kmaster \~]# kubectl get secrets mysec3 -o yaml | head -5 apiVersion: v1 data: name1: cmVkaGF0MQ== name2: cmVkaGF0Mg== kind: Secret

4.yaml文件方式 \[root@kmaster \~]# echo -n hehe | base64 aGVoZQ== \[root@kmaster \~]# echo -n aGVoZQ== | base64 YUdWb1pRPT0= \[root@kmaster \~]# echo -n aGVoZQ== | base64 -d hehe

\[root@kmaster \~]# vim mysec4.yaml \[root@kmaster \~]# cat mysec4.yaml apiVersion: v1 data: name1: aGVoZQ== name2: bWVtZWRh kind: Secret metadata: creationTimestamp: "2023-02-08T15:00:51Z" name: mysec4 namespace: sales resourceVersion: "149922" uid: 0ba4fb36-71c8-49f7-8d26-92ee1b999fca type: Opaque \[root@kmaster \~]# kubectl apply -f mysec4.yaml secret/mysec4 created \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec1 Opaque 1 89m mysec2 Opaque 1 11m mysec3 Opaque 2 6m51s mysec4 Opaque 2 3s

删除 \[root@kmaster \~]# kubectl delete secrets mysec mysec1 mysec2 mysec3 mysec4\
\[root@kmaster \~]# kubectl delete secrets mysec1 secret "mysec1" deleted \[root@kmaster \~]# kubectl delete secrets mysec2 secret "mysec2" deleted \[root@kmaster \~]# kubectl delete secrets mysec3 secret "mysec3" deleted \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec4 Opaque 2 82s

如何使用secret？ 1.使用卷挂载方式 可以参考官方文档 https://kubernetes.io/docs/concepts/configuration/secret/ 也可以自己写 \[root@kmaster \~]# kubectl run pod1 --image nginx --dry-run=client -o yaml > pod1.yaml \[root@kmaster \~]# vim pod1.yaml \[root@kmaster \~]# cat pod1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod1 name: pod1 spec: volumes:

* name: v1 secret: secretName: mysec4 optional: true containers:
* image: nginx name: pod1 resources: {} imagePullPolicy: IfNotPresent volumeMounts:
  * name: v1 mountPath: "/etc/xx" readOnly: true dnsPolicy: ClusterFirst restartPolicy: Always status: {}

aaa=111 卷方式挂载进来之后 创建一个文件aaa键 ，里面的内容111值

\[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec4 Opaque 2 18m \[root@kmaster \~]# kubectl apply -f pod1.yaml pod/pod1 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 3s \[root@kmaster \~]# kubectl exec -ti pod pod1 pods/\
\[root@kmaster \~]# kubectl exec -ti pod pod1 pods/\
\[root@kmaster \~]# kubectl exec -ti pod1 -- bash root@pod1:/# cd /etc/xx root@pod1:/etc/xx# ls name1 name2 root@pod1:/etc/xx# cat name1 heheroot@pod1:/etc/xx# cat name2 memedaroot@pod1:/etc/xx#

在系统里面以文件的方式来显示了。文件就是键，内容就是值。 但是对于某些镜像，它在运行时，需要这些变量，比如mysql，要加上 MYSQL\_ROOT\_PASSWORD.那如何把name1的值，设置为 MYSQL\_ROOT\_PASSWORD 的值呢？ 以卷的方式挂载，并不适合用于镜像变量传递密码。那它的应用场景是什么呢？一般用于传递配置文件。 root@pod1:\~# ls /etc/nginx/nginx.conf /etc/nginx/nginx.conf

\[root@kmaster \~]# vim 123.conf \[root@kmaster \~]# cat 123.conf 123

\[root@kmaster \~]# kubectl create secret generic --help |grep from \[root@kmaster \~]# kubectl create secret generic mysecfile --from-file=./123.conf secret/mysecfile created \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec4 Opaque 2 7h48m mysecfile Opaque 1 5s

\[root@kmaster \~]# vim pod1.yaml \[root@kmaster \~]# cat pod1.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod1 name: pod1 spec: volumes:

* name: v1 secret: secretName: mysecfile optional: true containers:
* image: nginx name: pod1 resources: {} imagePullPolicy: IfNotPresent volumeMounts:
  * name: v1 mountPath: "/etc/xx" readOnly: true dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f pod1.yaml pod/pod1 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod1 1/1 Running 0 3s \[root@kmaster \~]# kubectl exec -ti pod1 -- bash root@pod1:/# ls /etc/xx/ 123.conf root@pod1:/#

这种方式并不适合类似于mysql、wordpress等应用，需要提前加载secret变量。

2.变量方式 https://kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/

\[root@kmaster \~]# kubectl run pod2 --image nginx --image-pull-policy IfNotPresent --dry-run=client -o yaml > pod2.yaml \[root@kmaster \~]# kubectl get secrets NAME TYPE DATA AGE mysec4 Opaque 2 8h mysecfile Opaque 1 15m \[root@kmaster \~]# kubectl get secrets mysec4 -o yaml |head -5 apiVersion: v1 data: name1: aGVoZQ== name2: bWVtZWRh kind: Secret

\[root@kmaster \~]# vim pod2.yaml \[root@kmaster \~]# cat pod2.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: pod2 name: pod2 spec: containers:

* image: nginx imagePullPolicy: IfNotPresent name: pod2 resources: {} env:
  * name: env4 valueFrom: secretKeyRef: name: mysec4 key: name1 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f pod2.yaml pod/pod2 created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE pod2 1/1 Running 0 3s \[root@kmaster \~]# kubectl exec -ti pod2 -- bash root@pod2:/# echo $env4 hehe

例如mysql \[root@kmaster \~]# kubectl run mydb --image mysql --image-pull-policy IfNotPresent --port 3306 --dry-run=client -o yaml > mydb.yaml \[root@kmaster \~]# vim mydb.yaml \[root@kmaster \~]# cat mydb.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: mydb name: mydb spec: containers:

* image: mysql imagePullPolicy: IfNotPresent name: mydb ports:
  * containerPort: 3306 resources: {} env:
  * name: MYSQL\_ROOT\_PASSWORD valueFrom: secretKeyRef: name: mysec4 key: name1 dnsPolicy: ClusterFirst restartPolicy: Always status: {}

\[root@kmaster \~]# kubectl apply -f mydb.yaml pod/mydb created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE mydb 1/1 Running 0 2s pod2 1/1 Running 0 5m21s \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES mydb 1/1 Running 0 8s 10.244.69.209 knode2 pod2 1/1 Running 0 5m27s 10.244.69.208 knode2

注意：如果执行mysql提示没有该命令，说明没有安装mysql客户端。 yum install -y mariadb

\[root@kmaster \~]# mysql -uroot -predhat -h 10.244.69.209 ERROR 1045 (28000): Access denied for user 'root'@'10.244.189.0' (using password: YES)

\[root@kmaster \~]# kubectl exec -ti mydb -- /bin/bash bash-4.4# exit\
exit \[root@kmaster \~]# mysql -uroot -phehe -h 10.244.69.209 Welcome to the MariaDB monitor. Commands end with ; or \g. Your MySQL connection id is 9 Server version: 8.0.32 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL \[(none)]> show databases; +--------------------+ | Database | +--------------------+ | information\_schema | | mysql | | performance\_schema | | sys | +--------------------+ 4 rows in set (0.003 sec)

### configmap

用法和secret是一样的。 \[root@kmaster \~]# kubectl get cm NAME DATA AGE kube-root-ca.crt 1 37h \[root@kmaster \~]# kubectl create cm --help |grep from

\[root@kmaster \~]# kubectl create configmap my-config --from-literal=name1=redhat1 configmap/my-config created \[root@kmaster \~]# kubectl get cm my-config -o yaml | head apiVersion: v1 data: name1: redhat1 kind: ConfigMap metadata: creationTimestamp: "2023-02-08T23:36:41Z" name: my-config namespace: sales resourceVersion: "193462" uid: 48fa34e2-9653-477d-8fbf-385d964a44b3

可以看到，这里它是以明文的方式来显示的密码。但是用法和secret一致。参考 https://kubernetes.io/docs/concepts/configuration/configmap/

\[root@kmaster \~]# kubectl delete -f mydb.yaml pod "mydb" deleted \[root@kmaster \~]# vim mydb.yaml \[root@kmaster \~]# vim mydb.yaml \[root@kmaster \~]# cat mydb.yaml apiVersion: v1 kind: Pod metadata: creationTimestamp: null labels: run: mydb name: mydb spec: containers:

* image: mysql imagePullPolicy: IfNotPresent name: mydb ports:
  * containerPort: 3306 resources: {} env:
  * name: MYSQL\_ROOT\_PASSWORD valueFrom: configMapKeyRef: name: my-config key: name1 dnsPolicy: ClusterFirst restartPolicy: Always status: {} \[root@kmaster \~]# kubectl apply -f mydb.yaml pod/mydb created \[root@kmaster \~]# kubectl get pod NAME READY STATUS RESTARTS AGE mydb 1/1 Running 0 2s pod2 1/1 Running 0 34m \[root@kmaster \~]# kubectl get pod -o wide NAME READY STATUS RESTARTS AGE IP NODE NOMINATED NODE READINESS GATES mydb 1/1 Running 0 20s 10.244.69.210 knode2 pod2 1/1 Running 0 35m 10.244.69.208 knode2 \[root@kmaster \~]# mysql -uroot -predhat1 -h 10.244.69.210 Welcome to the MariaDB monitor. Commands end with ; or \g. Your MySQL connection id is 8 Server version: 8.0.32 MySQL Community Server - GPL

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL \[(none)]>

配合滚动升级操作，如果在pod里面自动加载configmap

手工创建一个index.html文件，通过命令行的方式创建configmap echo 1111 > index.html kubectl create configmap webindex --from-file=index.html

修改deploy，把config以卷的方式进行挂载给pod。

\[root@kmaster \~]# cat web1.yaml apiVersion: apps/v1 kind: Deployment metadata: creationTimestamp: null labels: app: web1 aaa: web2 bbb: web3 name: web1 spec: replicas: 3 selector: matchLabels: app1: web1 strategy: {} template: metadata: creationTimestamp: null labels: app1: web1 spec: volumes: - name: foo configMap: name: webindex containers: - image: nginx imagePullPolicy: Never name: nginx volumeMounts: - name: foo mountPath: "/usr/share/nginx/html" readOnly: true resources: {} status: {}

之后，创建 kubectl apply -f web1.yaml 完成后，访问测试 http://192.168.100.205:31704/ http://192.168.100.206:31704/ http://192.168.100.207:31704/

内容为：111

这时候如果把nginx进行滚动升级，也不会影响访问操作 kubectl set image deploy web1 nginx=nginx:1.20 --record=true

访问内容不会改变，依然是111
