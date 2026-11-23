# KEDA 深度研究报告

## 执行摘要

KEDA（Kubernetes Event-driven Autoscaling）是一个运行在 Kubernetes 之上的事件驱动自动伸缩组件，定位不是替代 Kubernetes 既有自动伸缩能力，而是把“外部事件/业务信号”接入 Kubernetes 的弹性体系中，使 Deployment、StatefulSet、Job 以及带 `/scale` 子资源的自定义资源，能够依据队列长度、流速、监控指标、数据库状态、定时窗口或任意外部信号进行伸缩，并支持经典 HPA 难以原生实现的 scale-to-zero。KEDA 已是 CNCF Graduated 项目；截至官方最新文档版本，KEDA v2.19 为 latest，并测试覆盖 Kubernetes v1.32–v1.34。 citeturn25search14turn31search3turn6view0turn16search0

从体系结构看，KEDA 的核心是 **Operator + Metrics Adapter + Admission Webhooks + Scalers + CRDs**。其中 Operator 负责监听 ScaledObject/ScaledJob、判定 0↔1 激活、生成并维护 HPA；Metrics Adapter 通过 `external.metrics.k8s.io` 向 Kubernetes/HPA 暴露外部指标；1↔N 的伸缩实质仍由 HPA 控制环执行。默认条件下，`pollingInterval` 为 30 秒、HPA 默认同步周期为 15 秒、`cooldownPeriod` 为 300 秒，因此 KEDA 的“快慢”主要由事件源轮询、HPA 控制周期、镜像拉取、Pod 启动、节点供给和应用初始化共同决定，而不是由 KEDA 单独决定。 citeturn10view0turn10view1turn13view0turn37view1

KEDA 的优势在于：第一，直接支持 70+ 内建 scaler，且仍可通过 `external` / `external-push` 走 gRPC 协议扩展自定义 scaler；第二，把“是否激活（0↔1）”与“扩到多少（1↔N）”拆成两个阶段，既能低成本空闲归零，也能复用 HPA 的成熟机制；第三，既可扩 Deployment/StatefulSet，也可扩 Job；第四，可通过 TriggerAuthentication / ClusterTriggerAuthentication 复用认证、接入云工作负载身份与外部密钥系统。 citeturn6view0turn21view0turn22view0turn19view0turn20view0

对生产环境的结论很明确：**KEDA 最适合事件驱动消费者、异步 worker、批处理、定时扩缩、基于 Prometheus 或云监控指标的业务伸缩；不适合作为节点伸缩器，也不应把它理解为 HTTP 请求代理本身。** 对公网 HTTP 服务，如果目标是“真正的零副本唤醒 + 请求拦截”，应评估 KEDA HTTP Add-on 或 Knative；对计算资源右尺寸，应与 VPA 配合；对节点供给，应与 Cluster Autoscaler/Karpenter 一类节点自动伸缩器配合。 citeturn8search1turn37view3turn37view4turn37view0turn37view2

需要特别指出的是：截至目前，KEDA 社区**没有统一、权威、可对外引用的官方性能基准结论**；维护者在公开 issue 中明确表示，KEDA 的性能受外部依赖与 HPA 路径强烈影响，难以形成统一基准。官方目前提供的是 `kedacore/keda-performance` 测试仓库，用 Grafana k6、Prometheus 与故障注入扩展搭建可重复测试框架，而非产品级 SLA。实务上，建议将 KEDA 的性能评估建立在自身场景压测之上，而不是引用单一行业数字。 citeturn27view0turn26view0

## 定义、目标与定位

KEDA 的官方定义是“基于 Kubernetes 的事件驱动自动缩放器”。它的目标不是改写 Kubernetes 自动伸缩体系，而是在 HPA 之上补齐“面向外部事件和业务信号伸缩”的能力。官方首页与仓库说明都强调：KEDA 是 **single-purpose、lightweight** 组件，可与现有 Kubernetes 应用并行运行，并通过自定义资源显式指定哪些应用使用事件驱动弹性，而非全局接管集群。 citeturn31search11turn25search14turn39search3

这一定义决定了它与 HPA/VPA/节点自动伸缩器的边界。HPA 负责副本数横向伸缩，VPA 负责 Pod 请求/限制的纵向右尺寸，节点自动伸缩器负责节点供给；而 KEDA 负责把队列、流、外部 API、监控系统、云服务、数据库或时间窗口等“非 Pod 内部资源指标”接入控制环，并在 0↔1 阶段直接做激活/停用控制。换言之，KEDA 的本质是 **“事件源适配层 + HPA 外部指标桥接层 + CRD 控制层”**。 citeturn13view0turn37view0turn37view1turn37view2

