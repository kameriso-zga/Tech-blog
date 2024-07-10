# Volcano 相关

## 1. 背景

### 1.1 关于Volcano


Volcano 是一个开源的 Kubernetes 原生批处理系统，专为高性能、高吞吐量的数据处理和机器学习工作负载设计。它提供了一套丰富的特性来支持复杂的大规模计算工作负载，比如 AI、大数据处理和科学计算。Volcano 通过增强 Kubernetes 现有资源调度能力，使其更适合运行高性能计算（HPC）和 AI 工作负载。

#### 1.1.1 主要特点

1. **高级作业调度**：Volcano 通过引入作业（Job）、队列（Queue）等概念，支持多任务、作业优先级、作业依赖等高级调度功能，帮助用户更灵活地管理作业执行顺序和资源分配。

2. **资源共享与隔离**：通过队列（Queue）机制，Volcano 支持不同用户或项目组之间的资源共享与隔离，确保关键任务可以优先获取资源，同时也保障了系统资源的高效利用。

3. **弹性调度**：Volcano 支持基于作业的弹性伸缩，可以根据作业的实际需求自动调整分配给作业的资源数量。

4. **Batch Scheduling** Volcano 提供了一套完善的批处理调度框架，支持基于 Kubernetes 的批处理作业调度，包括作业生命周期管理、作业级别的资源请求和限制等功能。


#### 1.1.2 应用场景

   1.**机器学习与深度学习**：利用 Volcano 的高级调度特性和资源管理能力，有效管理机器学习和深度学习任务的资源分配和作业队列，加速模型训练和实验周期。

   2.**大数据处理**：对于需要处理大量数据的作业，Volcano 可以保证计算资源的高效分配，加快数据处理速度，提高系统吞吐量。

   3.**科学计算**：支持科学研究中的大规模计算任务，如基因测序、物理模拟等，通过优化资源调度，缩短研究结果的输出时间。

### 1.2 基本概念

#### 1.2.1 queue （队列）

queue是容纳一组**podgroup**的队列，也是该组podgroup获取集群资源的划分依据

##### 样例

```shell
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  creationTimestamp: "2020-08-10T11:54:36Z"
  generation: 1
  name: default
  resourceVersion: "559"
  selfLink: /apis/scheduling.volcano.sh/v1beta1/queues/default
  uid: 14082e4c-bef6-4248-a414-1e06d8352bf0
spec:
  reclaimable: true
  weight: 1
  capability:
    cpu: "4"
    memory: "4096Mi"
status:
  state: Open
```

##### 关键字段

- weight

weight表示该queue在集群资源划分中所占的**相对**比重，该queue应得资源总量为 **(weight/total-weight) \* total-resource**。其中， total-weight表示所有的queue的weight总和，total-resource表示集群的资源总量。weight是一个**软约束**，取值范围为[1, 2^31-1]

- capability

capability表示该queue内所有podgroup使用资源量之和的上限，它是一个**硬约束**

- reclaimable

reclaimable表示该queue在资源使用量超过该queue所应得的资源份额时，是否允许其他queue回收该queue使用超额的资源，默认值为**true**

##### 资源状态

- Open

该queue当前处于可用状态，可接收新的podgroup

- Closed

该queue当前处于不可用状态，不可接收新的podgroup

- Closing

该Queue正在转化为不可用状态，不可接收新的podgroup

- Unknown

该queue当前处于不可知状态，可能是网络或其他原因导致queue的状态暂时无法感知

#### 1.2.2 vcjob （Volcano Job）

Volcano Job，简称vcjob，是Volcano自定义的Job资源类型。区别于Kubernetes Job，vcjob提供了更多高级功能，如可指定调度器、支持最小运行pod数、 支持task、支持生命周期管理、支持指定队列、支持优先级调度等。Volcano Job更加适用于机器学习、大数据、科学计算等高性能计算场景。

##### 样例

```shell
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: test-job
spec:
  minAvailable: 3
  schedulerName: volcano
  priorityClassName: high-priority
  policies:
    - event: PodEvicted
      action: RestartJob
  plugins:
    ssh: []
    env: []
    svc: []
  maxRetry: 5
  queue: default
  volumes:
    - mountPath: "/myinput"
    - mountPath: "/myoutput"
      volumeClaimName: "testvolumeclaimname"
      volumeClaim:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: "my-storage-class"
        resources:
          requests:
            storage: 1Gi
  tasks:
    - replicas: 6
      name: "default-nginx"
      template:
        metadata:
          name: web
        spec:
          containers:
            - image: nginx
              imagePullPolicy: IfNotPresent
              name: nginx
              resources:
                requests:
                  cpu: "1"
          restartPolicy: OnFailure
```

##### 关键字段

- schedulerName

schedulerName表示该job的pod所使用的调度器，默认值为volcano，也可指定为default-scheduler。它也是tasks.template.spec.schedulerName的默认值。

- minAvailable

minAvailable表示运行该job所要运行的**最少**pod数量。只有当job中处于running状态的pod数量不小于minAvailable时，才认为该job运行正常。

