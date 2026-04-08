# GPU 虚拟化

## 背景

随着 AI 应用的日益普及, 对 GPU 的需求也随之激增. GPU 作为模型训练与推理任务的核心组件, 其重要性不言而喻. 然而, GPU 成本高昂, 如何在云原生环境下最大化其利用率, 已成为业界关注的焦点. 在实际应用中, 常出现以下情况: 对于小型工作负载, 单个 GPU 可能造成资源浪费; 而对于大型工作负载, 单个 GPU 的算力又可能未被充分挖掘.

为应对这一挑战, Volcano 通过提供强大的虚拟 GPU (vGPU) 调度能力, 实现了物理 GPU 在多个容器和作业间的有效共享. 这不仅能显著提升 GPU 利用率、降低运营成本, 也为各类 AI/ML 工作负载带来了更灵活的资源调度方案.

Volcano 致力于简化 GPU 虚拟化的复杂度, 使用户能便捷地运用这些高级共享机制. 用户只需在 Pod 或作业的配置中声明所需的 GPU 资源及期望的切分方式, Volcano 便能自动完成底层的资源编排工作.

Volcano 主要支持以下两种 GPU 共享模式, 用以实现 vGPU 调度并满足不同的硬件能力与性能需求:

### 1. HAMI-core(基于软件的 vGPU)

描述: 通过 VCUDA (一种 CUDA API 劫持技术) 对 GPU 核心与显存的使用进行限制, 从而实现软件层面的虚拟 GPU 切片.

使用场景: 适用于需要细粒度 GPU 共享的场景, 兼容所有类型的 GPU.

### 2. Dynamic MIG(硬件级 GPU 切片)

描述: 采用 NVIDIA 的 MIG (Multi-Instance GPU)技术, 可将单个物理 GPU 分割为多个具备硬件级性能保障的隔离实例.

使用场景: 尤其适用于对性能敏感的工作负载, 要求 GPU 支持 MIG 特性(如 A100、H100 系列).

> Dynamic MIG 的设计文档请参考: [dynamic-mig](https://github.com/volcano-sh/volcano/blob/master/docs/design/dynamic-mig.md)
>
> GPU 共享模式属于节点级别的配置. Volcano 支持异构集群, 即集群中可同时包含采用 HAMI-core 模式的节点和采用 Dynamic MIG 模式的节点. 更多配置及详情, 请参阅[volcano-vgpu-device-plugin](https://github.com/Project-HAMi/volcano-vgpu-device-plugin).
>

## 安装

若需启用 vGPU 调度功能, 请根据所选模式配置以下组件:

### 通用要求

环境依赖:

- NVIDIA 驱动 > 440

- nvidia-docker > 2.0

- Docker 已配置 `nvidia` 为默认运行时

- Kubernetes >= 1.16

- Volcano >= 1.9

- 安装 Volcano:

  - 具体步骤请参照 Volcano 安装指南.

- 安装设备插件**:

  - 部署 [volcano-vgpu-device-plugin](https://github.com/Project-HAMi/volcano-vgpu-device-plugin).

    > 说明: [vgpu 设备插件的 YAML 文件](https://github.com/Project-HAMi/volcano-vgpu-device-plugin/blob/main/volcano-vgpu-device-plugin.yml)中亦包含节点 GPU 模式及 MIG 实例规格等相关配置. 详情请参阅[vgpu 设备插件配置文档](https://github.com/Project-HAMi/volcano-vgpu-device-plugin/blob/main/doc/config.md).

- 验证部署: 请确保节点的可分配资源(Allocatable Resources)中包含以下信息:

```yaml
volcano.sh/vgpu-memory: "89424"
volcano.sh/vgpu-number: "8"
```

- 更新调度器配置:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: volcano-scheduler-configmap
  namespace: volcano-system
data:
  volcano-scheduler.conf: |
    actions: "enqueue, allocate, backfill"
    tiers:
    - plugins:
      - name: predicates
      - name: deviceshare
        arguments:
          deviceshare.VGPUEnable: true # 启用 vgpu 插件
          deviceshare.SchedulePolicy: binpack  # 调度策略: binpack / spread
```

可通过以下命令检查:

```bash
kubectl get node {node-name} -o yaml
```

### HAMI-core 使用方法

Pod配置示例:

```yaml
metadata:
  name: hami-pod
  annotations:
    volcano.sh/vgpu-mode: "hami-core"
spec:
  schedulerName: volcano
  containers:
  - name: cuda-container
    image: nvidia/cuda:9.0-devel
    resources:
      limits:
        volcano.sh/vgpu-number: 1 # 请求 1 张 GPU 卡
        volcano.sh/vgpu-cores: 50 # (可选)每个 vGPU 使用 50% 核心
        volcano.sh/vgpu-memory: 3000 # (可选)每个 vGPU 使用 3G 显存
```

### Dynamic MIG 使用方法

- 启用 MIG 模式:

若需启用 MIG (Multi-Instance GPU)模式, 请在目标 GPU 节点上执行以下命令:

```bash
sudo nvidia-smi -mig 1
```

- MIG 实例规格配置(可选):

`volcano-vgpu-device-plugin` 会自动生成一套初始 MIG 配置, 并存储于 `kube-system` 命名空间下的 `volcano-vgpu-device-config` ConfigMap 中. 用户可按需自定义此配置. 更多详情请参阅 [vgpu 设备插件 YAML 文件](https://github.com/Project-HAMi/volcano-vgpu-device-plugin/blob/main/volcano-vgpu-device-plugin.yml).

- 带 MIG 注解的 Pod 配置示例:

```yaml
metadata:
  name: mig-pod
  annotations:
    volcano.sh/vgpu-mode: "mig"
spec:
  schedulerName: volcano
  containers:
  - name: cuda-container
    image: nvidia/cuda:9.0-devel
    resources:
      limits:
        volcano.sh/vgpu-number: 1
        volcano.sh/vgpu-memory: 3000
```

注意: 实际分配的显存大小取决于最匹配的 MIG 实例规格(例如: 请求 3GB 显存, 可能会分配到规格为 5GB 的 MIG 实例).

## 调度器模式选择

- 显式指定模式:

  - 通过 Pod 注解 `volcano.sh/vgpu-mode` 来强制指定 HAMI-core 或 MIG 模式.
  - 若未指定该注解, 调度器将依据资源匹配度及预设策略自动选择合适的模式.

- 调度策略影响:

  - 调度策略(如 `binpack` 或 `spread`)会影响 vGPU Pod 在节点间的分布.

## 总结表

| 模式 | 隔离级别 | 是否依赖 MIG GPU | 需注解指定模式 | 核心/显存控制方式 | 推荐应用场景 |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| HAMI-core | 软件 (VCUDA) | 否 | 否 | 用户自定义 (核心/显存) | 通用型工作负载 |
| Dynamic MIG | 硬件 (MIG) | 是 | 是 | MIG 实例规格决定 | 对性能敏感的工作负载 |

## 监控

- 调度器监控指标:

```bash
curl http://<volcano-scheduler-ip>:8080/metrics
```

- 设备插件监控指标:

```bash
curl http://<plugin-pod-ip>:9394/metrics
```

监控指标包括 GPU 利用率、各 pod 的显存使用量及限制等.

## 问题和贡献

- 提交 issue: [volcano issues](https://github.com/volcano-sh/volcano/issues)

- 贡献代码: [Pull Request指南](https://help.github.com/articles/using-pull-requests/)