从场景匹配看，KEDA 最典型的价值在于解决三类问题：其一，**按队列积压或流延迟扩 worker**；其二，**按业务指标或监控查询扩服务**；其三，**空闲归零**。这也是为什么云厂商中文材料通常把它定位为“事件驱动弹性伸缩”，并强调除了 CPU/内存之外，还可以基于消息队列、QPS、Cron 等信号伸缩，甚至缩到 0。AKS 中文文档与腾讯云中文文档都在这一点上与官方主文档保持一致。 citeturn31search0turn31search2turn31search4turn28search7

## 架构、组件、伸缩流程与连接器生态

KEDA 的核心组件由文档明确分为几部分：**Operator、Metrics Adapter、Admission Webhooks、Scalers、CRDs**。Operator 负责监视 ScaledObject/ScaledJob、启动/停用 scaler、在需要时直接把目标资源从 0 拉起，并创建/维护对应 HPA；Metrics Adapter 以聚合 API 的方式向 Kubernetes 暴露外部指标；Admission Webhooks 负责校验配置；Scaler 则是连向 Kafka、RabbitMQ、Prometheus、云监控或自定义 gRPC 服务的适配器。官方概念文档还说明，KEDA 可伸缩 Deployment、StatefulSet 和带 `/scale` 子资源的自定义资源。 citeturn28search7turn35search0turn13view0turn15view0

```mermaid
flowchart LR
  ES[外部事件源\nKafka/RabbitMQ/Prometheus/Cloud API/DB/Cron] --> S[Scaler 或 External Scaler]
  SO[ScaledObject / ScaledJob / TriggerAuthentication] --> OP[KEDA Operator]
  S --> OP
  OP --> HPA[生成并维护 HPA]
  OP --> GRPC[gRPC Metrics Service]
  GRPC --> MA[KEDA Metrics Adapter]
  MA --> API[external.metrics.k8s.io]
  API --> HPA
  HPA --> TGT[Deployment / StatefulSet / Custom Resource]
  OP --> JOB[创建 Kubernetes Jobs]
  AW[Admission Webhooks] --> SO
```

上图是依据 KEDA 官方概念、Metrics Server、Cluster operate 文档综合整理的逻辑图：Operator 与 Metrics Adapter 之间通过 gRPC 指标服务协作，Adapter 再把指标暴露给 Kubernetes API Server/HPA；默认 gRPC 指标服务地址为 `keda-operator.keda.svc.cluster.local:9666`。 citeturn16search0turn15view0turn35search0

在 CRD 层，KEDA 当前主线仍以 `keda.sh/v1alpha1` 暴露资源。最重要的对象包括：`ScaledObject`（面向 Deployment/StatefulSet/CustomResource 的伸缩定义）、`ScaledJob`（面向 Job 的事件驱动作业伸缩）、`TriggerAuthentication`（命名空间级认证复用）、`ClusterTriggerAuthentication`（集群级认证复用）。`TriggerAuthentication` 只能在同一命名空间内被引用，而 `ClusterTriggerAuthentication` 可跨命名空间复用；默认情况下，`ClusterTriggerAuthentication` 通过 `secretTargetRef` 加载的 Secret 需要位于 KEDA 部署命名空间，除非显式设置 `KEDA_CLUSTER_OBJECT_NAMESPACE`。 citeturn23view0turn19view0turn20view0turn20view1

KEDA 的伸缩判定需要分成两个阶段理解。**第一阶段是激活阶段**：当目标副本数为 0 时，Operator 按 `pollingInterval` 轮询 trigger，调用 scaler 的 `IsActive`/`GetMetricsAndActivity`，判断是否应从 0 拉到 1；默认 `pollingInterval` 为 30 秒。**第二阶段是扩缩阶段**：一旦副本数已大于等于 1，HPA 会按自身同步周期（默认 15 秒）去查询 KEDA 暴露的指标，并据此在 1↔N 范围内调节副本数；KEDA 则继续负责与事件源交互和维护 HPA。对于缩回 0，`cooldownPeriod` 默认 300 秒，只对回到 0 的路径生效。 citeturn10view0turn10view1turn13view0turn37view1

```mermaid
flowchart TD
  A[事件到达或定时窗口开启] --> B{当前副本是否为 0}
  B -- 是 --> C[KEDA Operator 按 pollingInterval 轮询 scaler]
  C --> D{IsActive 或 activationThreshold 是否满足}
  D -- 否 --> E[保持 0 或 idleReplicaCount]
  D -- 是 --> F[直接激活到 minReplicaCount 或 1]
  F --> G[KEDA 创建/维护 HPA]
  G --> H[HPA 按 sync period 拉取 external.metrics]
  H --> I[按 target/metricType 计算 1↔N 副本]
  I --> J{最后一次活动后是否超过 cooldownPeriod}
  J -- 否 --> H
  J -- 是 --> K[Operator 缩回 0]
  B -- 否 --> H
```

