## NVIDIA GPU调度和共享

### 1.NVIDIA

### 1.1GPU-operator

Kubernetes 可通过设备插件框架访问英伟达™（NVIDIA®）图形处理器、网卡、Infiniband 适配器和其他设备等特殊硬件资源。但是，配置和**管理带有这些硬件资源的节点需要配置多个软件组件**，如驱动程序、容器运行时或其他库，这既困难又容易出错。

英伟达™（NVIDIA®）**GPU operator**使用 Kubernetes 中的操作员框架来自动**管理配置 GPU 所需的所有英伟达™（NVIDIA®）软件组件**。这些组件包括英伟达驱动程序（启用 CUDA）、用于 GPU 的 Kubernetes 设备插件、英伟达容器工具包、使用 GFD 的自动节点标签、基于 DCGM 的监控等。它包含以下几个主要组件：

- **Node Feature Discovery (NFD)**：用于探测节点上的硬件特性并用标签的方式标出来，这样在部署应用时可以使其优先在拥有特定硬件特性的节点上运行，从而提高应用运行效率。

- **GPU Feature Discovery (GFD)**：如前面所述，这个工具与 NFD 相似，但更专注于 NVIDIA GPU 的特性发现。通过 GFD，可以将 GPU 的特性作为标签添加到节点上，从而实现精准调度。它通过 NVIDIA System Management Interface（nvidia-smi）工具来发现 NVIDIA GPU 的硬件特性。

- **Device Plugin**：Kubernetes 设备插件用于提供 GPU 资源的发现，管理和调度。它为 Kubernetes 提供了一种机制，使 Kubernetes 能够发现、管理、调度硬件加速器，如 GPU。

  ```
  以下是 NVIDIA Kubernetes Device Plugin 的主要工作流程：
  1.发现 GPU：插件首先会在启动时发现主机上所有可用的 NVIDIA GPU，并收集这些 GPU 的相关信息。
  2.注册 GPU：之后，插件会将这些 GPU 作为节点级别的可调度资源注册到 Kubernetes 中。这种注册过程是通过 Kubernetes 的 Device Plugin Mechanism 实现的。
  3.调度和分配：现在，当我们创建一个需要 GPU 的 pod 时，只需在 pod 的 spec 中声明对 "nvidia.com/gpu" 资源的需求，Kubernetes 便可以根据情况把 pod 调度到一个有足够 GPU 资源的节点上，并在 pod 启动后，通过 Device Plugin Mechanism 将这些 GPU 分配给 pod。
  4.容器运行时的设置：最后，会安排适当的 GPU 驱动（比如 libcuda.so 和相应的设备文件，如 /dev/nvidia0，/dev/nvidiactl）被映射到 Pod 中。
  ```

- **Driver Container**：这个组件负责在每个节点上安装和配置适合的 GPU 驱动。

  ```
  Driver Container 将 GPU 驱动的共享库添加到系统的共享库路径中。这样，任何需要使用 GPU 功能的进程都可以找到并链接这些库。
  ```

- **Toolkit Container**：这个组件包含一些支持库和工具，比如 CUDA 工具套件和 cuDNN 库，它们为 GPU 加速的应用程序提供运行环境。

  ```
  nvidia-container-toolkit 是 NVIDIA 开发的一个工具，其主要作用是允许 Docker 容器使用主机上安装的 NVIDIA GPU。
  以下是 nvidia-container-toolkit 的主要功能：
  -机器级别的资源隔离：nvidia-container-toolkit 具有机器级别的资源隔离功能，这样就可以防止不同的容器之间共享 GPU，并且可以将某个特定的 GPU 分配给特定的容器。
  -注入库和驱动程序：NVIDIA GPU 的运行需要特定的库和驱动，但在标准的容器环境中，这些元素是不存在的。nvidia-container-toolkit 会自动将需要的库和驱动注入到 GPU-enabled 的容器中，使其能够正常工作。
  -兼容性：nvidia-container-toolkit 兼容各种主流的容器运行环境，包括 Docker、Podman 等。
  -简化 GPU 资源管理：nvidia-container-toolkit 可以处理一些复杂的 GPU 资源管理问题，例如版本匹配、容器间的 GPU 资源隔离等。
  总的来说，nvidia-container-toolkit 的作用就是把主机上的 NVIDIA GPU 资源暴露给容器，并对这些资源进行一些基本的管理，使得容器能够像主机一样使用这些 GPU 资源。
  ```

- **DCGM (Data Center GPU Manager)**：这是一个用于 GPU 的监控和管理工具，可以提供各种 GPU 的运行状态信息，如温度、功耗、利用率等，帮助管理员更好地了解和管理 GPU 资源。


