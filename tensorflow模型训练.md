# tensorflow 图像识别 从demo 到AI框架测试

*版本信息*

*tensorflow  2.16.1*

*cuda  12.4*

*Python 3.9.9*

## 1..测试tensorflow 版本和cuda 兼容性 

测试版本信息

*tensorflow  2.16.1*

*cuda  12.6*

*Python 3.9.9*

- 使用工具：virtualenv，参考：https://virtualenv.pypa.io/en/latest/index.html
- tensoRT 参考：https://github.com/NVIDIA/TensorRT-LLM
- 参考tensorflow 官网关于版本信息：[Release TensorFlow 2.16.1 · tensorflow/tensorflow · GitHub](https://github.com/tensorflow/tensorflow/releases/tag/v2.16.1)

创建虚拟环境

```shell
virtualenv   --python /usr/bin/python3  venv
```

进入创建的虚拟环境

```shell
source venv/bin/activate
```

python 依赖包设置镜像源头

```shell
pip config set global.index-url https://pypi.mirrors.ustc.edu.cn/simple
```

安装python3 对应的依赖

```shell
pip install tensorflow 

pip install kras
...
```

python调用cuda GPU需要安装对应的依赖

```shell
pip install tensort
```

生成对应的requirement.txt

```shell
pip freeze > requirement.txt
```

基于requirement.txt 打镜像

```dockerfile
FROM python:3.9.9
WORKDIR /app
COPY . .
RUN pip config set global.index-url [Simple Index](https://pypi.mirrors.ustc.edu.cn/simple) RUN pip install -r requirements.txt --trusted-host https://pypi.mirrors.ustc.edu.cn/simpled-host
docker build -t registry.bingosoft.net/bingomatrix/base_tf_mnist .
```

运行

```shell
python3 main.py
```

查看nivida gpu

```shell
 watch -n0.1 nvidia-smi
```

退出python3 的虚拟环境

```shell
deactivate
```



**版本问题说明**：cuda 12.5 的版本，在使用tersorflow2.16对于Cuda版本支持描述：

```
TensorFlow pip packages are now built with CUDA 12.3 and cuDNN 8.9.7
```

tersorflow2.16对于cuda 12.5的支持需要安装对应的依赖包

```shell
pip install tf-nightly
```

- 解决方案：

变更tensorflow 和cuda 版本

```
CUDA Version: 12.3 
tensorflow 版本2.13.1
```

## 2.tensorflow 图像识别demo 挂载

下载tensorflow demo

```shell
wget https://github.com/toughmuffin12/TensorFlow-Digit-Recognizer.git
```

tensorflow demo 挂载（基于minst 数据集）

```dockerfile
FROM python:3.9.9
WORKDIR /app
COPY ../../Desktop/AI%20算力组文档 .
RUN pip config set global.index-url https://pypi.mirrors.ustc.edu.cn/simple
RUN pip install -r requirements.txt --trusted-host https://pypi.mirrors.ustc.edu.cn/simpled-host
```

```shell
 docker build -t registry.bingosoft.net/bingomatrix/base_tf_mnist .
```

推送镜像到远程仓库

```shell
docker login registry.bingosoft.net
docker push registry.bingosoft.net/bingomatrix/base_tf_mnist
```

创建pvc pv

**由于pod 内部网络不通，下载mnist.npz到本地，并且使用pvc 进行挂载**

```shell
wget https://storage.googleapis.com/tensorflow/tf-keras-datasets/mnist.npz
```

创建pvc

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mnt-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  volumeName: mnist-pv
  resources:
    requests:
      storage: 20Gi
```

创建pv

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mnist-pv
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/root/zjiajia/tensorflow-kube-file/mnt"
```

查看pvc 和pv 是否创建成功

![](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/1711349940.png)

创建pod 关联pvc minist数据集

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tf-demo 
spec:
  containers:
- name: gpu-container
  image: registry.bingosoft.net/bingomatrix/base_tf_mnist
  imagePullPolicy: IfNotPresent
  command: ["python3"]
  args: ["/app/main.py"] #执行容器内的main.py函数进行图像识别
  resources:
    limits:
      nvidia.com/gpu: 1 #请求节点的GPU资源
  volumeMounts:
  - name: mnt-data
    mountPath: /app/mnt #容器内的数据挂载路径
    volumes:
    - name: mnt-data
      persistentVolumeClaim:
      claimName: tf-data-pvc
```

查看是否创建成功

```shell
kubectl logs -f tf-demo
```

查看nivida gpu 显卡状态

```shell
watch -n0.1 nvidia-smi
```

## 3.运用AI 训练框架Volcano 训练tensorflow 

- 参考volcano官方文档：https://volcano.sh/zh/docs/
- 参考volcano集成tensorflow：https://gitlab.bingosoft.net/bingomatrix/example/volcano-start/tree/develop

本机下载volcano-start 

```shell
git clone git@gitlab.bingosoft.net:bingomatrix/example/volcano-start.git
git checkout develop
```

拷贝到远程的服务器

```shell
scp -r volcano-start root@10.16.0.1:/root/zjaja
```

进入到volcano-start 目录

```shell
cd volcano-start 
```

创建队列

```shell
 kubectl apply -f newqueue.yaml
```

将文件复制到etc目录

```shell
cp -r volcano /etc
```

如果本地或者远程有镜像不需要构建，没有的话需要构建

```shell
docker build -t registry.bingosoft.net/bingomatrix/distributed-tensorflow-volcano:0.1
```

登陆远程的harbor仓库

```shell
#先docker login,并按照提示填写用户名和密码
docker login registry.bingosoft.net
```

推送到远程的镜像仓库

```shell
docker push registry.bingosoft.net/bingomatrix/distributed-tensorflow:0.0.1
```

查看volcano1.yml文件

```yaml
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

使用volcano调度器进行调度作业,调度yaml 文件：volcano1.yml 

```shell
kubectl apply -f volcano1.yml 
```

查看调度作业是否有被成功创建

```shell
[root@master volcano-start]# kubectl get pods
NAME                                  READY   STATUS      RESTARTS   AGE
csi-nfs-test-connection               0/1     Error       0          3d22h
job-test-nginx-0                      0/1     Completed   0          6h6m
tensorflow-distributed-job-chief-0    1/1     Running     0          94m
tensorflow-distributed-job-ps-0       1/1     Running     0          94m
tensorflow-distributed-job-ps-1       1/1     Running     0          94m
tensorflow-distributed-job-worker-0   1/1     Running     0          94m
tensorflow-distributed-job-worker-1   1/1     Running     0          94m
```

查看nivida GPU显卡状态

```shell
[root@master volcano-start]# nvidia-smi
Tue Mar 26 15:39:46 2024       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 545.23.08              Driver Version: 545.23.08    CUDA Version: 12.3     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce RTX 2080 Ti     Off | 00000000:D8:00.0 Off |                  N/A |
| 32%   50C    P2              48W / 250W |   1543MiB / 11264MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A   1065593      C   python                                      308MiB |
|    0   N/A  N/A   1065598      C   python                                      308MiB |
|    0   N/A  N/A   1065994      C   python                                      308MiB |
|    0   N/A  N/A   1066283      C   python                                      308MiB |
|    0   N/A  N/A   1066284      C   python                                      308MiB |
+---------------------------------------------------------------------------------------+
```

