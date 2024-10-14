# 09 job和cronjob

https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/job/#job-termination-and-cleanup

传统运行的pod，比如deployment管理的pod，或手工管理的pod，只要创建好pod，该pod会一致运行下去。pod里面运行的是一个daemon守护进程。 pod没有问题的情况下可以长期运行。 但有时候想临时做一件事情，比如测试等，执行个脚本等。一下子就可以完成的。这种情况下可以通过job或cronjob来完成。

对于job来说是一次性的，做完就完成了。 如果想要周期性，循环来做，就要考虑cronjob了。

\[root@kmaster job]# kubectl create job job1 --image busybox --dry-run=client -o yaml -- sh -c "echo 123" > job1.yaml \[root@kmaster job]# ls job1.yaml \[root@kmaster job]# vim job1.yaml \[root@kmaster job]# cat job1.yaml apiVersion: batch/v1 kind: Job metadata: creationTimestamp: null name: job1 spec: template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - echo 123 image: busybox name: job1 resources: {} restartPolicy: Never status: {} \[root@kmaster job]# kubectl apply -f job1.yaml job.batch/job1 created \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-7hlmx 0/1 ContainerCreating 0 2s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-7hlmx 0/1 ContainerCreating 0 7s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-7hlmx 0/1 Completed 0 92s

pod运行完成后，并不会继续重启，因为job创建的pod，重启策略里没有 always 选项。做完就做完了。

其他参数 带工作队列的并行 Job：完成3个pod 不设置 spec.completions，默认值为 .spec.parallelism。 多个 Pod 之间必须相互协调，或者借助外部服务确定每个 Pod 要处理哪个工作条目。 例如，任一 Pod 都可以从工作队列中取走最多 N 个工作条目。 每个 Pod 都可以独立确定是否其它 Pod 都已完成，进而确定 Job 是否完成。

带有确定完成计数的 Job，即 .spec.completions 不为 null 的 Job， 都可以在其 .spec.completionMode 中设置完成模式：job结束需要成功运行pod个数，即状态为completed的pod数。 NonIndexed（默认值）：当成功完成的 Pod 个数达到 .spec.completions 所 设值时认为 Job 已经完成。换言之， 每个 Job 完成事件都是独立无关且同质的。 要注意的是，当 .spec.completions 取值为 null 时，Job 被隐式处理为 NonIndexed。

Pod 回退失效策略 在有些情形下，你可能希望 Job 在经历若干次重试之后直接进入失败状态， 因为这很可能意味着遇到了配置错误。 为了实现这点，可以将 .spec.backoffLimit 设置为视 Job 为失败之前的重试次数。 失效回退的限制值默认为 6。 与 Job 相关的失效的 Pod 会被 Job 控制器重建，回退重试时间将会按指数增长 （从 10 秒、20 秒到 40 秒）最多至 6 分钟。 如果pod失败，则重试几次？

这里parallelism的值指的是一次性并行运行几个pod，该值不会超过completions的值。

\[root@kmaster job]# cp job1.yaml job2.yaml

\[root@kmaster job]# vim job2.yaml \[root@kmaster job]# cat job2.yaml apiVersion: batch/v1 kind: Job metadata: creationTimestamp: null name: job1 spec: parallelism: 3 ---并行运行几个 completions: 3 ---必须完成几个 backoffLimit: 2 ---失败了重启几次 template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - echo 123 image: busybox imagePullPolicy: IfNotPresent name: job1 resources: {} restartPolicy: Never status: {}

apiVersion: batch/v1 kind: Job metadata: creationTimestamp: null name: job1 spec: parallelism: 3 completions: 3 backoffLimit: 2 template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - echo 123 image: busybox imagePullPolicy: IfNotPresent name: job1 resources: {} restartPolicy: Never status: {}