### 1.2 NVIDIA GPU共享类型

[NVIDIA/k8s-device-plugin: NVIDIA device plugin for Kubernetes (github.com)](https://github.com/NVIDIA/k8s-device-plugin?tab=readme-ov-file#configuring-the-nvidia-device-plugin-binary)

可以通过配置NVIDIA k8s-device-plugin设置GPU共享类型

- time-slicing（默认共享模式）

  ```
  优点：
  1.时间片方式实现公平的资源分配
  2.提高 GPU 利用率
  
  缺点：
  1.不同任务之间存在上下文切换开销
  2.无法实现资源的隔离。
  3.无法对显存、算力进行限制
  
  ```

- MIG

  https://docs.google.com/document/d/1mdgMQ8g7WmaI_XVVRrCvHPFPOMCm5LQD5JefgAh6N8g/edit#heading=h.2w5idh51dpam

  ```
  优点：
  
  MIG可以将一块物理 GPU 分割成多个小的 GPU Instance，每个实例都有自己独立的 GPU、内存和高速缓存资源，这样可以提供更好的隔离性和效率。MIG 适用于多任务并行执行、资源分布不均导致 GPU 未能充分利用的场景。
  
  缺点：
  1.只有A30、A100、H100设备支持
  2.一张显卡最后只能切割为7个实例
  ```

- MPS

  [Multi-Process Service :: GPU Deployment and Management Documentation (nvidia.com)](https://docs.nvidia.com/deploy/mps/index.html)

  - 无MPS时，进程在整个 GPU 上分配一个串行时间片调度

  ![image-20240429163112090](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240429181741.png)

  - 使用mps时，创建一个mps都服务端来对cuda客户端的上下文请求进行管理

  ![image-20240429163224359](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240429181753.png)

  ```
  mps优势：
  1.可以使多个 CUDA 进程共享一个 GPU 上下文，从而有效提高了 GPU 的利用率，并且减少了进程切换带来的开销
  2.能够对cuda客户端进行内存限制，
  
  
  使用限制：
  1.无资源硬隔离，一个cuda客户端错误，可能会影响mps服务端的运行，从而影响其他cuda客户端进程。
  2.cuda客户端数量有限制
  3.同一时刻，只能有一个用户运行mps-server
  ```

   

## 2.HAMi--异构算力虚拟化中间件

[Project-HAMi/HAMi: Heterogeneous AI Computing Virtualization Middleware (github.com)](https://github.com/Project-HAMi/HAMi?tab=readme-ov-file)

HAMi实现了一个新的GPU调度的k8s-device-plugin，在兼容nvidia原生的device-plugin功能基础上，增加了一些新的能力：

- 显存资源和算力资源的硬隔离
- 允许通过指定显存和算力使用来申请设备资源
- 新增对国产GPU卡的支持

### 2.1安装

- 1.安装NVIDIA的gpu-operaor,并关闭device-plugin插件

  ```
  
  helm upgrade --install gpu-operator -n gpu-operator --create-namespace \
  bingomatrix/gpu-operator \
  --set driver.rdma.enabled=true --set driver.rdma.useHostMofed=true \
  --set driver.enabled=true \
  --set image.repository=registry.cn-hangzhou.aliyuncs.com/k8s-gcr-proxy/node-feature-discovery
  --set devicePlugin.enabled=false
  
  ```

- 2.安装HAMi[HAMi/docs/config_cn.md at master · Project-HAMi/HAMi (github.com)](https://github.com/Project-HAMi/HAMi/blob/master/docs/config_cn.md)

  ```
  
  helm repo add hami-charts https://project-hami.github.io/HAMi/
  
  helm install hami hami-charts/hami --set scheduler.kubeScheduler.imageTag=v1.16.8 -n kube-system
  
  --set devicePlugin.deviceSplitCount: GPU的分割数
  ```

### 2.2使用示例

```
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
    - name: ubuntu-container
      image: ubuntu:18.04
      command: ["bash", "-c", "sleep 86400"]
      resources:
        limits:
          nvidia.com/gpu: 2 # requesting 2 vGPUs
          nvidia.com/gpumem: 3000 # Each vGPU contains 3000m device memory （Optional,Integer）
          nvidia.com/gpucores: 30 # Each vGPU uses 30% of the entire GPU （Optional,Integer)
```

- 容器内部查看

  ![image-20240412163952384](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240429182431.png)

- 当容器使用内存大小超过限制时

  ![image-20240412190434928](https://gitlab.bingosoft.net/ccpc/docs/images/raw/develop/imgs/20240429182445.png)