在 `ScaledObject` 中，几个字段非常关键：`pollingInterval` 默认 30 秒；`cooldownPeriod` 默认 300 秒；`minReplicaCount` 默认 0；`maxReplicaCount` 默认 100；`idleReplicaCount` 必须小于 `minReplicaCount`，且由于 HPA 控制器限制，当前唯一受支持值实际是 `0`；`fallback` 可在指标拉取连续失败达到阈值后把目标副本数钉到一个保底值；`advanced.horizontalPodAutoscalerConfig.behavior` 允许直接把 HPA `behavior` 写透给生成的 HPA。还要特别注意：当 `minReplicaCount >= 1` 时，激活阈值会被忽略，因为对象被视为始终处于活动状态。 citeturn10view3turn10view4turn14view0turn14view1turn13view0

`ScaledJob` 的决策逻辑与 `ScaledObject` 不同，它不是协调 HPA，而是直接按轮询结果创建 Job。官方规格说明中给出：`pollingInterval` 默认 30 秒，`maxReplicaCount` 表示单个轮询周期内最多创建的 Job 数；默认策略下，近似计算为 `maxScale - runningJobCount`，其中 `maxScale = min(maxReplicaCount, ceil(queueLength / targetAverageValue))`。它还支持 `default/custom/accurate/eager` 四类 `scalingStrategy`，以及 `multipleScalersCalculation` 在多个 trigger 间选择 `max/min/avg/sum`。这使它非常适合“每条消息一个 Job”或“消费即完成”的批处理模型。 citeturn11view0turn11view1turn11view2turn11view4

从连接器生态看，KEDA 2.19 官方目录列出了 70+ 内建 scaler，并支持两种扩展路径：`external` 与 `external-push`。两者都通过 gRPC 协议对接外部 scaler；其中 `external-push` 还支持 `StreamIsActive`，可在轮询之外主动把“激活事件”推给 KEDA，从而绕过固定轮询的上限，适合对低延迟激活要求较高的场景。 citeturn6view0turn21view0turn22view0

下表按“协议/服务范式”归类整理 KEDA 2.19 官方 scaler 目录，并单独标出 HTTP Add-on：

| 类别 | 代表协议/服务 | 官方 scaler 列表 |
|---|---|---|
| 消息队列与流处理 | AMQP、Kafka 协议、MQ、Pub/Sub、队列积压、流延迟 | ActiveMQ、ActiveMQ Artemis、Apache Kafka、Apache Kafka Experimental、Apache Pulsar、AWS Kinesis、AWS SQS、Azure Event Hubs、Azure Service Bus、Azure Storage Queue、Beanstalkd、GCP Cloud Tasks、GCP Pub/Sub、IBM MQ、Liiklus Topic、NATS JetStream、NSQ、RabbitMQ Queue、Redis Lists、Redis Streams、Solace PubSub+、Temporal |
| 数据与存储 | 数据库计数、流、查询结果、对象存储 | ArangoDB、AWS DynamoDB、AWS DynamoDB Streams、Azure Blob Storage、Azure Data Explorer、Cassandra、CouchDB、Elasticsearch、Etcd、GCP Storage、MongoDB、MSSQL、MySQL、OpenStack Swift、PostgreSQL、Solr |
| 监控与分析 | PromQL、REST Metrics、监控平台 API、日志/时序指标 | AWS CloudWatch、Azure Application Insights、Azure Log Analytics、Azure Monitor、Datadog、Dynatrace、Graphite、Huawei Cloudeye、InfluxDB、Loki、Metrics API、New Relic、OpenStack Metric、Predictkube、Prometheus、SolarWinds、Splunk、Splunk Observability、Sumo Logic |
| Kubernetes 与调度 | 资源利用率、时窗、K8s 资源状态 | CPU、Memory、Cron、Kubernetes Resource、Kubernetes Workload |
| CI/CD 与开发平台 | Pipeline 队列、代码托管任务 | Azure Pipelines、Forgejo、GitHub Runner Scaler |
| 测试与扩展 | 外部 gRPC、推送式激活、测试平台 | External、External Push、Selenium Grid Scaler |

上述列表取自 KEDA 2.19 官方 scalers 目录；HTTP 请求的“按请求到达触发 scale-to-zero”并不属于核心 scaler 目录，而是由 **KEDA HTTP Add-on** 提供。HTTP Add-on 的 0.14 文档显示它由 **operator、interceptor、scaler** 组成，并提供 HTTP 入口请求统计与唤醒能力。官方导航同时标注 `HTTPScaledObject` 已进入 deprecated 状态，并引入 `InterceptorRoute` 作为更新的对象模型。 citeturn6view0turn8search1