\[root@kmaster job]# kubectl apply -f job2.yaml job.batch/job1 created \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-kfh9p 0/1 ContainerCreating 0 2s job1-lndtq 0/1 Completed 0 2s job1-tt4cs 0/1 ContainerCreating 0 2s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-kfh9p 0/1 Completed 0 4s job1-lndtq 0/1 Completed 0 4s job1-tt4cs 0/1 Completed 0 4s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE job1-kfh9p 0/1 Completed 0 5s job1-lndtq 0/1 Completed 0 5s job1-tt4cs 0/1 Completed 0 5s

创建完成了3个，或者分批次 apiVersion: batch/v1 kind: Job metadata: creationTimestamp: null name: job1 spec: parallelism: 2 ---并行运行几个 completions: 6 ---必须完成几个 backoffLimit: 2 ---失败了重启几次 template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - sleep 5 image: busybox imagePullPolicy: IfNotPresent name: job1 resources: {} restartPolicy: Never status: {}

另外注意 job的restart策略： restartPolicy 指定什么情况下需要重启容器。对于 Job，只能设置为 Never 或者 OnFailure。对于其他 controller（比如 Deployment）可以设置为 Always 。 Never：只要任务没有完成，则是新创建pod运行，直到job完成，会产生多个pod OnFailure：只要pod没有完成，则会重启pod，直到job完成

例子：打印pi小数点后2000位。注意image版本问题 \[root@kmaster job]# cat pi.yaml apiVersion: batch/v1 kind: Job metadata: name: pi spec: template: spec: containers: - name: pi image: perl:5.34.0 imagePullPolicy: IfNotPresent command: \["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"] restartPolicy: Never

\[root@knode1 \~]# crictl pull perl:5.34.0 \[root@knode2 \~]# crictl pull perl:5.34.0

\[root@kmaster job]# kubectl apply -f pi.yaml job.batch/pi created \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE pi-x99td 1/1 Running 0 2s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE pi-x99td 1/1 Running 0 4s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE pi-x99td 0/1 Completed 0 5s

\[root@kmaster job]# kubectl logs pi-x99td 3.141592653589793238462643383279502884197169399375105820974944592307816406286208998628034 82534211706798214808651328230664709384460955058223172535940812848111745028410270193852110 55596446229489549303819644288109756659334461284756482337867831652712019091456485669234603 48610454326648213393607260249141273724587006606315588174881520920962829254091715364367892 59036001133053054882046652138414695194151160943305727036575959195309218611738193261179310 51185480744623799627495673518857527248912279381830119491298336733624406566430860213949463 95224737190702179860943702770539217176293176752384674818467669405132000568127145263560827 78577134275778960917363717872146844090122495343014654958537105079227968925892354201995611 21290219608640344181598136297747713099605187072113499999983729780499510597317328160963185 95024459455346908302642522308253344685035261931188171010003137838752886587533208381420617 17766914730359825349042875546873115956286388235378759375195778185778053217122680661300192 78766111959092164201989380952572010654858632788659361533818279682303019520353018529689957 73622599413891249721775283479131515574857242454150695950829533116861727855889075098381754 63746493931925506040092770167113900984882401285836160356370766010471018194295559619894676 78374494482553797747268471040475346462080466842590694912933136770289891521047521620569660 24058038150193511253382430035587640247496473263914199272604269922796782354781636009341721 64121992458631503028618297455570674983850549458858692699569092721079750930295532116534498 72027559602364806654991198818347977535663698074265425278625518184175746728909777727938000 81647060016145249192173217214772350141441973568548161361157352552133475741849468438523323 90739414333454776241686251898356948556209921922218427255025425688767179049460165346680498 86272327917860857843838279679766814541009538837863609506800642251252051173929848960841284 88626945604241965285022210661186306744278622039194945047123713786960956364371917287467764 6575739624138908658326459958133904780275901

计划任务：周期性做什么事情 crontab cronjob简称为cj \[root@kmaster job]# kubectl get cj No resources found in default namespace. \[root@kmaster job]# kubectl create cj --help

和linux一样：如果不考虑某个时间单位，可以使用\*表示。 分 时 天 月 周 几分/几点/几号/几月/星期几

上午9点整，怎么表示？

*
  *
    *
      *
        * 对吗？

\*/1 9 \* \* \* 每隔1分钟执行

