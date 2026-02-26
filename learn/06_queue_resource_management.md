# 队列资源管理

## 功能概述

[队列](https://volcano.sh/zh/docs/queue)是 Volcano 的核心概念之一, 用于支持多租户场景下的资源分配与任务调度. 通过队列, 用户可以实现多租资源分配、任务优先级控制、资源抢占与回收等功能, 显著提升集群的资源利用率和任务调度效率.

## 核心特性

### 1. 灵活的资源配置

- 支持多维度资源配额控制(CPU、内存、GPU、NPU 等)

- 提供三级资源配置机制:

  - capability: 队列资源使用上限

  - deserverd: 资源应得量(在无其他队列提交作业时, 该队列内作业所占资源量可超过 deserverd 值, 当有多个队列提交作业且集群资源不够用时, 超过 deserverd 值的资源量可以被其他队列回收)

  - guarantee: 资源预留量(预留资源只可被该队列所使用, 其他队列无法使用)

  > 建议及注意事项:
  >
  > 1. 进行三级资源配置时, 需遵循: guarantee <= deserverd <= capability;
  > 2. guarantee/capability 可按需配置, 在开启 capacity 插件时需要配置 deserverd 值;
  > 3. deserverd 配置建议: 在平级队列场景, 所有队列的 deserverd 值总和等于集群资源总量; 在层级队列场景, 子队列的 deserverd 值总和等于父队列的 deserverd 值, 但不能超过父队列的 deserverd 值.
  > 4. capability 配置注意事项: 在层级队列场景, 子队列的 capability 值不能超过父队列的 capability 值, 若子队列的 capability 未设置, 则会继承父队列的 capability 值.

- 支持动态资源配额调整

### 2.层级队列管理

- 支持多[层级队列](https://volcano.sh/zh/docs/hierarchical_queue)结构

- 提供父子队列间的资源继承与隔离

- 兼容 Yarn 式的资源管理模式, 便于大数据工作负载迁移

- 支持跨层级队列的资源共享与回收

### 3.智能资源调度

- 资源借用: 允许队列使用其他队列的空闲资源
- 资源回收: 当资源紧张时, 优先回收超额使用的资源
- 资源抢占: 确保高优先级任务的资源需求

### 4.多租户隔离

- 严格的资源配额控制
- 基于优先级的资源分配
- 防止单个租户过度占用资源

## 队列调度实现机制

### 队列相关 Actions

Volcano 中的队列调度涉及以下核心 action:

1. `enqueue`: 控制作业进入队列的准入机制, 根据队列的资源配额和当前使用情况决定是否允许新作业进入队列.

2. `allocate`: 负责资源分配过程, 确保分配符合队列配额限制, 同时支持队列间的资源借用机制, 提高资源利用率.

3. `preempt`: 支持**队列内**资源抢占. 高优先级作业可以抢占同队列内低优先级作业的资源, 确保关键任务的及时执行.

4. `reclaim`: 支持**队列间**的资源回收. 当队列资源紧张时, 触发资源回收机制. 优先回收超出队列 deserved 值的资源, 并结合队列/作业优先级选择合适的牺牲者.

> 注意: enqueue action 和 reclaim/preempt action 是互相冲突的, 如果 enqueue action 判断 podgroup 不允许入队, 则 vc-controller 不会创建 pending 状态的 pod, reclaim/preempt action 也不会执行.

### 队列调度插件

Volcano 提供了两个核心的队列调度插件:

#### capacity 插件

[capacity 插件](https://github.com/volcano-sh/volcano/blob/master/docs/user-guide/how_to_use_capacity_plugin.md)支持通过显式配置 deserverd 值来设置队列资源应得量, 如以下队列配置示例:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: capacity-queue
spec:
  deserved:
    cpu: "10"
    memory: "20Gi"
  capability:
    cpu: "20"
    memory: "40Gi"
```

capacity 插件通过精确的资源配置来进行配额控制, 结合[层级队列](https://volcano.sh/zh/docs/hierarchical_queue)能实现更加精细的多租资源分配, 也便于大数据工作负载迁移到 Kubernetes 集群上

> 注意: 当使用 Cluster Autoscaler 或 Karpenter 等集群弹性伸缩组件时, 集群资源总量会动态变化. 此时使用 capacity 插件需要手动调整队列的 deserverd 值以适应资源变化.

#### proportion 插件

与 capacity 插件不同的是, proportion 插件通过配置队列的 Weight 值来自动计算队列资源应得量, 无需显式配置 deserverd 值, 如以下队列配置示例:

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: proportion-queue
spec:
  weight: 1
  capability:
    cpu: "20"
    memory: "40Gi"
```

当集群总资源为 `total_resource` 时, 每个队列的 deserverd 值计算公式为:

```
queue_deserved = (queue_weight / total_weight) * total_resource
```

其中, `queue_weight` 表示当前队列的权重, `total_weight` 表示所有队列权重之和, `total_resource` 表示集群总资源量.

与 capacity 插件相比, capacity 插件可直接配置队列的 deserverd 值, 而 proportion 插件通过权重比例自动计算队列的 deserverd 值, 当集群资源发生变化时(如通过 Cluster Autoscaler 或 Karpenter 扩缩容), proportion 插件会自动根据权重比例重新计算各队列的 deserverd 值, 无需人工干预.

> 重要说明: 实际的 deserverd 值会进行动态调整, 如果计算得到的 `queue_deserved` 大于队列中待调度 PodGroup 的总资源请求量(Request), 则最终的 deserverd 值会被设置为总请求量(Request), 这样可以避免资源的过度预留, 提高整体利用率

#### 使用样例

以下示例展示了一个典型的队列资源管理场景, 通过 4 个步骤说明资源回收机制:

步骤1: 初始状态

集群初始状态下, default 队列可使用全部资源(4C).

步骤2: 创建初始作业

在 default 队列中创建两个作业, 分别申请 1C 和 3C 资源:

```yaml
# job1.yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: job1
spec:
  queue: default
  tasks:
    - replicas: 1
      template:
        spec:
          containers:
            - name: nginx
              image: nginx
              resources:
                requests:
                  cpu: "1"
---
# job2.yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: job2
spec:
  queue: default
  tasks:
    - replicas: 1
      template:
        spec:
          containers:
            - name: nginx
              image: nginx
              resources:
                requests:
                  cpu: "3"
```

此时两个 job 都能正常运行, 因为暂时可以使用超出 deserved 的资源

步骤3: 创建新队列

创建 test 队列并设置资源比例. 可以选择使用 capacity 插件或 proportion 插件:

```yaml
# 使用 capacity 插件时的队列配置
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: test
spec:
  reclaimable: true
  deserved:
    cpu: 3
```

或

```yaml
# 使用 proportion 插件时的队列配置
apiVersion: scheduling.volcano.sh/v1beta1
kind: Queue
metadata:
  name: test
spec:
  reclaimable: true
  weight: 3 # 资源分配比例为 default:test = 1:3
```

步骤4: 触发资源回收

在 test 队列创建 job3 并申请 3C 资源(配置与 job2 类似, 只需将 queue 改为 test):

```yaml
# job3.yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: job3
spec:
  queue: test # 将队列改为 test
  tasks:
    - replicas: 1
      template:
        spec:
          containers:
            - name: nginx
              image: nginx
              resources:
                requests:
                  cpu: "3"
```

提交 job3 后, 系统开始资源回收:

1. 系统回收 default 队列超出 deserved 的资源
2. job2(3C) 被驱逐
3. job1(1C) 保留运行
4. job3(3C) 开始运行

这个场景同时适用于 capacity plugin 和 proportion plugin:

- capacity plugin: 直接配置 deserved 值(default=1C, test=3C)
- proportion plugin: 配置 weight 值(default=1, test=3)最终计算得到相同的 deserved 值

> 注意: capacity 插件和 proportion 插件必须二选一, 不能同时使用. 选择哪个插件主要取决于您是想直接设置资源量(capacity)还是通过权重自动计算(proportion). Volcano v1.9.0 版本后推荐使用 capacity 插件, 因为它提供了更直观的资源配置方式