认证连接器方面，官方列出的 provider 包括：AWS IRSA、AWS EKS Pod Identity、AWS Secret Manager、Azure AD Workload Identity、Azure Key Vault、Bound Service Account Token、ConfigMap、Environment Variable、File Path、GCP Secret Manager、GCP Workload Identity、Hashicorp Vault、Secret。工程上，这些 provider 决定了 KEDA 是否能在不把明文凭据塞进 ScaledObject 的前提下访问事件源。 citeturn6view0turn19view0

## 部署、配置、兼容性与升级

本报告默认面向**通用、符合 Kubernetes 规范的集群**，不预设 AKS、EKS、GKE、OpenShift、TKE 或其他特定发行版；当涉及平台特性时会单独说明。KEDA 官方当前支持的主要安装方式是 **Helm、Operator Hub、YAML 清单**，并在 MicroK8s 上提供便捷路径。中文环境下，如果使用 AKS 或腾讯云 TKE，可参考其中文文档：AKS 提供托管 KEDA Add-on，TKE 也给出了中文部署与最佳实践文档。 citeturn33view0turn34view1turn31search0turn31search13turn31search9

最常见的安装方式仍然是 Helm。官方命令是：

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

如果偏好原生清单，官方建议直接应用发布页的 `keda-2.19.0.yaml` 或 `keda-2.19.0-core.yaml`，并使用 `--server-side` 以更平稳地处理 CRD 和 webhooks。 citeturn34view0turn34view2

在 chart 可配置项上，当前值文件里几个生产最有价值的选项是：`crds.install`、`watchNamespace`、`rbac.create`、`rbac.aggregateToDefaultRoles`、`rbac.enabledCustomScaledRefKinds`、`rbac.scaledRefKinds`、`networkPolicy.enabled`、`operator.replicaCount`、`serviceAccount.*.annotations`、`logging.*`、`permissions.operator.restrict.secret`、`permissions.metricServer.restrict.secret` 以及默认安全上下文。官方 chart 还强调：Operator 多副本只能减少故障切换停机，并不会提升 KEDA 的伸缩吞吐。 citeturn5view0turn5view1turn40view0turn40view2turn40view4

下面是一份更接近生产硬化的 `values.yaml` 片段，重点展示**命名空间收敛、RBAC 缩权、网络策略、日志格式、Secret 访问收敛**：

```yaml
crds:
  install: true

watchNamespace: "payments,ops"

rbac:
  create: true
  aggregateToDefaultRoles: false
  enabledCustomScaledRefKinds: false

networkPolicy:
  enabled: true
  flavor: kubernetes

operator:
  replicaCount: 2

serviceAccount:
  operator:
    create: true
    name: keda-operator
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/keda-operator
  metricServer:
    create: true
    name: keda-metrics-server
  webhooks:
    create: true
    name: keda-webhook

logging:
  operator:
    level: info
    format: json
  metricServer:
    zapLevel: info
    zapEncoder: json
  webhooks:
    level: info
    format: json

permissions:
  operator:
    restrict:
      secret: true
      namesAllowList:
        - rabbitmq-conn
        - prometheus-basic-auth
  metricServer:
    restrict:
      secret: true
```

该片段中的键名均来自官方 chart `values.yaml`；实际使用时应把 IAM 注解、命名空间和 Secret 白名单改成你的环境值。 citeturn5view0turn5view1turn40view0turn40view2turn40view4

兼容性上，KEDA 官方 `operate/cluster` 文档采用 **Kubernetes N-2** 测试窗口，近期版本矩阵如下：

| KEDA 版本 | 官方测试 Kubernetes 版本 |
|---|---|
| v2.19 | v1.32 – v1.34 |
| v2.18 | v1.31 – v1.33 |
| v2.17 | v1.30 – v1.32 |
| v2.16 | v1.29 – v1.31 |
| v2.15 | v1.28 – v1.30 |
| v2.14 / v2.13 | v1.27 – v1.29 |
| v2.12 | v1.26 – v1.28 |

上表来自官方兼容矩阵；如果你的集群仍停留在更低小版本，应反向选择合适的 KEDA 版本，而不要直接追 latest。 citeturn16search0

升级时，最重要的历史分界是 **v1 → v2** 与 **Helm chart 2.2.1**。v1 到 v2 的迁移中，CRD API 组从 `keda.k8s.io/v1alpha1` 变为 `keda.sh/v1alpha1`；`ScaledObject.spec.scaleTargetRef.deploymentName` 改为 `name`；`containerName` 改为 `envSourceContainerName`；Job 伸缩从 `ScaledObject + scaleType: job` 迁移成独立 `ScaledJob` 对象，并去掉若干旧字段，新增 `successfulJobsHistoryLimit` 与 `failedJobsHistoryLimit`。Helm 方面，官方说明从 v2.2.1 起 chart 开始自动管理 CRD，这会影响从更老 Chart 版本升级的流程。 citeturn23view0turn34view4turn8search2