- volumes

volumes表示该job的挂卷配置。volumes配置遵从kubernetes volumes配置要求。

- tasks.replicas

tasks.replicas表示某个task pod的副本数。

- tasks.template

tasks.template表示某个task pod的具体配置定义。

- tasks.policies

tasks.policies表示某个task的生命周期策略。

- policies

policies表示job中所有task的默认生命周期策略，在tasks.policies不配置时使用该策略。

- plugins

plugins表示该job在调度过程中使用的插件。

- queue

queue表示该job所属的队列。

- priorityClassName

priorityClassName表示该job优先级，在抢占调度和优先级排序中生效。

- maxRetry

maxRetry表示当该job可以进行的最大重启次数。

##### 资源状态

- pending

pending表示job还在等待调度中，处于排队的状态。

- aborting

aborting表示job因为某种外界原因正处于中止状态，即将进入aborted状态。

- aborted

aborted表示job因为某种外界原因已处于中止状态。

- running

running表示job中至少有minAvailable个pod正在运行状态。

- restarting

restarting表示job正处于重启状态，正在中止当前的job实例并重新创建新的实例。

- completing

completing表示job中至少有minAvailable个数的task已经完成，该job正在进行最后的清理工作。

- completed

completing表示job中至少有minAvailable个数的task已经完成，该job已经完成了最后的清理工作。

- terminating

terminating表示job因为某种内部原因正处于终止状态，正在等到pod或task释放资源。

- terminated

terminated表示job因为某种内部原因已经处于终止状态，job没有达到预期就结束了。

- failed

failed表示job经过了maxRetry次重启，依然没有正常启动。

##### 使用场景

- TensorFlow workload (详见2.3 tensorflow示例)

#### 1.2.3 podgroup

podgroup是一组强关联pod的集合，主要用于批处理工作负载场景，比如Tensorflow中的一组ps和worker。它是volcano自定义资源类型。

##### 样例

```shell
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  creationTimestamp: "2020-08-11T12:28:55Z"
  generation: 5
  name: test
  namespace: default
  ownerReferences:
  - apiVersion: batch.volcano.sh/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Job
    name: test
    uid: 028ecfe8-0ff9-477d-836c-ac5676491a38
  resourceVersion: "109074"
  selfLink: /apis/scheduling.volcano.sh/v1beta1/namespaces/default/podgroups/job-1
  uid: eb2508f5-3349-439c-b94d-4ac23afd71ff
spec:
  minMember: 1
  minResources:
    cpu: "3"
    memory: "2048Mi"
  priorityClassName: high-prority
  queue: default
status:
  conditions:
  - lastTransitionTime: "2020-08-11T12:28:57Z"
    message: '1/0 tasks in gang unschedulable: pod group is not ready, 1 minAvailable.'
    reason: NotEnoughResources
    status: "True"
    transitionID: 77d5be3f-6169-4f86-8e65-0bdc621ce983
    type: Unschedulable
  - lastTransitionTime: "2020-08-11T12:29:02Z"
    reason: tasks in gang are ready to be scheduled
    status: "True"
    transitionID: 54514401-5c90-4b11-840d-90c1cda93096
    type: Scheduled
  phase: Running
  running: 1
```

##### 关键字段

- minMember

minMember表示该podgroup下**最少**需要运行的pod或任务数量。如果集群资源不满足miniMember数量任务的运行需求，调度器将不会调度任何一个该podgroup 内的任务。

- queue

queue表示该podgroup所属的queue。queue必须提前已创建且状态为open。

- priorityClassName

priorityClassName表示该podgroup的优先级，用于调度器为该queue中所有podgroup进行调度时进行排序。**system-node-critical**和**system-cluster-critical** 是2个预留的值，表示最高优先级。不特别指定时，默认使用default优先级或zero优先级。

- minResources

minResources表示运行该podgroup所需要的最少资源。当集群可分配资源不满足minResources时，调度器将不会调度任何一个该podgroup内的任务。

- phase

phase表示该podgroup当前的状态。

- conditions

conditions表示该podgroup的具体状态日志，包含了podgroup生命周期中的关键事件。

- running

running表示该podgroup中当前处于running状态的pod或任务的数量。

- succeed

succeed表示该podgroup中当前处于succeed状态的pod或任务的数量。

- failed

failed表示该podgroup中当前处于failed状态的pod或任务的数量。

##### 资源状态

