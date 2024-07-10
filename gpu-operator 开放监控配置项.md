# gpu-operator 开放监控配置项

- 配置一个configmap
- 对gpu-operator进行role bining进行配置，使得pod可访问configmap
- 下载gpu-operator的配置文件，修改values， 让dcgmExporter 可访问对应配置项
- 重启gpu-operator



## 1、configmaps配置

- 这里忽略部分配置

```yaml
apiVersion: v1
data:
  metrics: |-
    DCGM_FI_DEV_SM_CLOCK,  gauge, SM clock frequency (in MHz).
    DCGM_FI_DEV_MEM_CLOCK, gauge, Memory clock frequency (in MHz).
    DCGM_FI_DEV_MEMORY_TEMP, gauge, Memory temperature (in C).
    DCGM_FI_DEV_GPU_TEMP,    gauge, GPU temperature (in C).
    DCGM_FI_DEV_POWER_USAGE,              gauge, Powgpu-operatorer draw (in W).
    ... 
kind: ConfigMap
metadata:
  name: dcgm-metrics-config
  namespace: kube-system
```

- ```bash
  kubectl apply -f config.yaml
  ```



## 2、role配置

- 添加role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nvidia-dcgm-exporter-configmaps
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch"]
```

- 添加rolebinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: nvidia-dcgm-exporter-configmaps-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nvidia-dcgm-exporter-configmaps
subjects:
- kind: ServiceAccount
  name: nvidia-dcgm-exporter
  namespace: gpu-operator
```



## 3、修改启动的values

```yaml
dcgmExporter:
  enabled: true
  repository: nvcr.io/nvidia/k8s
  image: dcgm-exporter
  version: 3.3.0-3.2.0-ubuntu22.04
  imagePullPolicy: IfNotPresent
  env:
    - name: DCGM_EXPORTER_LISTEN
      value: ":9400"
    - name: DCGM_EXPORTER_KUBERNETES
      value: "true"
    - name: DCGM_EXPORTER_CONFIGMAP_DATA
      value: "kube-system:dcgm-metrics-config"
  arguments: ["-m", "kube-system:dcgm-metrics-config"]
  resources: {}
  serviceMonitor:
    enabled: false
    interval: 15s
    honorLabels: false
    additionalLabels: {}
    relabelings: []
```

关键修改点：

```yaml
- name: DCGM_EXPORTER_CONFIGMAP_DATA
  value: "kube-system:dcgm-metrics-config"
arguments: ["-m", "kube-system:dcgm-metrics-config"]
```



## 4、启动

```bash
helm upgrade --install gpu-operator -n gpu-operator bingomatrix/gpu-operator --create-namespace -f opt-values.yaml
```

```
helm upgrade --install gpu-operator -n gpu-operator bingomatrix/gpu-operator --create-namespace --set driver.rdma.enabled=true --set driver.rdma.useHostMofed=true --set global.repository=registry.bingosoft.net
```



----

#### 全部所需配置项的configmaps

```yaml
apiVersion: v1
data:
  metrics: |-
    DCGM_FI_DEV_SM_CLOCK,  gauge, SM clock frequency (in MHz).
    DCGM_FI_DEV_MEM_CLOCK, gauge, Memory clock frequency (in MHz).
    DCGM_FI_DEV_MEMORY_TEMP, gauge, Memory temperature (in C).
    DCGM_FI_DEV_GPU_TEMP,    gauge, GPU temperature (in C).
    DCGM_FI_DEV_POWER_USAGE,              gauge, Power draw (in W).
    DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION, counter, Total energy consumption since boot (in mJ).
    DCGM_FI_DEV_PCIE_TX_THROUGHPUT,  counter, Total number of bytes transmitted through PCIe TX (in KB) via NVML.
    DCGM_FI_DEV_PCIE_RX_THROUGHPUT,  counter, Total number of bytes received through PCIe RX (in KB) via NVML.
    DCGM_FI_DEV_PCIE_REPLAY_COUNTER, counter, Total number of PCIe retries.
    DCGM_FI_DEV_GPU_UTIL,      gauge, GPU utilization (in %).
    DCGM_FI_DEV_MEM_COPY_UTIL, gauge, Memory utilization (in %).
    DCGM_FI_DEV_ENC_UTIL,      gauge, Encoder utilization (in %).
    DCGM_FI_DEV_DEC_UTIL ,     gauge, Decoder utilization (in %).
    DCGM_FI_DEV_XID_ERRORS,            gauge,   Value of the last XID error encountered.
    DCGM_FI_DEV_FB_FREE, gauge, Framebuffer memory free (in MiB).
    DCGM_FI_DEV_FB_USED, gauge, Framebuffer memory used (in MiB).
    DCGM_FI_DEV_VGPU_LICENSE_STATUS, gauge, vGPU License status
    DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS, counter, Number of remapped rows for uncorrectable errors
    DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS,   counter, Number of remapped rows for correctable errors
    DCGM_FI_DEV_ROW_REMAP_FAILURE,           gauge,   Whether remapping of rows has failed
    DCGM_FI_PROF_GR_ENGINE_ACTIVE,   gauge, Ratio of time the graphics engine is active (in %).
    DCGM_FI_PROF_SM_ACTIVE,          gauge, The ratio of cycles an SM has at least 1 warp assigned (in %).
    DCGM_FI_PROF_SM_OCCUPANCY,       gauge, The ratio of number of warps resident on an SM (in %).
    DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Ratio of cycles the tensor (HMMA) pipe is active (in %).
    DCGM_FI_PROF_DRAM_ACTIVE,        gauge, Ratio of cycles the device memory interface is active sending or receiving data (in %).
    DCGM_FI_PROF_PCIE_TX_BYTES,      counter, The number of bytes of active pcie tx data including both header and payload.
    DCGM_FI_PROF_PCIE_RX_BYTES,      counter, The number of bytes of active pcie rx data including both header and payload.
    DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL,counter, Total number of NVLink bandwidth counters for all lanes
    DCGM_FI_DEV_NVLINK_BANDWIDTH_L0,counter, The number of bytes of active NVLink rx or tx data including both header and payload.
    DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL, counter, Total number of NVLink flow-control CRC errors.
    DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL, counter, Total number of NVLink data CRC errors.
    DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL,   counter, Total number of NVLink retries.
    DCGM_FI_DEV_NVLINK_RECOVERY_ERROR_COUNT_TOTAL, counter, Total number of NVLink recovery errors.
    DCGM_FI_PROF_PIPE_FP64_ACTIVE,   gauge, Ratio of cycles the fp64 pipes are active (in %).
    DCGM_FI_PROF_PIPE_FP32_ACTIVE,   gauge, Ratio of cycles the fp32 pipes are active (in %).
    DCGM_FI_PROF_PIPE_FP16_ACTIVE,   gauge, Ratio of cycles the fp16 pipes are active (in %).
kind: ConfigMap
metadata:
  name: dcgm-metrics-config
  namespace: kube-system

```