升级治理策略上，KEDA governance 文档给出比较清晰的预期：运行时功能的弃用期通常为 **6 个月/2 个 release**；CRD 的 breaking change 只有在引入新的 `apiVersion` 时才允许发生；实验性功能可以在无弃用周期前提下调整，不建议直接作为生产依赖。这个策略对做大规模集群标准化非常重要。 citeturn24view0

## 与替代方案对比及性能分析

先给结论：KEDA 最适合作为 **“工作负载级事件驱动伸缩层”**；HPA 是更基础的副本调节器；VPA 是资源右尺寸器；Cluster Autoscaler 是节点供给器；Knative 则是面向请求驱动服务平台的一整套控制面；自定义 controller 则提供最大自由度，但也最容易把复杂性和维护债搬回自己团队。下面的“延迟、开销、复杂度”列是基于各组件官方机制的分析性判断，而不是统一基准测试结果。 citeturn13view0turn37view0turn37view1turn37view2turn37view3turn37view4

| 方案 | 核心能力 | 是否支持缩到 0 | 决策延迟特征 | 资源开销 | 实施复杂度 | 更适合的用例 |
|---|---|---|---|---|---|---|
| KEDA | 外部事件/业务信号接入，生成并维护 HPA，支持 Job 与 CRD | 支持 | 0→1 常受 `pollingInterval` 约束；1→N 受 HPA sync 周期约束；若用 `external-push` 可缩短激活等待 | 低到中，新增 Operator/Adapter/Webhook 三类组件 | 中 | 队列消费者、异步 worker、Cron、Prometheus/云监控驱动伸缩 |
| Kubernetes HPA | 基于资源、自定义或外部指标的副本伸缩 | 原生 HPA 不擅长 scale-to-zero | 默认 15 秒同步周期，控制环成熟稳定 | 低 | 低 | CPU/内存驱动 Web 服务、已有 metrics pipeline 的常规应用 |
| Kubernetes VPA | 自动调节 Pod 请求/限制 | 不解决副本归零 | 通常慢于 HPA，因为核心目标是 rightsizing，且可能涉及 Pod 重建 | 中 | 中 | 资源右尺寸、长期稳定负载、降低 request 偏差 |
| Cluster Autoscaler | 按 unschedulable Pod 自动加减节点 | 节点组可到 0，但不是工作负载副本级 scale-to-zero | 通常慢于 Pod 级伸缩，受云主机创建/回收时间影响 | 中 | 中 | 节点供给、与 HPA/KEDA 联动保证有地方调度新 Pod |
| Knative Serving | 请求驱动服务平台，KPA/HPA，默认请求流量场景 scale-to-zero | 支持 | 对 HTTP/请求驱动更友好，配套 activator/route 机制 | 中到高 | 高 | 公网/内网请求驱动服务、Serverless 风格平台 |
| 自定义 Controller | 自定义一切 | 取决于实现 | 取决于实现 | 取决于实现 | 很高 | 内建 scaler 覆盖不到、且必须实现专有控制语义的场景 |

对比所依据的官方机制包括：KEDA 通过 HPA 处理 1↔N、HPA 默认 15 秒同步、VPA 单独调整 requests/limits、节点自动伸缩器负责 provision/consolidation、Knative KPA 支持 scale-to-zero 而其 HPA 后端不支持。 citeturn13view0turn37view0turn37view1turn37view2turn37view3turn37view4

性能上，如果不做额外优化，KEDA 的几个“先天时钟”决定了系统响应上界：当副本为 0 时，默认每 30 秒轮询一次 trigger；副本已大于等于 1 后，HPA 默认每 15 秒拉一次指标；缩回 0 默认在最后一次活动后等待 300 秒；许多走 HTTP 的 scaler 默认请求超时为 3 秒。官方生产资源建议也给出了基线：Admission Webhooks、Metrics Server、Operator 在默认 YAML 中的 request 均为 `100m CPU / 100Mi Memory`，limit 为 `1 CPU / 1000Mi Memory`。 citeturn10view0turn10view1turn16search0

KEDA 自身还提供几个与“规模化使用”强相关的调参点：`--kube-api-qps` 默认 20、`--kube-api-burst` 默认 30；`KEDA_SCALEDOBJECT_CTRL_MAX_RECONCILES` 默认 5、`KEDA_SCALEDJOB_CTRL_MAX_RECONCILES` 默认 1。官方 troubleshoot 文档明确指出，在 **400+ ScaledObjects** 的环境里，默认值可能过低，建议逐步提高并同时监控 API Server 负载。换言之，KEDA 真正的大规模性能瓶颈，很多时候不是 scaler 算法，而是 **Kubernetes API Server、metrics pipeline、事件源 API 配额、节点供给和应用冷启动**。 citeturn8search2turn16search0turn39search8