![image](https://volcano.sh/img/status-DAG.png)

- pending

pending表示该podgroup已经被volcano接纳，但是集群资源暂时不能满足它的需求。一旦资源满足，该podgroup将转变为running状态。

- running

running表示该podgroup至少有**minMember**个pod或任务处于running状态。

- unknown

unknown表示该podgroup中**minMember**数量的pod或任务分为2种状态，部分处于running状态，部分没有被调度。没有被调度的原因可能是资源不够等。调度 器将等待controller重新拉起这些pod或任务。

- inqueue

inqueue表示该podgroup已经通过了调度器的校验并入队，即将为它分配资源。inqueue是一种处于pending和running之间的中间状态。

## 2. 步骤

### 2.1 Helm 部署 Volcano

``` bash
helm repo add volcano-sh https://volcano-sh.github.io/helm-charts
helm repo update 
helm install volcano volcano-sh/volcano -n volcano-system --create-namespace
```

检查安装状态

``` bash
[root@localhost volcano-example]# kubectl get pods -n volcano-system 
NAME                                   READY   STATUS      RESTARTS   AGE
volcano-admission-6f66476fd8-cgkkg     1/1     Running     0          157m
volcano-admission-init-5qtt8           0/1     Completed   0          157m
volcano-controllers-6745ddbc55-jxcp5   1/1     Running     0          157m
volcano-scheduler-7878844b4f-swztb     1/1     Running     0          157m
```

### 2.2 基本使用方法

#### 2.2.1 创建一个名为“test”的queue（队列）

``` bash
touch test.yaml
vim test.yaml
```

``` bash
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: test
spec:
  weight: 1
  reclaimable: false
  capability:
    cpu: 2
```

``` bash
kubectl apply -f test.yaml
```

经过上述操作，容器云环境中会生成一个叫做test 的 queue（队列）资源

``` bash
[root@localhost volcano-example]# kubectl get queue
NAME      AGE
default   160m
test      21m
```

#### 2.2.2. 创建一个叫job-1的vcjob（volcano job）

``` bash
touch job-1.yaml
vim job-1.yaml
```

``` bash
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: job-1
spec:
  minAvailable: 1
  schedulerName: volcano
  queue: test
  policies:
    - event: PodEvicted
      action: RestartJob
  tasks:
    - replicas: 1
      name: nginx
      policies:
      - event: TaskCompleted
        action: CompleteJob
      template:
        spec:
          containers:
            - command:
              - sleep
              - 10m
              image: nginx:latest ## 可以看到，这个测试的vcjob执行了一个nginx的镜像实例
              name: nginx
              resources:
                requests:
                  cpu: 1
                limits:
                  cpu: 1
          restartPolicy: Never
```

``` bash
kubectl apply -f job-1.yaml
```

验证vcjob的部署状态

``` bash
[root@localhost volcano-example]# kubectl get vcjob 
NAME    STATUS      MINAVAILABLE   RUNNINGS   AGE
job-1   Completed   1                         20m
[root@localhost volcano-example]# kubectl describe vcjob job-1 
Name:         job-1
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  batch.volcano.sh/v1alpha1
Kind:         Job
Metadata:
  Creation Timestamp:  2024-03-15T05:59:15Z
  Generation:          1
  Managed Fields:
    API Version:  batch.volcano.sh/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:minAvailable:
        f:policies:
        f:queue:
        f:schedulerName:
        f:tasks:
    Manager:      kubectl-client-side-apply
    Operation:    Update
    Time:         2024-03-15T05:59:15Z
    API Version:  batch.volcano.sh/v1alpha1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
        f:minAvailable:
        f:runningDuration:
        f:state:
          .:
          f:lastTransitionTime:
          f:phase:
        f:succeeded:
        f:taskStatusCount:
          .:
          f:nginx:
            .:
            f:phase:
              .:
              f:Succeeded:
        f:version:
    Manager:         Go-http-client
    Operation:       Update
    Subresource:     status
    Time:            2024-03-15T06:09:29Z
  Resource Version:  1127907
  UID:               3d0814db-4243-4de0-8e93-a08fc103b9ea
Spec:
  Max Retry:      3
  Min Available:  1
  Policies:
    Action:        RestartJob
    Event:         PodEvicted
  Queue:           test
  Scheduler Name:  volcano
  Tasks:
    Max Retry:      3
    Min Available:  1
    Name:           nginx
    Policies:
      Action:  CompleteJob
      Event:   TaskCompleted
    Replicas:  1
    Template:
      Metadata:
      Spec:
        Containers:
          Command:
            sleep
            10m
          Image:  nginx:latest
          Name:   nginx
          Resources:
            Limits:
              Cpu:  1
            Requests:
              Cpu:       1
        Restart Policy:  Never
Status:
  Conditions:
    Last Transition Time:  2024-03-15T05:59:15Z
    Status:                Pending
    Last Transition Time:  2024-03-15T05:59:27Z
    Status:                Running
    Last Transition Time:  2024-03-15T06:09:29Z
    Status:                Completing
    Last Transition Time:  2024-03-15T06:09:29Z
    Status:                Completed
  Min Available:           1
  Running Duration:        10m14.475393813s
  State:
    Last Transition Time:  2024-03-15T06:09:29Z
    Phase:                 Completed
  Succeeded:               1
  Task Status Count:
    Nginx:
      Phase:
        Succeeded:  1
  Version:          3
Events:
  Type     Reason           Age                 From                   Message
  ----     ------           ----                ----                   -------
  Warning  PodGroupPending  10m (x10 over 20m)  vc-controller-manager  PodGroup default:job-1 unschedule,reason: 1/0 tasks in gang unschedulable: pod group is not ready, 1 minAvailable
  Normal   ExecuteAction    10m                 vc-controller-manager  Start to execute action CompleteJob

```

``` bash
[root@localhost volcano-example]# kubectl get vcjob job-1 -oyaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"batch.volcano.sh/v1alpha1","kind":"Job","metadata":{"annotations":{},"name":"job-1","namespace":"default"},"spec":{"minAvailable":1,"policies":[{"action":"RestartJob","event":"PodEvicted"}],"queue":"test","schedulerName":"volcano","tasks":[{"name":"nginx","policies":[{"action":"CompleteJob","event":"TaskCompleted"}],"replicas":1,"template":{"spec":{"containers":[{"command":["sleep","10m"],"image":"nginx:latest","name":"nginx","resources":{"limits":{"cpu":1},"requests":{"cpu":1}}}],"restartPolicy":"Never"}}}]}}
  creationTimestamp: "2024-03-15T05:59:15Z"
  generation: 1
  name: job-1
  namespace: default
  resourceVersion: "1127907"
  uid: 3d0814db-4243-4de0-8e93-a08fc103b9ea
spec:
  maxRetry: 3
  minAvailable: 1
  policies:
  - action: RestartJob
    event: PodEvicted
  queue: test
  schedulerName: volcano
  tasks:
  - maxRetry: 3
    minAvailable: 1
    name: nginx
    policies:
    - action: CompleteJob
      event: TaskCompleted
    replicas: 1
    template:
      metadata: {}
      spec:
        containers:
        - command:
          - sleep
          - 10m
          image: nginx:latest
          name: nginx
          resources:
            limits:
              cpu: "1"
            requests:
              cpu: "1"
        restartPolicy: Never
status:
  conditions:
  - lastTransitionTime: "2024-03-15T05:59:15Z"
    status: Pending
  - lastTransitionTime: "2024-03-15T05:59:27Z"
    status: Running
  - lastTransitionTime: "2024-03-15T06:09:29Z"
    status: Completing
  - lastTransitionTime: "2024-03-15T06:09:29Z"
    status: Completed
  minAvailable: 1
  runningDuration: 10m14.475393813s
  state:
    lastTransitionTime: "2024-03-15T06:09:29Z"
    phase: Completed
  succeeded: 1
  taskStatusCount:
    nginx:
      phase:
        Succeeded: 1
  version: 3
```

检查podgroup的状态

``` bash
[root@localhost volcano-example]# kubectl get podgroup -A 
NAMESPACE   NAME                                         STATUS    MINMEMBER   RUNNINGS   AGE
default     job-1-cf43dde4-1158-4f86-8b01-a9a985256756   Running   1           1          41s
[root@localhost volcano-example]# kubectl get podgroup job-1-cf43dde4-1158-4f86-8b01-a9a985256756 -oyaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"batch.volcano.sh/v1alpha1","kind":"Job","metadata":{"annotations":{},"name":"job-1","namespace":"default"},"spec":{"minAvailable":1,"policies":[{"action":"RestartJob","event":"PodEvicted"}],"queue":"test","schedulerName":"volcano","tasks":[{"name":"nginx","policies":[{"action":"CompleteJob","event":"TaskCompleted"}],"replicas":1,"template":{"spec":{"containers":[{"command":["sleep","10m"],"image":"nginx:latest","name":"nginx","resources":{"limits":{"cpu":1},"requests":{"cpu":1}}}],"restartPolicy":"Never"}}}]}}
  creationTimestamp: "2024-03-15T06:37:57Z"
  generation: 4
  name: job-1-cf43dde4-1158-4f86-8b01-a9a985256756
  namespace: default
  ownerReferences:
  - apiVersion: batch.volcano.sh/v1alpha1
    blockOwnerDeletion: true
    controller: true
    kind: Job
    name: job-1
    uid: cf43dde4-1158-4f86-8b01-a9a985256756
  resourceVersion: "1140147"
  uid: 07dac98b-65a2-42c0-811b-86986dedf5a2
spec:
  minMember: 1
  minResources:
    count/pods: "1"
    cpu: "1"
    limits.cpu: "1"
    pods: "1"
    requests.cpu: "1"
  minTaskMember:
    nginx: 1
  queue: test
status:
  conditions:
  - lastTransitionTime: "2024-03-15T06:37:58Z"
    message: '1/0 tasks in gang unschedulable: pod group is not ready, 1 minAvailable'
    reason: NotEnoughResources
    status: "True"
    transitionID: 0fd628d2-59cb-465b-8d62-7db7b7e0f497
    type: Unschedulable
  - lastTransitionTime: "2024-03-15T06:38:05Z"
    reason: tasks in gang are ready to be scheduled
    status: "True"
    transitionID: 0d7269a3-a5d2-429e-9749-3f3d89e60525
    type: Scheduled
  phase: Running
  running: 1
```

现在已经有工作负载运行在名称为test的queue里面了，我们可以检查此队列的状态

``` bash
[root@localhost volcano-example]# kubectl get queue test -oyaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"scheduling.volcano.sh/v1beta1","kind":"Queue","metadata":{"annotations":{},"name":"test"},"spec":{"capability":{"cpu":2},"reclaimable":false,"weight":1}}
  creationTimestamp: "2024-03-15T05:52:28Z"
  generation: 1
  name: test
  resourceVersion: "1140091"
  uid: 01d1320e-ccea-4cac-a9b2-5093e6446020
spec:
  capability:
    cpu: 2
  reclaimable: false
  weight: 1
status:
  allocated:
    cpu: "1"
    memory: "0"
    pods: "1"
  reservation: {}
  running: 1
  state: Open
```

### 2.3 Tensorflow 示例

#### 2.3.1 介绍

[参数服务器训练](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-li_mu.pdf)是一种常见的数据并行方法，用于在多台机器上扩展模型训练。

参数服务器训练集群由 *工作进程Worker* (worker) 和 *参数服务器ParameterServer* (ps) 组成。变量在参数服务器上创建，并在每个步骤中由工作进程读取和更新。默认情况下，工作进程会独立读取和更新这些变量，而不会彼此同步。因此，参数服务器式训练有时也称为*异步训练*。

在 TensorFlow 2 中，参数服务器训练由 [`tf.distribute.ParameterServerStrategy`](https://www.tensorflow.org/api_docs/python/tf/distribute/experimental/ParameterServerStrategy?hl=zh-cn) 类提供支持，该类会将训练步骤分发到一个集群，该集群可扩展到数千个工作进程（伴随着参数服务器）。

在使用参数服务器训练时，建议具有：

- 一个*协调器*作业（作业名称为 `chief`）
- 多个*工作进程*作业（作业名称为 `worker`）(本示例中创建2个worker)
- 多个*参数服务器*作业（作业名称为 `ps`）（本示例中创建2个ps）

*协调器*会创建资源、调度训练任务、编写检查点并处理任务失败。*工作进程*和*参数服务器*会运行 [`tf.distribute.Server`](https://www.tensorflow.org/api_docs/python/tf/distribute/Server?hl=zh-cn) 实例来监听来自协调器的请求。

#### 2.3.2 安装准备

本示例要求准备已经安装好显卡及*Nvidia-GPU-Operator*的k8s环境，将物理显卡进行分时分割，并且安装好Volcano调度器。

本示例要求已经配置好镜像仓库，包括docker login和docker push的操作，以进行镜像推送。

本示例代码库地址位于(https://gitlab.bingosoft.net/bingomatrix/example/volcano-start/tree/develop)内

#### 2.3.3 Volcano准备

###### 2.3.3.1 创建带有GPU资源的queue（队列）

```bash
git clone https://gitlab.bingosoft.net/bingomatrix/example/volcano-start/tree/develop)
cd volcano-start
```

``` bash
[root@master volcano-start]# cat newqueue.yaml # 查看newqueue.yaml 文档
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: test #新建名称为test的queue
spec:
  weight: 1
  reclaimable: false
  capability:
    cpu: "40"
    memory: "40960Mi"
    nvidia.com/gpu: "100" #请求节点的GPU资源
[root@master volcano-start]# kubectl apply -f newqueue.yaml #创建队列 
```

###### 2.3.3.2 配置tensorflow分布式训练集群配置TF_CONFIG

``` bash
[root@master volcano-start]# cat volcano/chief.host 
tensorflow-distributed-job-chief-0.tensorflow-distributed-job #chief容器的服务地址
[root@master volcano-start]# cat volcano/ps.host
tensorflow-distributed-job-ps-0.tensorflow-distributed-job 
tensorflow-distributed-job-ps-1.tensorflow-distributed-job #ps容器的服务地址
[root@master volcano-start]# cat volcano/worker.host
tensorflow-distributed-job-worker-0.tensorflow-distributed-job
tensorflow-distributed-job-worker-1.tensorflow-distributed-job #worker容器的服务地址
[root@master volcano-start]# cp -r volcano /etc/ #将文件夹复制到/etc 目录
```

###### 2.3.3.3 进行镜像构建

查看```mnist.py```脚本

``` bash
[root@master volcano-start]# cat mnist3.py 
import os
import gzip
import numpy as np
import json
import tensorflow as tf
import logging

# Enable GPU memory growth
# 避免cuda驱动耗尽GPU显存的限制
gpus = tf.config.experimental.list_physical_devices('GPU')
if gpus:
  try:
    for gpu in gpus:
      tf.config.experimental.set_memory_growth(gpu, True)
  except RuntimeError as e:
    print(e)

from tensorflow.keras.layers import Dense, Flatten
from tensorflow.keras.models import Sequential
from tensorflow.keras.optimizers import Adam
from tensorflow.keras.losses import SparseCategoricalCrossentropy
from tensorflow.keras.metrics import SparseCategoricalAccuracy

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s %(levelname)s: %(message)s',
                    handlers=[
                        logging.FileHandler("mnist-log.txt"),
                        logging.StreamHandler()
                    ])

#获取tensorflow分布式训练集群配置TF_CONFIG
TF_CONFIG = json.loads(os.environ.get('TF_CONFIG', '{}'))
logging.info(f"TF_CONFIG: {TF_CONFIG}")

cluster_resolver = tf.distribute.cluster_resolver.TFConfigClusterResolver()
logging.info(f"Cluster Resolver: {cluster_resolver}")

#如果容器收到的TF_CONFIG指定他的角色为worker或者ps，加入集群
if cluster_resolver.task_type in ("worker", "ps"):
    server = tf.distribute.Server(cluster_resolver.cluster_spec(),
                                  job_name=cluster_resolver.task_type,
                                  task_index=cluster_resolver.task_id,
                                  protocol="grpc",
                                  start=True)
    server.join()
    
#如果容器收到的TF_CONFIG指定他的角色为chief，创建集群，带领进行训练
elif cluster_resolver.task_type == "chief":
    # Use ParameterServerStrategy for the 'chief' role
    strategy = tf.distribute.experimental.ParameterServerStrategy(cluster_resolver)
    logging.info("ParameterServerStrategy is initialized.")

    # Define the model training and evaluation inside the strategy's scope
    with strategy.scope():
        # 导入MNIST训练数据
        def load_mnist_data(data_directory):
            logging.info("Loading MNIST data from gzipped files.")
            files = [
                'train-images-idx3-ubyte.gz', 'train-labels-idx1-ubyte.gz',
                't10k-images-idx3-ubyte.gz', 't10k-labels-idx1-ubyte.gz'
            ]
            paths = [os.path.join(data_directory, file_name) for file_name in files]

            with gzip.open(paths[0], 'rb') as imgpath:
                x_train = np.frombuffer(imgpath.read(), np.uint8, offset=16).reshape(-1, 28, 28)
            with gzip.open(paths[1], 'rb') as lblpath:
                y_train = np.frombuffer(lblpath.read(), np.uint8, offset=8)
            return x_train, y_train
        
        # 训练
        def make_datasets():
            x_train, y_train = load_mnist_data('./mnist-data')
            x_train = x_train / 255.0
            return tf.data.Dataset.from_tensor_slices((x_train, y_train)).batch(64).repeat()

        model = Sequential([
            Flatten(input_shape=(28, 28)),
            Dense(128, activation='relu'),
            Dense(10)
        ])

        model.compile(optimizer=Adam(),
                      loss=SparseCategoricalCrossentropy(from_logits=True),
                      metrics=['accuracy'])

        train_dataset = make_datasets()

        # Training steps need to be defined explicitly when using ClusterCoordinator
        steps_per_epoch = 60000 // 64  # Number of samples in the dataset / batch size

        # The model.fit call remains the same
        model.fit(train_dataset, epochs=5, steps_per_epoch=steps_per_epoch)

else:
    logging.error("Task type not recognized.")

logging.info("Script execution completed.")
```

查看Docker文件

```bash
[root@master volcano-start]# cat Dockerfile 
FROM tensorflow/tensorflow:latest-gpu #使用具有cuda gpu驱动支持的tensorflow
WORKDIR /app
ADD . /app
RUN export TF_CPP_MIN_LOG_LEVEL=0
ENTRYPOINT ["python", "mnist3.py"]
```

进行镜像构建和镜像推送

``` bash
docker build -t registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1 .
docker push registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1
```

###### 2.3.3.4 使用volcano调度器进行调度作业

查看 ```volcano1.yaml``` 文档

```bash
[root@master volcano-start]# cat volcano1.yml 
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: tensorflow-distributed-job
spec:
  minAvailable: 5  # 1 chief + 2 workers + 2 ps
  schedulerName: volcano
  plugins:
    env: [] 
    svc: []
  policies:
    - event: PodEvicted
      action: RestartJob
  queue: test
  tasks:
    - replicas: 1
      name: chief
      template:
        spec:
          containers:
            - command: #volcano调度器自动将/etc/volcano目录内的文件挂载到容器内，此yaml负载创建文件会在容器内寻找chief、ps、worker的各个负载service地址，并添加相应的端口（2222），按照容器工作类型界定task_type和task_index，生成容器运行所需的TF_CONFIG
                - sh
                - -c
                - |
                  CHIEF_HOST=`cat /etc/volcano/chief.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  PS_HOST=`cat /etc/volcano/ps.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  WORKER_HOST=`cat /etc/volcano/worker.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  export TF_CONFIG={\"cluster\":{\"chief\":[${CHIEF_HOST}],\"worker\":[${WORKER_HOST}],\"ps\":[${PS_HOST}]},\"task\":{\"type\":\"chief\",\"index\":0},\"environment\":\"cloud\"};
                  echo $TF_CONFIG >> /app/tfconfig.txt
                  python mnist3.py
              image: registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1
              name: tensorflow
              ports:
                - containerPort: 2222
                  name: tfjob-port
              resources:
                limits:
                 nvidia.com/gpu: 1 #请求节点的GPU资源
          restartPolicy: Never
    - replicas: 2
      name: worker
      template:
        spec:
          containers:
            - image: registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1
              name: tensorflow
              command:
                - sh
                - -c
                - |
                  CHIEF_HOST=`cat /etc/volcano/chief.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  PS_HOST=`cat /etc/volcano/ps.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  WORKER_HOST=`cat /etc/volcano/worker.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  export TF_CONFIG={\"cluster\":{\"chief\":[${CHIEF_HOST}],\"worker\":[${WORKER_HOST}],\"ps\":[${PS_HOST}]},\"task\":{\"type\":\"worker\",\"index\":${VK_TASK_INDEX}},\"environment\":\"cloud\"};
                  python mnist3.py
              ports:
                - containerPort: 2222
                  name: tfjob-port
              resources:
                limits:
                 nvidia.com/gpu: 1 #请求节点的GPU资源
          restartPolicy: Never
    - replicas: 2
      name: ps
      template:
        spec:
          containers:
            - image: registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1
              name: tensorflow
              command:
                - sh
                - -c
                - |
                  CHIEF_HOST=`cat /etc/volcano/chief.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  PS_HOST=`cat /etc/volcano/ps.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  WORKER_HOST=`cat /etc/volcano/worker.host | sed 's/$/&:2222/g' | sed 's/^/"/;s/$/"/' | tr "\n" ","`;
                  export TF_CONFIG={\"cluster\":{\"chief\":[${CHIEF_HOST}],\"worker\":[${WORKER_HOST}],\"ps\":[${PS_HOST}]},\"task\":{\"type\":\"ps\",\"index\":${VK_TASK_INDEX}},\"environment\":\"cloud\"};
                  python mnist3.py
              ports:
                - containerPort: 2222
                  name: tfjob-port
              resources:
                limits:
                 nvidia.com/gpu: 1 #请求节点的GPU资源
          restartPolicy: Never
```

创建调度任务

``` BASH
[root@master volcano-start]# kubectl apply -f volcano1.yml 
job.batch.volcano.sh/tensorflow-distributed-job created
```

查看容器状态

```bash
[root@master ~]# kubectl get pods 
NAME                                  READY   STATUS    RESTARTS   AGE
tensorflow-distributed-job-chief-0    1/1     Running   0          35s
tensorflow-distributed-job-ps-0       1/1     Running   0          35s
tensorflow-distributed-job-ps-1       1/1     Running   0          35s
tensorflow-distributed-job-worker-0   1/1     Running   0          35s
tensorflow-distributed-job-worker-1   1/1     Running   0          35s
```

查看显卡使用状态（以下环境安装了多张物理显卡，可以看到，volcano调度器将算力负担分散到了多张显卡）

```bash
[root@master ~]# nvidia-smi
Thu Mar 21 18:09:24 2024       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 545.23.08              Driver Version: 545.23.08    CUDA Version: 12.3     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  Tesla P100-PCIE-16GB           Off | 00000000:18:00.0 Off |                    0 |
| N/A   33C    P0              32W / 250W |  15852MiB / 16384MiB |      6%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
|   1  Tesla P100-PCIE-16GB           Off | 00000000:AF:00.0 Off |                    0 |
| N/A   33C    P0              33W / 250W |  14898MiB / 16384MiB |      6%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A   3585637      C   python                                    15476MiB |
|    0   N/A  N/A   3585638      C   python                                      374MiB |
|    1   N/A  N/A   3584465      C   python                                      374MiB |
|    1   N/A  N/A   3585107      C   python                                      886MiB |
|    1   N/A  N/A   3586105      C   python                                    13636MiB |
+---------------------------------------------------------------------------------------+
```

查看chief容器的日志，可证实训练的完成进度

``` bash
[root@master volcano-start]# kubectl logs -f tensorflow-distributed-job-chief-0
2024-03-21 10:08:30,226 INFO: TF_CONFIG: {'cluster': {'chief': ['tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222'], 'worker': ['tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222'], 'ps': ['tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222']}, 'task': {'type': 'chief', 'index': 0}, 'environment': 'cloud'}
2024-03-21 10:08:30,227 INFO: Cluster Resolver: <tensorflow.python.distribute.cluster_resolver.tfconfig_cluster_resolver.TFConfigClusterResolver object at 0x7f58517bcfd0>
INFO:tensorflow:`tf.distribute.experimental.ParameterServerStrategy` is initialized with cluster_spec: ClusterSpec({'chief': ['tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222'], 'ps': ['tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222'], 'worker': ['tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222']})
2024-03-21 10:08:30,228 INFO: `tf.distribute.experimental.ParameterServerStrategy` is initialized with cluster_spec: ClusterSpec({'chief': ['tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222'], 'ps': ['tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222'], 'worker': ['tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222']})
INFO:tensorflow:ParameterServerStrategyV2 is now connecting to cluster with cluster_spec: ClusterSpec({'chief': ['tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222'], 'ps': ['tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222'], 'worker': ['tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222']})
2024-03-21 10:08:30,229 INFO: ParameterServerStrategyV2 is now connecting to cluster with cluster_spec: ClusterSpec({'chief': ['tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222'], 'ps': ['tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222'], 'worker': ['tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222', 'tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222']})
2024-03-21 10:08:30.231388: I tensorflow/core/platform/cpu_feature_guard.cc:151] This TensorFlow binary is optimized with oneAPI Deep Neural Network Library (oneDNN) to use the following CPU instructions in performance-critical operations:  AVX2 AVX512F FMA
To enable them in other operations, rebuild TensorFlow with the appropriate compiler flags.
2024-03-21 10:08:30.948138: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1525] Created device /job:localhost/replica:0/task:0/device:GPU:0 with 14766 MB memory:  -> device: 0, name: Tesla P100-PCIE-16GB, pci bus id: 0000:af:00.0, compute capability: 6.0
2024-03-21 10:08:30.955381: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1525] Created device /job:chief/replica:0/task:0/device:GPU:0 with 14766 MB memory:  -> device: 0, name: Tesla P100-PCIE-16GB, pci bus id: 0000:af:00.0, compute capability: 6.0
2024-03-21 10:08:30.968380: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job chief -> {0 -> tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222}
2024-03-21 10:08:30.968435: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job ps -> {0 -> tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222, 1 -> tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222}
2024-03-21 10:08:30.968444: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job worker -> {0 -> tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222, 1 -> tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222}
2024-03-21 10:09:12.482265: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job chief -> {0 -> tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222}
2024-03-21 10:09:12.482315: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job ps -> {0 -> tensorflow-distributed-job-ps-0.tensorflow-distributed-job:2222, 1 -> tensorflow-distributed-job-ps-1.tensorflow-distributed-job:2222}
2024-03-21 10:09:12.482325: I tensorflow/core/distributed_runtime/rpc/grpc_channel.cc:272] Initialize GrpcChannelCache for job worker -> {0 -> tensorflow-distributed-job-worker-0.tensorflow-distributed-job:2222, 1 -> tensorflow-distributed-job-worker-1.tensorflow-distributed-job:2222}
2024-03-21 10:09:12.482786: I tensorflow/core/distributed_runtime/rpc/grpc_server_lib.cc:427] Started server with target: grpc://tensorflow-distributed-job-chief-0.tensorflow-distributed-job:2222
INFO:tensorflow:ParameterServerStrategy (CentralStorageStrategy if you are using a single machine) with compute_devices = ['/job:chief/replica:0/task:0/device:GPU:0'], variable_device = '/job:chief/replica:0/task:0/device:GPU:0'
2024-03-21 10:09:12,484 INFO: ParameterServerStrategy (CentralStorageStrategy if you are using a single machine) with compute_devices = ['/job:chief/replica:0/task:0/device:GPU:0'], variable_device = '/job:chief/replica:0/task:0/device:GPU:0'
INFO:tensorflow:Number of GPUs on workers: 1
2024-03-21 10:09:12,485 INFO: Number of GPUs on workers: 1
2024-03-21 10:09:12,485 INFO: ParameterServerStrategy is initialized.
2024-03-21 10:09:12,576 INFO: Loading MNIST data from gzipped files.
/usr/local/lib/python3.8/dist-packages/tensorflow/python/data/ops/dataset_ops.py:453: UserWarning: To make it possible to preserve tf.data options across serialization boundaries, their implementation has moved to be part of the TensorFlow graph. As a consequence, the options value is in general no longer known at graph construction time. Invoking this method in graph mode retains the legacy behavior of the original implementation, but note that the returned value might not reflect the actual value of the options.
  warnings.warn("To make it possible to preserve tf.data options across "
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:15,114 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:15,115 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:15,116 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:15,117 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:16,306 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:16,307 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:16,308 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
INFO:tensorflow:Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
2024-03-21 10:09:16,308 INFO: Reduce to /device:CPU:0 then broadcast to ('/replica:0/device:CPU:0',).
Epoch 1/5
937/937 - 10s - loss: 0.2852 - accuracy: 0.9196 - 10s/epoch - 10ms/step
Epoch 2/5
937/937 - 4s - loss: 0.1475 - accuracy: 0.9576 - 4s/epoch - 4ms/step
Epoch 3/5
937/937 - 4s - loss: 0.0974 - accuracy: 0.9721 - 4s/epoch - 4ms/step
Epoch 4/5
937/937 - 4s - loss: 0.0807 - accuracy: 0.9770 - 4s/epoch - 4ms/step
Epoch 5/5
937/937 - 4s - loss: 0.0506 - accuracy: 0.9865 - 4s/epoch - 4ms/step
2024-03-21 10:09:39,682 INFO: Script execution completed.
```

## 3.总结

本文讲述了volcano调度器的作用，他的几种自定义资源类型，他们的特征和使用方法示意，以及tensorflow parameter server的示例。

后续：

1. TF_CONFIG的自动生成方式尚未明确。示例中volcano的工作负载类型的创建示例中，TF_CONFIG的导入依赖手动添加的操作。后续需要明确其自动化生成的方式，或许需要依赖volcano的相关plugin机制。

2. 研究volcano与kubeflow的集成