\[root@kmaster job]# kubectl create cronjob my-job --image=busybox --schedule="\*/1 \* \* \* \*" --dry-run=client -o yaml -- sh -c "echo $(date "+%Y-%m-%d %H:%M:%S")" > myjob3.yaml

\[root@kmaster job]# cat myjob.yaml apiVersion: batch/v1 kind: CronJob metadata: creationTimestamp: null name: my-job spec: jobTemplate: metadata: creationTimestamp: null name: my-job spec: template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - echo $(date "+%Y-%m-%d %H:%M:%S") image: busybox imagePullPolicy: IfNotPresent name: my-job resources: {} restartPolicy: OnFailure schedule: '\*/1 \* \* \* \*' status: {}

\[root@kmaster job]# kubectl apply -f myjob3.yaml cronjob.batch/my-job created \[root@kmaster job]# kubectl get cj NAME SCHEDULE SUSPEND ACTIVE LAST SCHEDULE AGE my-job \*/1 \* \* \* \* False 0 4s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE my-job-27969382-q9cgl 0/1 ContainerCreating 0 1s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE my-job-27969382-q9cgl 0/1 Completed 0 6s \[root@kmaster job]# kubectl logs my-job-27969382-q9cgl 2023-03-07 04:22:03

记录：为什么cronjob周期性任务只保留3个。 https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/cron-jobs/ 任务历史限制 .spec.successfulJobsHistoryLimit 和 .spec.failedJobsHistoryLimit 字段是可选的。 这两个字段指定应保留多少已完成和失败的任务。 默认设置分别为 3 和 1。将限制设置为 0 代表相应类型的任务完成后不会保留。

创建一个pod 完成 创建一个pod 完成 创建一个pod 完成

自动清理完成的 Job

\[root@kmaster job]# vim myjob4.yaml \[root@kmaster job]# cat myjob4.yaml apiVersion: batch/v1 kind: CronJob metadata: creationTimestamp: null name: my-job spec: jobTemplate: metadata: creationTimestamp: null name: my-job spec: ttlSecondsAfterFinished: 60 template: metadata: creationTimestamp: null spec: containers: - command: - sh - -c - echo $(date "+%Y-%m-%d %H:%M:%S") image: busybox imagePullPolicy: IfNotPresent name: my-job resources: {} restartPolicy: OnFailure schedule: '\*/1 \* \* \* \*' status: {}

\[root@kmaster job]# kubectl apply -f myjob4.yaml cronjob.batch/my-job created \[root@kmaster job]# kubectl get cj NAME SCHEDULE SUSPEND ACTIVE LAST SCHEDULE AGE my-job \*/1 \* \* \* \* False 0 3s

\[root@kmaster job]# kubectl get pod --刚开始创建，不着急，等待一会 No resources found in default namespace. \[root@kmaster job]# kubectl get pod No resources found in default namespace. \[root@kmaster job]# kubectl get pod No resources found in default namespace. \[root@kmaster job]# kubectl get pod No resources found in default namespace. \[root@kmaster job]# kubectl get cj NAME SCHEDULE SUSPEND ACTIVE LAST SCHEDULE AGE my-job \*/1 \* \* \* \* False 0 25s

\[root@kmaster job]# kubectl get pod --开始创建pod NAME READY STATUS RESTARTS AGE my-job-27969389-r92ds 0/1 ContainerCreating 0 0s \[root@kmaster job]# kubectl get pod NAME READY STATUS RESTARTS AGE my-job-27969389-r92ds 0/1 Completed 0 7s

\[root@kmaster job]# kubectl logs my-job-27969389-r92ds --完成之后查询 2023-03-07 04:29:00

\[root@kmaster job]# kubectl get pod --之后等待60秒，可以看到，新的pod被创建，而之前的pod被删除了。

NAME READY STATUS RESTARTS AGE my-job-27969390-6rccf 0/1 Completed 0 18s

\[root@kmaster job]# kubectl logs my-job-27969389-r92ds Error from server (NotFound): pods "my-job-27969389-r92ds" not found