关于“公开 benchmark”，截至目前没有可直接当作结论引用的官方统一成绩单。维护者在公开 issue 中明确写到：社区经常被问及 load testing / performance benchmarking，但由于 KEDA 依赖大量外部系统且又依赖 HPA，很难做出通用结论。另一方面，官方已经开源 `kedacore/keda-performance` 仓库，其基于 Grafana k6、Prometheus 与故障注入扩展来测试 KEDA core，并在示例配置中给出了诸如 `keda_internal_latency` `p95 < 7`、`p99 < 15`、`max < 20` 这类阈值，但这应被理解为**测试框架中的验收门槛示例**，不是产品 SLA。 citeturn27view0turn26view0

因此，更合理的测试方法是把性能拆成五类指标：**激活延迟（事件到 0→1）、扩容收敛时间（1→N）、队列排空时间、节点供给时间、控制面自身延迟/错误率**。场景上至少要覆盖队列型（RabbitMQ/Kafka/SQS）、监控型（Prometheus）、自定义 gRPC `external-push`、以及节点不足触发 Cluster Autoscaler 的联动场景；同时记录 KEDA `/metrics`、Adapter `/metrics`、API Server 请求延迟、队列 backlog、应用 readiness 和节点 ready 时间。若采用官方 `keda-performance` 仓库，可直接参数化“ScaledObjects 数量”和“每个对象绑定的指标数”，再配合故障注入验证在指标源抖动、认证失败、网络延迟情况下的退化路径。 citeturn26view0turn9search1turn15view0turn21view0

真实案例方面，可看到 KEDA 已进入较大规模生产：CNCF HDFC Bank 案例写到其 Governor 应用在 GKE 上使用 Dapr metrics + KEDA ScaledObject 按 HTTP 流量伸缩，业务背景约为 **7.5 亿笔/月** UPI 交易；Infosys 客户案例写到其在 Azure 主机重托管中，用 KEDA 伸缩在线容器，并取得 **25% 开发周期下降** 与 **60% 年度 TCO 节省**；Cloudchipr 客户案例则把 KEDA 用在队列深度与 stream lag 上，让后台 worker 在空闲时缩到 0；KEDA 项目页当前还展示了 SNOW Corp. 的案例，提到 **1000+ A100 GPU、2 亿+ 用户、700% 流量峰值**。官方博客还曾以标题形式公开介绍“Why Alibaba Cloud uses KEDA for application autoscaling”，说明其在头部云厂商生态中已有实际采用。 citeturn30view2turn30view3turn30view4turn29search4turn39search0

## 生产实践、安全与故障排查

生产实践上，最先要做的是把 KEDA 当成“控制面组件”而不是简单的 sidecar。官方文档给出了可观测性基础设施：Operator、Admission Webhooks、Metrics Adapter 都在 `:8080/metrics` 暴露 Prometheus 指标，官方还提供了预制 Grafana dashboard；同时，Metrics Server 的外部指标可以直接通过 `kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1"` 查询，`ScaledObject.status.externalMetricNames` 也可直接导出当前对象暴露的指标名。对调试伸缩异常，这比盯 Pod 数量变化更有效。 citeturn9search1turn15view0

在日志与运行时方面，KEDA 使用 zap；`operator`、`metrics-apiserver`、`webhooks` 都支持日志级别/编码/时间编码配置。官方 chart 的默认安全上下文也已经比较收敛：`capabilities.drop: [ALL]`、`allowPrivilegeEscalation: false`、`readOnlyRootFilesystem: true`、`seccompProfile: RuntimeDefault`、Pod `runAsNonRoot: true`。如果你通过 Helm 分发给多团队复用，建议把这组默认值视为“不可回退”的平台基线。 citeturn16search0turn40view4

多租户上，最关键的不是“能否跨 namespace 缩容”，而是“谁能读到谁的 Secret、谁能缩谁的工作负载”。KEDA 默认会 watch 全集群 namespace，但可用 `WATCH_NAMESPACE` 或 Helm 的 `watchNamespace` 收敛到一个或多个命名空间；`TriggerAuthentication` 天生是命名空间级隔离，而 `ClusterTriggerAuthentication` 是全局对象，适合平台团队统一管理，但同时也放大了误配影响面。对于多团队平台，推荐优先：**命名空间级 TriggerAuthentication + watchNamespace 收敛 + `permissions.*.restrict.secret` 开启 + `rbac.scaledRefKinds` 白名单化**。 citeturn16search0turn5view0turn20view0turn40view0

在 RBAC 和网络安全上，KEDA 文档的态度非常明确：默认 RBAC 为了支持缩放任意带 `/scale` 的资源，会授予很宽的 `*/scale` 与 `* get` 权限；如果直接照搬默认值进入大生产，很容易突破最小权限原则。官方建议通过 `rbac.enabledCustomScaledRefKinds` 和 `rbac.scaledRefKinds` 进行收敛，同时审查最终渲染出来的角色。网络层面，chart 支持 `networkPolicy.enabled`，并可按 Cilium/Kubernetes 两种 flavor 加额外 egress 规则；对需要走外部 API 的 scaler，Operator 和 Metrics Adapter 都支持代理、TLS 最低版本（默认 TLS 1.2）和禁用 Keep-Alive 的全局设置。 citeturn16search0turn5view1turn40view2

