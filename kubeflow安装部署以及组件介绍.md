# 一、kubeflow环境安装部署

## 1.安装依赖

### 1.1操作系统

- openEuler2202(X86)

### 1.2.K8s集群

- 1.22.17版本(自行安装)

### 1.3.Kustomize：

- 3.2.0版本 ,[Kustomize](https://github.com/kubernetes-sigs/kustomize) 是一个独立的工具，用来通过 [kustomization 文件](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization) 定制 Kubernetes 对象。

```
wget https://github.com/kubernetes-sigs/kustomize/releases/download/v3.2.0/kustomize_3.2.0_linux_amd64
tar -zxvf kustomize_3.2.0_linux_amd64
sudo mv kustomize /usr/local/bin/kustomize
kustomize help
```

### 1.4.CSI Plugin : Local Path Provisioner准备

- a.CSI 插件是 Kubernetes 中负责存储的模块。安装 CSI 插件 "本地路径供应器"（Local Path Provisioner），可在单节点集群中轻松使用，会根据classStorage和pvc自动创建pv.

  ```
  kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.20/deploy/local-path-storage.yaml
  
  kubectl get pod  -n local-path-storage
  NAME                                      READY   STATUS    RESTARTS   AGE
  local-path-provisioner-556d4466c8-45jkr   1/1     Running   0          13m
  
  kubectl get sc
  NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
  local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  15m
  ```

- b.设置默认sc

  ```
  kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
  
  kubectl get sc
  NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
  local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  15m
  ```

  

## 2.kubeflow组件安装

### 2.1 一键安装

- 所有组件资源全在kubeflow.yaml里面，出现错误不方便定位（不建议）

  ```
  while ! kustomize build example | kubectl apply -f -; do echo "Retrying to apply resources"; sleep 10; done
  ```

### 2.2组件单独安装

- 1.单独生成各个组件yaml文件（组件yaml文件：10.16.0.9:/root/componentFile）

  ```
  kustomize build common/cert-manager/cert-manager/base > compnentFile/001-cert-manager.yaml
  kustomize build common/cert-manager/kubeflow-issuer/base > compnentFile/002-kubeflow-issuer.yaml
  kustomize build common/istio-1-14/istio-crds/base > compnentFile/003-istio-crds.yaml
  kustomize build common/istio-1-14/istio-namespace/base > compnentFile/004-istio-namespace.yaml
  kustomize build common/istio-1-14/istio-install/base > componentFile/005-istio-install.yaml
  kustomize build common/dex/overlays/istio > componentFile/006-dex-overlays-istio.yaml
  kustomize build common/oidc-authservice/base > componentFile/007-oidc-authservice.yaml
  kustomize build common/kubeflow-namespace/base > componentFile/008-kubeflow-namespace.yaml
  kustomize build common/knative/knative-serving/overlays/gateways > componentFile/009-knative-serving.yaml
  kustomize build common/istio-1-14/cluster-local-gateway/base > 010-cluster-local-gateway.yaml
  kustomize build common/knative/knative-eventing/base > 011-knative-eventing.yaml
  kustomize build apps/centraldashboard/upstream/overlays/kserve > 012.centraldashboard.yaml
  kustomize build apps/admission-webhook/upstream/overlays/cert-manager > 013-admission-webhook.yaml
  kustomize build common/kubeflow-roles/base > componentFile/014-kubeflow-roles.yaml
  kustomize build common/istio-1-14/kubeflow-istio-resources/base > componentFile/015-kubeflow-istio-resources/yaml
  kustomize build apps/jupyter/notebook-controller/upstream/overlays/kubeflow > componentFile/016-notebook-controller
  kustomize build apps/jupyter/jupyter-web-app/upstream/overlays/istio > componentFile/017-jupyter-web-app.yaml
  kustomize build apps/profiles/upstream/overlays/kubeflow > componentFile/018-profiles.yaml
  kustomize build apps/volumes-web-app/upstream/overlays/istio > componentFile/019-volumes-web-app.yaml
  kustomize build apps/tensorboard/tensorboards-web-app/upstream/overlays/istio > componentFile/020-tensorboards-web-app.yaml
  kustomize build apps/tensorboard/tensorboard-controller/upstream/overlays/kubeflow > componentFile/021-tensorboard-controller.yaml
  kustomize build apps/training-operator/upstream/overlays/kubeflow > componentFile/022-training-operator.yaml
  kustomize build apps/katib/upstream/installs/katib-with-kubeflow > componentFile/023-katib-with-kubeflow.yaml
  kustomize build contrib/kserve/kserve > componentFile/024.kserve.yamlkustomize build contrib/kserve/models-web-app/overlays/kubeflow > componentFile/025-models-web-app.yaml
  kustomize build apps/pipeline/upstream/env/cert-manager/platform-agnostic-multi-user > componentFile/026-platform-agnostic-multi-user.yaml
  kustomize build common/user-namespace/base > 027-user-namespace.yaml
  ```

  <!--容器运行时不为docker时，026-platform-agnostic-multi-user.yaml需要替换-->

  ```
  kustomize build apps/pipeline/upstream/env/platform-agnostic-multi-user-pns > componentFile/026-platform-agnostic-multi-user-pns.yaml
  ```

- 2.修改镜像源， 由于很多组件依赖的镜像无法下载(主要为gcr.io镜像源)，需要替换为使用daocloud代理的镜像

  ```
  sed -i 's/gcr.io/gcr.m.daocloud.io/g' ./* 
  ```

- 3.提前下载镜像

  ```
  docker pull gcr.m.daocloud.io/arrikto/istio/pilot:1.14.1-1-g19df463bb
  docker pull gcr.m.daocloud.io/arrikto/kubeflow/oidc-authservice:28c59ef
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/eventing/cmd/controller@sha256:dc0ac2d8f235edb04ec1290721f389d2bc719ab8b6222ee86f17af8d7d2a160f                                                                                                                                          
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/eventing/cmd/mtping@sha256:632d9d710d070efed2563f6125a87993e825e8e36562ec3da0366e2a897406c0
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/eventing/cmd/webhook@sha256:b7faf7d253bd256dbe08f1cac084469128989cf39abbe256ecb4e1d4eb085a31
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/net-istio/cmd/controller@sha256:f253b82941c2220181cee80d7488fe1cefce9d49ab30bdb54bcb8c76515f7a26                                                                                                                             
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:a705c1ea8e9e556f860314fe055082fbe3cde6a924c29291955f98d979f8185e
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/activator@sha256:93ff6e69357785ff97806945b284cbd1d37e50402b876a320645be8877c0d7b7
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/autoscaler@sha256:007820fdb75b60e6fd5a25e65fd6ad9744082a6bf195d72795561c91b425d016                                                                                                                                          
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/controller@sha256:75cfdcfa050af9522e798e820ba5483b9093de1ce520207a3fedf112d73a4686                                                                                                                                           
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/domain-mapping@sha256:23baa19322320f25a462568eded1276601ef67194883db9211e1ea24f21a0beb                                                                                                                                       
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/domain-mapping-webhook@sha256:847bb97e38440c71cb4bcc3e430743e18b328ad1e168b6fca35b10353b9a2c22                                                                                                                               
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/queue@sha256:14415b204ea8d0567235143a6c3377f49cbd35f18dc84dfa4baa7695c2a9b53d
  docker pull gcr.m.daocloud.io/knative-releases/knative.dev/serving/cmd/webhook@sha256:9084ea8498eae3c6c4364a397d66516a25e48488f4a9871ef765fa554ba483f0
  docker pull gcr.m.daocloud.io/kubebuilder/kube-rbac-proxy:v0.8.0                                                                                        
  docker pull gcr.m.daocloud.io/ml-pipeline/api-server:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/cache-server:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/frontend:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/metadata-writer:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance
  docker pull gcr.m.daocloud.io/ml-pipeline/mysql:5.7-debian
  docker pull gcr.m.daocloud.io/ml-pipeline/persistenceagent:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/scheduledworkflow:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/viewer-crd-controller:2.0.0-alpha.3
  docker pull gcr.m.daocloud.io/ml-pipeline/visualization-server:2.0.0-alpha.3                                                                                         
  docker pull gcr.m.daocloud.io/ml-pipeline/workflow-controller:v3.3.8-license-compliance
  docker pull gcr.m.daocloud.io/tfx-oss-public/ml_metadata_store_server:1.5.0   
  ```

- 4.部署组件，建议单个组件部署，也可以编写脚本一键部署

  ```
  #!/bin/bash
  
  # 获取当前目录下的所有文件
  files=$(ls $pwd)
  for file in $files; do
    kubectl apply -f $file
  done
  ```

- 5.检查所有组件pod是否运行成功

  ```
  kubectl get pods -n cert-manager
  kubectl get pods -n istio-system
  kubectl get pods -n auth
  kubectl get pods -n knative-eventing
  kubectl get pods -n knative-serving
  kubectl get pods -n kubeflow
  kubectl get pods -n kubeflow-user-example-com
  ```

  ![image-20240319160631811](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240319160633.png)

- 6.通过暴露主机8080端口访问kubeflow 

  ```
  kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80 --address 0.0.0.0
  
  浏览器访问：http://[IP]:8080
  默认用户名：user@example.com
  默认密码：12341234
  ```
  ![image-20240319160938455](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240319160942.png)

## 3.安装部署遇到的问题

- 1.类似：no matches for kind "ClusterServingRuntime" in version "serving.kserve.io/v1alpha1"
  ensure CRDs are installed first

  解决： 有些CRD资源没有创建出来比较慢，等几秒之后，重新再执行一次kubectl apply命令。

  ![image-20240319161811004](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240319162126.png)

- 2.挂载pv的pod内部报错，文件: permission denied
  解决：dex、mysql、minio、katib-mysql四个pod需要挂载pv，需要修改pv挂载目录的权限 chmod 777 -R /data/k8s
  ![image-20240319162724822](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240319162726.png)

- 3.登陆kubeflow界面，不展示命令空间

  解决：容器云kube(Kubernetes v1.22.23-tenant)版本问题，具体原因不明，安装官方k8s1.22.7集群正常,暂时通过添加标签解决

  ```
  kubectl label ns kubeflow-user-saber environment=system
  ```
  
  ![image-20240319163321715](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240319163325.png)

## 4.待处理

- 1.在容器云kube(Kubernetes v1.22.23-tenant)底座上部署kubeflow，登陆kubeflow界面，不展示命令空间



# 二、kubeflow组件介绍

Kubeflow 项目致力于让机器学习（ML）工作流在 Kubernetes 上的部署变得简单、可移植和可扩展。我们的目标不是再造其他服务，而是提供一种直接的方式，将最优秀的开源 ML 系统部署到不同的基础设施上。在运行 Kubernetes 的任何地方，都可以运行 Kubeflow。下图显示了 Kubeflow 的主要组件，涵盖了 Kubernetes 上 ML 生命周期的每个步骤。

![image-20240326174324292](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240326174329.png)

## 1.Notebook

Jupyter Notebook是以网页的形式打开，可以在网页页面中**直接**编写代码和运行代码，代码的运行结果也会直接在代码块下显示。一个网页版的IDE开发环境，不同的镜像有不同的开发环境。

[Container Images | Kubeflow](https://www.kubeflow.org/docs/components/notebooks/container-images/)

## 1.1官方提供的kubeflow镜像信息

<img src="https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152153.png" alt="image-20240320104043168" style="zoom: 67%;" />

### 1.2Notebooks使用

- 1.创建Notebooks

  ![image-20240321160731154](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152212.png)

  ![image-20240321161504507](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152229.png)

  2.running状态之后，连接Notebook

  ![image-20240321161628451](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152248.png)

- 3.创建python环境或者终端

  ![image-20240321161848400](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152259.png)

- 4.克隆minist代码 https://github.com/michaelliao/mnist.git

  ![image-20240321162217143](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152321.png)

- 5.运行train.py示例

  ![image-20240321162354009](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152333.png)

## 2.tensorBoard

TensorBoard是一套用于检查和理解TensorFlow运行和图的网页应用程序。TensorBoard可以在训练过程中实时地可视化模型的性能和状态，观察到训练过程中的损失值、准确率、梯度、权重直方图等。

## 2.1tensorBoard使用示例

- 1.在tensorflow的Notebook环境中，运行demo并输出训练日志

  ```
  import tensorflow as tf 
   
  # Load and normalize MNIST data 
  mnist_data = tf.keras.datasets.mnist 
  (X_train, y_train), (X_test, y_test) = mnist_data.load_data() 
  X_train, X_test = X_train / 255.0, X_test / 255.0 
   
  # Define the model 
  model = tf.keras.models.Sequential([ 
  tf.keras.layers.Flatten(input_shape=(28, 28)), 
  tf.keras.layers.Dense(128, activation='relu'), 
  tf.keras.layers.Dropout(0.2), 
  tf.keras.layers.Dense(10, activation='softmax') 
  ]) 
   
  # Compile the model 
  model.compile(optimizer='adam', loss='sparse_categorical_crossentropy', metrics=['accuracy'])
  tf_callback = tf.keras.callbacks.TensorBoard(log_dir="./logs")
  model.fit(X_train, y_train, epochs=5, callbacks=[tf_callback])
  ```

![image-20240321163004341](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152349.png)

- 2.创建tensorboard

![image-20240321163713874](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152400.png)

- 3.连接tensorboard

![image-20240321163942434](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152411.png)

## 3.pipleline

pipleline是对 ML 工作流的描述，包括工作流中的所有组件以及它们如何以图形的形式组合在一起。(管道包括运行管道所需的输入（参数）定义以及每个组件的输入和输出。

[Kubeflow Pipelines | Kubeflow](https://www.kubeflow.org/docs/components/pipelines/)

### 3.1基本概念

- 实验（experiment）是一个工作空间，在其中可以针对流水线尝试不同的配置。
- 组件（component ）是一套独立的代码，用于执行 ML 工作流（管道）中的一个步骤，如数据预处理、数据转换、模型训练等。组件类似于函数，有名称、参数、返回值和主体。

- 运行（run）是流水线的一次执行，用户在执行的过程中可以看到每一步的输出文件，以及日志。

- 步（step）是组件的一次运行，步与组件的关系就像是运行与流水线的关系一样。

- 步输出工件（step output artifacts）是在组件的一次运行结束后输出的，能被系统的前端理解并渲染可视化的文件。

### 3.2如何创建一个pipleline

- 1.创建一个experiment

  ![image-20240322142456149](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152426.png)

- 2.编写pipeline的python代码

  以tensorflow的模型训练、模型推理为例，以下主要实现四个组件：

  1.创建pvc，作为模型训练和模型推理的数据存储；

  2.模型训练任务；

  3.模型推理任务；

  4.输出模型推理结果；

  其中第1步创建的pvc会被后面3步用到，第2步训练产生的模型文件和用于第3步测试的数据都会被保存在pvc中，第3步从pvc中读取数据并进行测试，测试结果写在txt中，依旧保存在pvc中，第四步从pvc中读取txt并进行输出。代码如下：

  [pipeline/train_and_predict.py · develop · bingomatrix / example / kubeflow-start · GitLab (bingosoft.net)](https://gitlab.bingosoft.net/-/ide/project/bingomatrix/example/kubeflow-start/edit/develop/-/pipeline/train_and_predict.py)

​	   在tensorflow环境中运行上述代码,生成**train_predict.yaml**文件保存到本机。

- 3.在UI界面上创建pipeline，生成pipeline的graph

![image-20240322145401020](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152517.png)

![image-20240322145509983](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152533.png)

- 4.创建run

![image-20240322154718374](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152555.png)

- 5.查看运行结果

![image-20240322154815565](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152612.png)

可以查看每一次组件的运行情况以及输入输出数据。

![image-20240322154927412](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240323152630.png)

## 4.Training Operator （分布式训练）

[Training Operators | Kubeflow](https://v1-6-branch.kubeflow.org/docs/components/training/)

- Kubeflow Training Operator 是一个 Kubernetes 原生项目，用于对使用 PyTorch、TensorFlow、XGBoost 等各种 ML 框架创建的机器学习（ML）模型进行微调和可扩展的分布式训练。

- 用户可以将 HuggingFace、DeepSpeed 或 Megatron 等其他 ML 库与 Training Operator 集成，以便在 Kubernetes 上协调其 ML 训练。

- 通过 Kubernetes Custom Resources API 或使用 Training Operator Python SDK，Training Operator 允许您使用 Kubernetes 工作负载有效地训练大型模型。

- Training Operator 实现了集中式 Kubernetes 控制器，以协调分布式训练作业。

- 用户可以使用 Training Operator 和 MPIJob 运行高性能计算（HPC）任务，因为它支持在 Kubernetes 上运行大量用于 HPC 的消息传递接口（MPI）。Training Operator 实现了 MPI Operator V1 API 版本。如需 MPI Operator V2 版本，请按照本指南安装 MPI Operator V2

### 1.kubeflow中用于 ML 框架的定制资源

| ML Framework | Custom Resource                                              |
| ------------ | ------------------------------------------------------------ |
| PyTorch      | [PyTorchJob](https://www.kubeflow.org/docs/components/training/pytorch/) |
| TensorFlow   | [TFJob](https://www.kubeflow.org/docs/components/training/tftraining/) |
| XGBoost      | [XGBoostJob](https://www.kubeflow.org/docs/components/training/xgboost/) |
| MPI          | [MPIJob](https://www.kubeflow.org/docs/components/training/mpi/) |
| PaddlePaddle | [PaddleJob](https://www.kubeflow.org/docs/components/training/paddlepaddle/) |

### 2.架构

![image-20240323170305325](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240325134613.png)

- Training Operator 负责调度适当的 Kubernetes 工作负载，为不同的 ML 框架实施各种分布式训练策略。以下示例主要展示 Training Operator 如何在 Kubernetes 上运行分布式 **PyTorch** 和 **TensorFlow**。

### 3.PyTorch Training (PyTorchJob)

该图显示了Training Operator job 如何为* [ring all-reduce algorithm](https://tech.preferred.jp/en/blog/technologies-behind-distributed-deep-learning-allreduce/)*创建 PyTorch Worker。

用户负责使用本地 PyTorch 分布式 API 编写培训代码，并使用 Training Operator Python SDK 创建一个 PyTorchJob，其中包含所需数量的 Worker 和 GPU。然后，Training Operator为 torchrun CLI 创建带有适当环境变量的 Kubernetes pod，以启动分布式 PyTorch 培训作业。

在环形全还原算法结束时，梯度会在每个 Worker（g1、g2、g3、g4）中同步，模型就完成了训练。

您可以在训练代码中定义 PyTorch 支持的各种分布式策略（例如 PyTorch FSDP），Training Operator会为 torchrun 设置相应的环境变量。

![image-20240323170727554](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240325134627.png)

#### 3.1pyTorch示例：

```
apiVersion: "kubeflow.org/v1"
kind: PyTorchJob
metadata:
  name: pytorch-simple
  namespace: kubeflow
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: docker.io/kubeflowkatib/pytorch-mnist:v1beta1-45c5727
              imagePullPolicy: Always
              command:
                - "python3"
                - "/opt/pytorch-mnist/mnist.py"
                - "--epochs=1"
    Worker:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: docker.io/kubeflowkatib/pytorch-mnist:v1beta1-45c5727
              imagePullPolicy: Always
              command:
                - "python3"
                - "/opt/pytorch-mnist/mnist.py"
                - "--epochs=1"
```

### 4.TensorFlow Training (TFJob)

该图显示了Training Operator job 如何创建 TensorFlow 参数服务器（PS）和用于 PS 分布式训练的工作者。

用户负责使用本地 TensorFlow 分布式 API 编写训练代码，并使用 Training Operator Python SDK 创建一个包含所需数量 PS、Worker 和 GPU 的 TFJob。然后，Training Operator 为 TF_CONFIG 创建带有适当环境变量的 Kubernetes pod，以启动分布式 TensorFlow 训练任务。

参数服务器为每个工作者分割训练数据，并根据每个工作者产生的梯度平均模型权重。

您可以在训练代码中定义 TensorFlow 支持的各种分布式策略，**Training Operator 会为 TF_CONFIG 设置相应的环境变量**。

![image-20240323171046601](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240325134649.png)

TFJob 与内置控制器的不同之处在于，TFJob 规范旨在管理分布式 TensorFlow 训练作业。分布式 TensorFlow 作业通常包含以下 0 个或多个进程：

- **Chief** ：负责协调训练和执行检查点模型等任务。
- **Ps** 是参数服务器；这些服务器为模型参数提供分布式数据存储。
- **Worker** ：负责训练模型的实际工作。在某些情况下，Worker 0 还可能充当Chief 。
- **Evaluator** ：在模型训练过程中，评价器可用于计算评价指标。

#### 4.1.tfjob YAML示例

```
apiVersion: kubeflow.org/v1
kind: TFJob
metadata:
  generateName: tfjob
  namespace: your-user-namespace
spec:
  tfReplicaSpecs:
    PS:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: tensorflow
              image: gcr.io/your-project/your-image
              command:
                - python
                - -m
                - trainer.task
                - --batch_size=32
                - --training_steps=1000
    Worker:
      replicas: 3
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: tensorflow
              image: gcr.io/your-project/your-image
              command:
                - python
                - -m
                - trainer.task
                - --batch_size=32
                - --training_steps=1000
              resources:
                limits:
                  nvidia.com/gpu: 1
```

## 5.Job调度

本节介绍如何在 Kubeflow 中使用 **Volcano** 调度器支持帮派调度（gang-scheduling），以允许作业同时运行多个 pod。

[[Job Scheduling | Kubeflow](https://v1-6-branch.kubeflow.org/docs/components/training/job-scheduling/)](https://www.kubeflow.org/docs/components/training/job-scheduling/)

Kubeflow 中的**Volcano**调度器和operator 通过使用 PodGroup 实现gang-scheduling，operator 会自动创建作业的 PodGroup。

### 5.1gang-scheduling 

Training Operator支持使用 **Volcano 调度器**以**gang-scheduling**调度方式运行作业。

使用 Volcano 调度器来应用群组调度时，只有当worker的所有 pod 都有足够的资源时，作业才能运行。否则，所有 pod 都将处于待处理状态，等待足够的资源。例如，如果创建了一个需要 N 个 pod 的作业，但只有足够的资源调度 N-2 个 pod，那么该作业的 N 个 pod 将处于待处理状态。

注意：当工作负载较高时，如果作业的某个 pod 在作业仍在运行时死亡，可能会给其他 pod 占用资源的机会，从而导致死锁。

- 在安装好Volcano 的kubeflow环境中，在training-operator新增--enable-gang-scheduling=true如下：

  ```
  ...
      spec:
        containers:
          - command:
              - /manager
  +           - --enable-gang-scheduling=true
            image: kubeflow/training-operator
            name: training-operator
  ...
  ```


使用Volcano 调度程序将作业调度为 gang 的 yaml 与non-gang-scheduler程序相同，tfjob的yaml文件不需要进行修改。

### 5.2使用Volcano 调度器和不使用Volcano 对比

环境中有一个GPU，以创建tfjob为例，创建2个ps和带gpu的4个worker,文件tf_job_mnist.yaml如下：

```
apiVersion: "kubeflow.org/v1"
kind: "TFJob"
metadata:
  name: "dist-mnist-for-e2e-test"
spec:
  tfReplicaSpecs:
    PS:
      replicas: 2
      restartPolicy: Never
      template:
        spec:
          containers:
            - name: tensorflow
              image: kubeflow/tf-dist-mnist-test:latest
    Worker:
      replicas: 4
      restartPolicy: Never
      template:
        spec:
          containers:
            - name: tensorflow
              image: kubeflow/tf-dist-mnist-test:latest
              resources:
                limits:
                  nvidia.com/gpu: 1
```

- 不使用**Volcano** 时

  其中只有一个worker运行，3个worker处于pending状态

  ```
  [root@master zhangqi]# kubectl get pod -A|grep mnis
  default                     dist-mnist-for-e2e-test-ps-0                                  1/1     Running     0                10s
  default                     dist-mnist-for-e2e-test-ps-1                                  1/1     Running     0                10s
  default                     dist-mnist-for-e2e-test-worker-0                              1/1     Running     0                11s
  default                     dist-mnist-for-e2e-test-worker-1                              0/1     Pending     0                11s
  default                     dist-mnist-for-e2e-test-worker-2                              0/1     Pending     0                11s
  default                     dist-mnist-for-e2e-test-worker-3                              0/1     Pending     0                11s
  
  其他worker
  Events:
    Type     Reason            Age   From               Message
    ----     ------            ----  ----               -------
    Warning  FailedScheduling  96s   default-scheduler  0/1 nodes are available: 1 Insufficient nvidia.com/gpu.
  ```

- 使用**Volcano** 时:

  podGroup处于pending状态，没有ps和worker的pod被创建出来

  ```
  [root@master zhangqi]# kubectl get podGroup
  NAME                      STATUS    MINMEMBER   RUNNINGS   AGE
  dist-mnist-for-e2e-test   Pending   6                      25m
  
  [root@master zhangqi]# kubectl get pod -A|grep mnist
  [root@master zhangqi]#
  ```

  