下表给出一份更贴近生产的检查清单：

| 维度 | 建议 |
|---|---|
| 可观测性 | 抓取 Operator / Metrics Adapter / Webhooks 三类 `/metrics`，配套 Grafana，看外部指标、gRPC 延迟与 HPA 结果是否一致 |
| 缩放参数 | 先保守设定 `pollingInterval`、`cooldownPeriod`、`min/maxReplicaCount`，再按场景收紧 |
| 大规模对象数 | 当 ScaledObjects 达到数百量级时，评估 `kube-api-qps`、`kube-api-burst` 和 `KEDA_SCALEDOBJECT_CTRL_MAX_RECONCILES` |
| 多租户 | 收缩 `watchNamespace`，优先使用命名空间级 `TriggerAuthentication` |
| Secret 与 RBAC | 开启 Secret 限制，最小化 `scaledRefKinds`，审查最终 Role/ClusterRole |
| HTTP/公网服务 | 若要求真正的 HTTP scale-to-zero 唤醒体验，优先考虑 HTTP Add-on 或 Knative，而不是仅靠普通 Prometheus 指标 |
| 长任务 | 长耗时消息处理不要只靠 Deployment + HPA；要么利用 SIGTERM / `terminationGracePeriodSeconds`，要么切换到 `ScaledJob` |
| 服务网格 | 在 Istio 下要显式排除 KEDA 组件必须通信端口，避免内部探测与 mTLS 误伤 |

该清单来自官方运维、集成与故障排查文档的生产向抽象。 citeturn9search1turn16search0turn35search5turn13view0turn8search1

常见故障上，优先排这几类。第一类是 **`external.metrics.k8s.io` 不可用**：在自建集群上常见于 `no_proxy` 或 APIService 证书/聚合链路问题；在 GKE 私有集群上要特别确认控制面到 6443 的放行与 `kube-system` 中 `konnectivity` 相关网络策略。第二类是 **认证错误**，典型日志是 AWS 场景里的 `NoCredentialProviders`，应先核查 `keda-operator` Deployment 注解、ServiceAccount 注解与 IRSA/KIAM 是否配置在正确位置。第三类是 **客户端节流**，当日志中出现 `Waited for … due to client-side throttling`，要查看对象规模、QPS/Burst 与 API Server 负载，而不是只盯业务队列。 citeturn35search3turn9search3turn8search2

第四类是 **升级与清理问题**。从老 Helm chart 升到 2.2.1+ 时，可能因为 CRD 管理方式变化而失败；删除 KEDA 资源时，也可能因为 finalizer 挂住导致对象删不掉，官方文档给出的是直接 `kubectl patch scaledobject|scaledjob ... finalizers: null`。第五类是 **“为什么没缩到 0”**：常见原因包括 `minReplicaCount >= 1`、激活阈值设置、还有 `idleReplicaCount` 误以为可配成非 0；官方规格明确说明，后者当前唯一受支持值就是 0。 citeturn34view4turn34view1turn10view3turn13view0

## 迁移建议与典型 YAML

从现有 HPA 迁移到 KEDA，推荐用 **“先包裹，后替换，最后再启用缩零”** 的节奏。先把现有的 CPU/内存或业务指标映射到 KEDA 的 trigger 模型，再通过 `scaledobject.keda.sh/transfer-hpa-ownership: "true"` 与 `advanced.horizontalPodAutoscalerConfig.name` 接管现有 HPA；等 1↔N 行为与旧 HPA 对齐后，再考虑把 `minReplicaCount` 降到 0，并引入 activation threshold 与 cooldown 调优。这样做的好处是：旧 HPA 行为、命名与历史告警不会一次性全部失效。需要注意的是，KEDA 官方明确说明：若 `minReplicaCount >= 1`，activation 值会被忽略；所以“先不缩零”阶段本身就是一种过渡保护。 citeturn13view0turn14view0

如果现有 HPA 只是基于 CPU/内存，是否一定要迁到 KEDA，要看平台目标。如果你希望平台统一都用 `ScaledObject` 作为入口，KEDA 2.x 已内建 CPU/Memory scaler，可以把“所有自动伸缩对象统一收敛到 KEDA CRD”作为治理目标；否则，保留原生 HPA 完全合理。官方 2.0 发布说明就明确强调，CPU/Memory scaler 的加入是为了避免用户在 HPA 与 ScaledObject 之间来回混搭。 citeturn29search5turn6view0

下面给出一个**队列型 worker**的典型示例，以 RabbitMQ 为例，推荐使用 `TriggerAuthentication` 隔离连接串，而不是把凭据直接写进 `ScaledObject`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-secret
  namespace: default
type: Opaque
stringData:
  host: amqp://guest:password@rabbitmq.default.svc.cluster.local:5672/%2f
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: rabbitmq-auth
  namespace: default
spec:
  secretTargetRef:
    - parameter: host
      name: rabbitmq-secret
      key: host
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: orders-worker
  namespace: default
spec:
  scaleTargetRef:
    name: orders-worker
  pollingInterval: 15
  cooldownPeriod: 120
  minReplicaCount: 0
  maxReplicaCount: 50
  fallback:
    failureThreshold: 3
    replicas: 2
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: orders
        mode: QueueLength
        value: "20"
      authenticationRef:
        name: rabbitmq-auth
```

这个示例对应 RabbitMQ scaler 当前推荐字段：`mode: QueueLength` 与 `value`，因为旧的 `queueLength` 字段已被标记为 deprecated。`fallback` 则用于在指标源短时不可达时保底副本。 citeturn41search0turn10view4

对 **HTTP 服务**，如果只是基于 Prometheus 中的 HTTP 指标进行伸缩，核心 KEDA 已足够：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: web-http-metrics
  namespace: default
spec:
  scaleTargetRef:
    name: web
  minReplicaCount: 1
  maxReplicaCount: 10
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc.cluster.local:9090
        query: sum(rate(http_requests_total{job="web"}[1m]))
        threshold: "5"
```

这类方案适合“副本大于等于 1 时”的常规流量控制。如果你的目标是**HTTP 请求触发的真实 scale-to-zero 唤醒**，则应评估 KEDA HTTP Add-on，因为它引入了 operator/interceptor/scaler 三类组件专门处理请求拦截与唤醒。 citeturn33view0turn8search1

对 **Cron 定时伸缩**，官方推荐做法不是把 `desiredReplicas` 设成 0，而是把 `minReplicaCount` 设为 0、在工作时间窗口把“动态最小副本”抬起来：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: office-hours-api
  namespace: default
spec:
  scaleTargetRef:
    name: office-hours-api
  minReplicaCount: 0
  maxReplicaCount: 4
  cooldownPeriod: 300
  triggers:
    - type: cron
      metadata:
        timezone: Asia/Shanghai
        start: 0 9 * * 1-5
        end: 0 18 * * 1-5
        desiredReplicas: "1"
    - type: cpu
      metricType: Utilization
      metadata:
        value: "80"
```

这个模式在上班时间内至少维持 1 个副本，并允许 CPU trigger 在 1–4 之间继续扩展；下班后再归零。官方文档同时提醒：Cron scaler 的 `start` 与 `end` 使用 Linux cron 表达式，且两者不能相同。 citeturn42search0

如果你准备用 KEDA 接管现有 HPA，可用下面的**迁移式 ScaledObject**骨架：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: legacy-api
  namespace: default
  annotations:
    scaledobject.keda.sh/transfer-hpa-ownership: "true"
spec:
  scaleTargetRef:
    name: legacy-api
  minReplicaCount: 1
  maxReplicaCount: 20
  advanced:
    horizontalPodAutoscalerConfig:
      name: legacy-api
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-server.monitoring.svc.cluster.local:9090
        query: sum(rate(http_requests_total{job="legacy-api"}[1m]))
        threshold: "10"
```

这个写法的关键是：**先保留 `minReplicaCount: 1`，把 1↔N 的运行特性对齐，再讨论缩零。** citeturn13view0turn14view0

## 局限与未决问题

第一，KEDA 生态虽成熟，但**公开、统一、可横向比较的官方性能 benchmark 仍然缺位**。现有最靠谱的公开资料是维护者对 benchmark 缺失原因的说明，以及 `kedacore/keda-performance` 提供的测试框架；它们适合做方法学参考，不适合直接拿来当性能承诺。 citeturn27view0turn26view0

第二，**HTTP scale-to-zero 仍是一个需要额外架构设计的主题**。核心 KEDA 更擅长基于现成指标做伸缩；若希望请求到达时立即接管流量、隐藏冷启动，需要 HTTP Add-on 或 Knative 类机制。并且 KEDA HTTP Add-on 文档已经把 `HTTPScaledObject` 标为 deprecated，说明这一子生态的 API 仍在演进。 citeturn8search1turn37view4

第三，**托管云上的 KEDA 版本可能落后于上游**。AKS 中文文档就特别提示了其托管 Add-on 与上游 2.15+ 身份机制变更之间的节奏差异；腾讯云也单独维护了 TKE 侧的 KEDA 部署文档。因此，若你运行在特定发行版上，最终标准应以“发行版支持矩阵 + 上游兼容矩阵”的交集为准，而不是只看 upstream latest。 citeturn31search0turn31search13turn16search0