# Harness Engineering 深度研究初稿

> 研究对象：<https://yuanchaofa.com/post/harness-engineering-for-ai-agents>  
> 相关入口：<https://yuanchaofa.com/post>  
> 初稿目的：梳理文章核心观点、上下文脉络、工程启发与可扩展写作方向。

---

## 0. 摘要

《Harness Engineering — Agent 不好用，也许不是模型的问题》这篇文章讨论的是一个正在变得越来越重要的 Agent 工程问题：当 Agent 表现不好时，问题未必首先出在模型能力本身，而可能出在模型外部的运行环境、工具、上下文、权限、反馈循环和验证机制上。

文章用一句话概括 Harness 的定义：

> Agent = Model + Harness

如果不是模型权重本身，那么它就属于 Harness 的一部分。

也就是说，Harness 指的是模型权重之外，所有支撑 Agent 完成任务的系统组件，包括 system prompt、项目级上下文文件、工具、sandbox、上下文压缩、middleware、hooks、sub-agent 管理、测试验证、执行环境和反馈机制等。

文章最有冲击力的案例是：LangChain 在同一个模型 GPT-5.2-Codex 上，只修改模型之外的 Harness，就让 Terminal Bench 2.0 得分从 52.8 提升到 66.5，排名从 Top 30 提升到 Top 5。这个例子说明，Agent 能力不仅取决于模型，还强烈依赖外部系统设计。

本文档将从以下几个角度展开：

1. Harness Engineering 的定义与边界
2. Prompt Engineering、Context Engineering、Harness Engineering 的关系
3. 文章中的关键案例拆解
4. Harness Engineering 背后的工程思想
5. The Bitter Lesson 视角下哪些设计会被淘汰，哪些设计更持久
6. 对 Coding Agent、平台工程和个人 Agent 系统设计的启发
7. 后续可继续扩展的研究方向

---

## 1. 背景：为什么 Agent 不好用，不一定是模型的问题

很多人在使用 Coding Agent 或通用任务 Agent 时，都会遇到类似问题：

- Agent 写完代码后不运行测试；
- 没有理解项目结构就开始修改；
- 在同一个文件中反复小修小补，陷入循环；
- 对自己的修改过度自信，提前结束任务；
- 长任务中上下文越来越混乱，逐渐偏离原目标；
- 工具调用顺序不合理；
- 缺少验证、回滚、审计和恢复机制；
- 遇到错误后只会重复尝试，而不是更换策略。

面对这些问题，直觉反应往往是：

> 模型还不够聪明，等下一代模型就好了。

但文章提出了另一种视角：

> 这不完全是 model problem，而是 harness problem。

也就是说，模型能力固然重要，但模型外部的系统如何组织信息、提供工具、约束行为、反馈错误、执行验证，同样决定 Agent 的最终表现。

这类似于传统软件工程中的一个经验：不能只依赖个体开发者永远不犯错，而要通过类型系统、测试、CI、代码审查、权限控制、灰度发布、监控告警等工程机制降低错误概率。Agent 也是如此，不能只依赖模型每次“想对”，而要通过 Harness 让正确行为更容易发生，让错误行为更难发生。

---

## 2. Harness 是什么

文章引用 LangChain 的定义：

> Agent = Model + Harness. If you're not the model, you're the harness.

换句话说，Harness 是模型权重之外，所有让 Agent 能够执行任务的外部系统。

### 2.1 Harness 的典型组成

| 组件 | 具体形式 | 解决的问题 |
|---|---|---|
| System Prompt | 系统提示词、角色定义、行为规范 | 模型不知道自己应该扮演什么角色 |
| 项目级上下文文件 | CLAUDE.md、AGENTS.md、README、项目规范 | 模型不了解当前项目约定 |
| Tools | shell、文件读写、浏览器、数据库、MCP、K8s API | 模型只能生成文本，不能直接作用于环境 |
| Sandbox | 容器、临时目录、权限隔离、网络限制 | 防止 Agent 破坏真实环境 |
| Context Management | prompt caching、compaction、检索、外部记忆 | 防止上下文膨胀和 context rot |
| Hooks / Middleware | 调用前后拦截、退出前检查、死循环检测 | 纠正模型容易忽略的流程问题 |
| Sub-agent 管理 | 子代理派生、并行探索、结果汇总 | 单一上下文无法承载复杂任务 |
| Verification | 测试、lint、type check、benchmark、review | 模型自我评价不可靠，需要外部验证 |
| Observability | trace、日志、工具调用记录、成本统计 | 为调试和持续优化提供依据 |

### 2.2 Harness Engineering 的核心思想

文章引用 Mitchell Hashimoto 的说法：

> Harness engineering is the idea that anytime you find an agent makes a mistake, you take the time to engineer a solution such that the agent never makes that mistake again.

这句话可以翻译为：

> 每当你发现 Agent 犯了一个错误，就花时间设计一个系统性解决方案，让它以后不再犯同类错误。

这也是 Harness Engineering 的核心：

不是每次出错后手动改 prompt、重新运行、祈祷下次变好，而是把失败模式沉淀为系统约束。

例如：

- Agent 总是忘记跑测试：增加退出前测试检查 hook；
- Agent 经常找不到测试命令：启动时自动探测项目环境；
- Agent 反复修改同一个文件：增加循环检测；
- Agent 容易误删文件：对删除操作加权限确认和备份；
- Agent 经常调用危险命令：把高风险工具放进 sandbox 或 require confirmation；
- Agent 长任务中遗忘目标：把任务目标和验收标准持久化，并周期性重新注入上下文。

从这个角度看，Harness Engineering 更接近可靠性工程，而不是单纯的 prompt 技巧。

---

## 3. Prompt Engineering、Context Engineering 与 Harness Engineering

文章提出，这三个概念不是简单的替代关系，而是逐层扩大的包含关系。

可以粗略理解为：

```text
Prompt Engineering ⊂ Context Engineering ⊂ Harness Engineering
```

### 3.1 Prompt Engineering

Prompt Engineering 关注的是“一条指令如何写”。

典型问题包括：

- 要不要给 few-shot 示例；
- 要不要要求模型一步步思考；
- 如何定义角色；
- 如何指定输出格式；
- 如何描述约束条件；
- 如何减少歧义。

失败时的修复方式通常是：

> 改 prompt。

### 3.2 Context Engineering

Context Engineering 关注的是“整个上下文窗口如何组织”。

它不再只看单条 prompt，而是考虑：

- 哪些信息应该放进上下文；
- 哪些信息应该检索；
- 哪些信息应该压缩；
- 哪些信息应该写入外部存储；
- 哪些任务应该隔离到子上下文；
- 如何减少 context rot；
- 如何利用 prompt caching；
- 如何避免频繁破坏缓存前缀。

LangChain 曾将 Context Engineering 总结为四类策略：

1. Write：把信息写到外部存储；
2. Select：按需选择相关信息；
3. Compress：压缩已有上下文；
4. Isolate：隔离不同任务的上下文。

失败时的修复方式通常是：

> 调整上下文组织方式。

### 3.3 Harness Engineering

Harness Engineering 进一步扩展到上下文窗口之外。

它不仅关心“给模型什么信息”，也关心“让模型运行在什么环境中”。

它关注的问题包括：

- Agent 有哪些工具；
- 工具是否安全；
- 是否有 sandbox；
- 是否有权限控制；
- 是否有测试 gate；
- 是否有退出前 checklist；
- 是否有死循环检测；
- 是否能记录完整执行轨迹；
- 是否支持失败恢复；
- 是否能从历史任务中学习；
- 是否能把失败模式固化为系统规则。

失败时的修复方式是：

> 修改环境和流程，让同类错误更难发生，甚至不可能发生。

### 3.4 为什么 Harness 更适合作为超集

文章提到社区存在争议：

- LangChain 倾向认为 Harness 包含 Context；
- HumanLayer 似乎更倾向认为 Harness 是 Context 的一部分。

从工程角度看，将 Harness 理解为 Context 的超集更合理。

原因是：

- sandbox 不属于上下文管理；
- CI gate 不属于上下文管理；
- linter 不属于上下文管理；
- 权限系统不属于上下文管理；
- 文件系统隔离不属于上下文管理；
- 审计日志不属于上下文管理。

但这些都是 Agent 能否可靠运行的关键组成。因此，Harness Engineering 应该被视为覆盖 Prompt、Context、Tools、Runtime、Verification、Observability 的系统工程概念。

---

## 4. 案例拆解：同一个模型，只改 Harness

文章中最关键的案例来自 LangChain。

基线配置：

- deepagents-cli；
- GPT-5.2-Codex；
- 默认配置；
- Terminal Bench 2.0 得分：52.8。

优化后：

- 同一个模型；
- 主要修改 System Prompt、Tools、Middleware；
- Terminal Bench 2.0 得分：66.5；
- 排名从 Top 30 提升到 Top 5。

提升幅度：

- 绝对提升：13.7 分；
- 相对提升：约 25.95%。

这说明 Harness 不是可有可无的外围配置，而是直接影响 Agent 任务完成率的核心系统。

### 4.1 修改一：退出前强制自检

常见失败模式：

```text
Agent 写完代码
→ 重读一遍自己的代码
→ 觉得“看起来没问题”
→ 直接结束
→ 实际上没有运行测试，也没有验证边界条件
```

Harness 的解决方案：

在 Agent 试图退出时，由 middleware 拦截退出动作，强制注入 checklist，让 Agent 对照原始任务重新验证。

这个机制类似软件工程中的 pre-submit hook。

它不是要求 Agent “记得要测试”，而是把“结束前检查”变成系统流程的一部分。

可能的 checklist 包括：

- 是否完整理解了用户需求；
- 是否修改了正确文件；
- 是否运行了相关测试；
- 测试是否通过；
- 是否检查了边界情况；
- 是否引入不必要复杂度；
- 是否留下临时文件或调试代码；
- 是否说明了风险和后续工作。

### 4.2 修改二：启动时注入环境信息

Agent 进入陌生代码库时，经常浪费大量时间探索环境：

- 项目是什么语言；
- package manager 是什么；
- 测试命令在哪里；
- 入口文件在哪里；
- 是否有虚拟环境；
- README 是否有特殊说明；
- 当前 git 状态如何。

Harness 的解决方案：

在 Agent 启动时自动运行一组环境探测命令，并将结果注入上下文。

例如：

```text
pwd
git status
识别 package.json / go.mod / pyproject.toml / Cargo.toml
读取 README / AGENTS.md / CLAUDE.md
查找测试命令
查看最近修改文件
```

这类机制符合 Just-in-Time Context 思路：

> 在正确的时机注入正确的信息，而不是让模型自己盲目探索。

### 4.3 修改三：死循环检测

Agent 常见的另一类失败是陷入局部循环。

例如：

```text
修改 A 文件
运行测试失败
再修改 A 文件
测试仍失败
继续修改 A 文件
重复十几次后仍然没有本质进展
```

模型未必能稳定感知自己已经在原地打转。

Harness 的解决方案：

通过 hook 追踪文件编辑次数、相同命令失败次数、重复错误信息等指标。如果超过阈值，就向 Agent 注入提醒：

> 你可能陷入了局部循环，请重新审视问题，考虑换一个方案。

这类机制的核心是：

系统层记录比模型自我反省更可靠。

### 4.4 修改四：Reasoning Sandwich

文章提到 GPT-5.2-Codex 有多档推理强度：

- low；
- medium；
- high；
- xhigh。

直觉上似乎全程使用最高推理强度最好，但实验发现并非如此。全程 xhigh 可能因为推理成本和时间过高导致整体效果变差。

更好的策略是：

```text
xhigh → high → xhigh
```

也就是：

- 规划阶段使用更高推理强度；
- 执行阶段使用中等或较高推理强度；
- 验证阶段再次使用高推理强度。

这类似人类工程流程：

- 设计架构时多想；
- 写常规代码时快速执行；
- review 和验收时再次仔细检查。

这个案例说明，Agent 系统不仅要选择模型，还要根据任务阶段动态分配推理预算。

---

## 5. Harness Engineering 背后的工程思想

### 5.1 不相信单点智能

Harness Engineering 的第一原则是：不要假设模型每次都能自觉做对。

这和传统工程类似：

- 不依赖开发者永远不写 bug，所以需要测试；
- 不依赖运维永远不误操作，所以需要权限和审计；
- 不依赖服务永远不失败，所以需要重试和熔断；
- 不依赖网络永远稳定，所以需要超时和幂等。

同理：

- 不依赖 Agent 永远记得跑测试，所以需要退出前检查；
- 不依赖 Agent 永远理解项目，所以需要启动环境探测；
- 不依赖 Agent 永远知道自己陷入循环，所以需要循环检测；
- 不依赖 Agent 永远安全使用工具，所以需要 sandbox 和权限控制。

### 5.2 把失败模式产品化

Agent 犯错后，不应该只是重试，而应该问：

> 为什么系统允许这个错误发生？

这类似 SRE 里的 postmortem 文化。

关注点不是责怪某一次模型输出，而是把错误转化为系统改进：

- 增加一条检查；
- 修改工具接口；
- 增加 sandbox 限制；
- 改善上下文注入；
- 增加 regression task；
- 优化 middleware；
- 记录到未来训练数据中。

### 5.3 把经验固化为流程

很多 Agent 规则不应该只写在 prompt 里。

例如：

- 修改代码前先读相关文件；
- 修改代码后运行测试；
- 删除文件前确认；
- 部署前 dry-run；
- 结束任务前对照验收标准；
- 长任务中定期压缩上下文；
- 高风险工具调用需要人工确认。

这些规则如果只写在 prompt 中，模型可能遗忘、忽略或误解。

更可靠的方法是将其固化为 Harness 的机制。

### 5.4 把 Agent 当作不稳定但强大的执行器

模型不是完美 planner，也不是完美 executor。

更合理的定位是：

> 模型是一个能力很强但不稳定的推理与执行组件。

Harness 的职责是：

- 给它上下文；
- 限制动作空间；
- 提供工具；
- 记录行为；
- 检查输出；
- 提供反馈；
- 隔离风险；
- 支持恢复；
- 将失败沉淀为系统改进。

---

## 6. The Bitter Lesson 视角：什么会被淘汰，什么会持久

文章讨论了 The Bitter Lesson 对 Harness Engineering 的影响。

The Bitter Lesson 的核心观点是：

> 长期来看，依赖通用计算和规模的方法，往往会胜过大量人工设计的启发式规则。

对应到 Agent Harness，这意味着很多当前看起来必要的控制逻辑，可能只是为了弥补当前模型能力不足。

### 6.1 可能会被淘汰的 Harness 设计

以下设计可能随着模型能力提升而逐渐减少：

- 过度复杂的手写 planner；
- 针对单个模型缺陷写的 prompt hack；
- 僵硬的任务状态机；
- 大量人工设计的工具选择规则；
- 过度细碎的工具；
- 为弥补模型短期记忆不足写的复杂提示模板；
- 依赖经验规则的多轮纠偏逻辑。

文章提到：

- Manus 在 6 个月内重构了 5 次 Harness；
- Vercel 删除了 80% 的 Agent 工具后效果反而更好。

这说明 Harness 不是越复杂越好。

好的 Harness 应该 build to delete：

> 为今天的问题提供工程解法，但保持模块化，使其在模型进步后可以被删除或替换。

### 6.2 更可能持久的 Harness 设计

有些设计不是为了弥补模型不够聪明，而是来自物理约束、工程边界和安全要求，因此更可能长期存在。

#### 6.2.1 Sandbox

无论模型多聪明，都不应该无约束访问生产环境。

Sandbox 是安全边界，而不是模型能力补丁。

#### 6.2.2 权限系统

工具调用必须有权限边界。

例如：

- 读文件和删文件风险不同；
- 查询数据库和修改数据库风险不同；
- dry-run 和真实部署风险不同；
- 本地测试和生产操作风险不同。

#### 6.2.3 测试与验证

模型再强，也需要外部 truth signal。

包括：

- 单元测试；
- 集成测试；
- e2e 测试；
- benchmark；
- linter；
- type checker；
- 编译器；
- 安全扫描；
- 人类 review。

#### 6.2.4 可观测性

需要记录：

- 模型输入输出；
- 工具调用；
- 文件修改；
- 失败轨迹；
- token 消耗；
- latency；
- 成本；
- 人类反馈。

没有 trace，就无法复现和改进 Agent 行为。

#### 6.2.5 版本控制

Agent 修改代码必须能 diff、review、rollback。

这不是模型是否聪明的问题，而是软件工程基本要求。

#### 6.2.6 外部记忆

上下文窗口再长，也不等于持久记忆。

文件系统、数据库、事件日志、向量库、任务状态存储仍然重要。

#### 6.2.7 Cache-aware 架构

Prompt Caching / KV Cache 相关设计具有长期价值，因为它们不是单纯弥补模型智力，而是适配当前 Transformer decoder 推理范式下的物理限制。

只要长前缀 prefill、KV Cache 和 memory bandwidth 仍然是推理系统的核心约束，cache-aware 的 prompt 布局、上下文隔离和 compaction 策略就不会轻易过时。

---

## 7. 从文章列表看作者的知识脉络

访问 <https://yuanchaofa.com/post> 后可以看到，作者的博客内容经历了一个比较清晰的演进路径。

早期主题包括：

- 中文分词；
- HMM；
- 倒排索引；
- NER；
- 情绪原因抽取；
- Python 类型系统；
- Git 配置；
- 工具链效率。

近两年重点转向 LLM 和 Agent：

- Transformer；
- RoPE；
- MoE；
- SwiGLU；
- DeepSeek MLA；
- 显存估算；
- KV Cache；
- Prompt Caching；
- DPO；
- RLHF；
- DeepSeek-R1；
- Kimi 系列；
- Gemini 技术报告；
- Agentic RAG。

最近几篇文章形成了一条明显的 Agent System Design 主线：

1. 理解 KV Cache 与 Prompt Caching；
2. Agent 系统中的 Prompt Caching 设计（上）；
3. Agent 系统中的 Prompt Caching 设计（下）；
4. Harness Engineering — Agent 不好用，也许不是模型的问题。

这条线索可以总结为：

```text
KV Cache / Prompt Caching
→ 上下文组织与缓存友好设计
→ Context Compaction / Sub-agent
→ Harness Engineering
→ Agent 可靠性工程
```

因此，这篇 Harness Engineering 不是孤立文章，而是作者从底层推理机制逐渐推导到 Agent 系统工程的一篇阶段性总结。

---

## 8. 一个可落地的 Agent Harness 分层模型

如果把 Harness Engineering 落到真实系统，可以将 Agent Harness 拆成七层。

### 8.1 任务入口层

负责把用户意图转成结构化任务。

包括：

- 任务类型识别；
- 风险等级判断；
- 输入约束；
- 输出格式；
- 验收标准；
- 是否需要人工确认。

例如，用户说“帮我修一个 bug”，系统不应该直接开始改代码，而应该先明确：

- bug 描述；
- 复现步骤；
- 相关日志；
- 目标分支；
- 验收测试；
- 是否允许修改公共接口。

### 8.2 上下文层

负责给模型提供足够但不过量的信息。

包括：

- 项目说明；
- 相关文件；
- 最近 diff；
- 测试结果；
- 依赖关系；
- 历史决策；
- 文档检索；
- 外部知识。

这层主要对应 Context Engineering。

### 8.3 工具层

定义 Agent 可以做什么。

包括：

- 文件读写；
- shell；
- git；
- 浏览器；
- 数据库；
- Kubernetes API；
- CI；
- issue tracker；
- 文档搜索。

工具设计不应该只是暴露裸 shell，更好的方式是提供有语义的、安全边界更清晰的操作。

例如，不只是：

```text
run("kubectl delete pod xxx")
```

而是：

```text
delete_pod(namespace, pod_name, dry_run=true)
```

这样更容易做权限控制、审计和回滚。

### 8.4 执行环境层

负责隔离风险。

包括：

- sandbox；
- container；
- 临时 workspace；
- branch isolation；
- read-only credential；
- secret masking；
- network policy；
- resource quota。

这是生产级 Agent 的基础层。

### 8.5 控制层

负责 Agent loop 的行为控制。

包括：

- 最大步数；
- 最大成本；
- 最大时间；
- 退出前 checklist；
- 死循环检测；
- 重试策略；
- 失败转人工；
- planner/executor 分离；
- reasoning budget 分配。

文章中的 middleware、hooks、Reasoning Sandwich 都属于这一层。

### 8.6 验证层

负责判断结果是否真的正确。

包括：

- 单元测试；
- 集成测试；
- e2e 测试；
- benchmark；
- 类型检查；
- lint；
- 安全扫描；
- diff review；
- smoke test。

这层非常关键，因为 LLM 的自我评价不可靠。

### 8.7 学习层

负责把每次执行变成未来改进的数据。

包括：

- trace logging；
- 成功轨迹；
- 失败轨迹；
- 工具调用序列；
- 人类反馈；
- prompt/harness 版本；
- benchmark 回放；
- 自动生成 regression task。

文章提到的：

> The Harness is the Dataset

可以理解为：

真正有价值的不是某条 prompt，而是 Harness 在真实任务中积累的执行轨迹、失败样本和人类反馈。

---

## 9. 对 Coding Agent 设计的启发

如果要设计或改造一个 Coding Agent，优先应该做的不是无限堆 prompt，而是完善 Harness。

### 9.1 强制退出前检查

Agent 最终回答前必须确认：

- 是否完成原始需求；
- 是否修改了正确文件；
- 是否运行了相关测试；
- 测试是否通过；
- 是否检查了边界情况；
- 是否引入不必要复杂度；
- 是否留下临时文件；
- 是否说明了风险和后续工作。

### 9.2 自动环境探测

Agent 启动时收集：

- 当前目录；
- git status；
- 当前分支；
- 项目语言；
- package manager；
- 测试命令；
- lint 命令；
- README / AGENTS / CLAUDE 文件；
- 最近相关文件。

### 9.3 编辑循环检测

如果 Agent 出现以下行为，应触发提醒：

- 反复修改同一文件；
- 多次运行同一失败测试；
- 多次出现相同错误；
- 连续多步没有获得新信息；
- 多次撤销又重新应用类似修改。

### 9.4 测试优先级策略

不同任务不必都跑全量测试，可以分层：

- 小修改：相关单测；
- 中等修改：包级测试；
- 核心逻辑：全量测试；
- infra 改动：集成测试 + smoke test；
- 安全相关：额外安全扫描。

### 9.5 工具权限分级

可以将工具分为不同风险等级。

低风险：

- read_file；
- search_files；
- git diff；
- 查看日志。

中风险：

- write_file；
- patch；
- run tests；
- 安装依赖。

高风险：

- 删除文件；
- git push；
- 部署；
- 数据库迁移；
- kubectl apply/delete；
- cloud API 修改资源。

高风险工具应支持：

- dry-run；
- sandbox；
- 人工确认；
- 回滚；
- 审计日志。

### 9.6 Trace 记录

每次 Agent 执行都应该记录：

- 用户任务；
- 模型版本；
- system prompt 版本；
- tool 版本；
- middleware 版本；
- 工具调用；
- 文件修改；
- 测试结果；
- 最终结果；
- 人类是否接受。

这些 trace 是后续优化 Agent 的核心资产。

---

## 10. 对平台工程和 KubeMin-Cli 类项目的启发

对于 Kubernetes / PaaS / 控制面类项目，Harness Engineering 可以和 Controller 模式建立类比。

Kubernetes Controller 的工作方式是：

```text
观察当前状态
→ 对比期望状态
→ 执行 reconcile
→ 失败重试
→ 保持幂等
→ 记录 event
→ 暴露 status
→ 最终收敛
```

Agent Harness 也可以类似理解：

```text
观察任务状态
→ 对比验收标准
→ 调用工具执行动作
→ 失败重试或换策略
→ 保持幂等
→ 记录 trace/event
→ 限制权限和并发
→ 暴露最终结果
```

如果把模型看作一个不稳定但能力很强的 reconciler，那么 Harness 就类似：

```text
controller-runtime
+ client-go
+ workqueue
+ rate limiter
+ event recorder
+ leader election
+ finalizer
+ RBAC
+ admission webhook
+ status condition
```

这个类比很重要。

真正的问题不是：

> 怎么让模型一次想对？

而是：

> 怎么构建一个控制面，使模型即使偶尔想错，也能被约束、纠偏、回滚、审计，并最终收敛到正确状态？

这与平台工程中常见的可靠性设计是一致的：

- CAS 状态迁移；
- 幂等 reconcile；
- 分布式锁/租约；
- retry backoff；
- cancellation signal；
- dry-run warnings；
- 失败清理；
- 共享资源保护；
- 结构化错误码；
- 状态条件与事件记录。

因此，Agent Harness 可以被视为 Agent 时代的控制面工程。

---

## 11. 对原文的评价

### 11.1 优点

#### 1. 把 Harness 解释得足够具体

文章没有停留在抽象概念，而是列出了 system prompt、tools、sandbox、compaction、hooks、sub-agent 等具体组件。

这让 Harness Engineering 从一个概念变成了可以执行的工程框架。

#### 2. 清晰区分 Prompt、Context、Harness

文章指出：

- Prompt Engineering 管一条指令；
- Context Engineering 管整个上下文窗口；
- Harness Engineering 管模型之外的整个运行环境。

这个区分有助于避免把所有 Agent 问题都归结为“prompt 没写好”。

#### 3. 有 benchmark 案例支撑

同一模型从 52.8 提升到 66.5 的例子很有说服力。

它说明 Harness 的影响可以被量化，不只是工程师的主观感受。

#### 4. 引入 The Bitter Lesson，避免过度工程化

文章没有鼓励无限堆复杂逻辑，而是提醒很多 Harness 设计可能只是短期补丁。

这使文章的判断更成熟：

- 需要 Harness；
- 但不要迷信复杂 Harness；
- Harness 应该模块化、可替换、可删除。

### 11.2 可以继续扩展的地方

#### 1. 生产级安全讨论还可以更深入

原文提到 sandbox，但没有系统展开：

- secret 管理；
- 权限分级；
- 审计日志；
- 多租户隔离；
- 数据泄漏；
- prompt injection；
- tool injection；
- supply chain 风险。

这些是企业级 Agent Harness 的核心问题。

#### 2. 评估体系可以进一步展开

Harness 怎么改才算更好？

需要一套 eval 方法：

- benchmark；
- replay；
- A/B test；
- trace diff；
- 成本指标；
- 延迟指标；
- 成功率；
- 人类接受率；
- regression task set。

#### 3. Harness 版本化值得单独讨论

生产系统中，Harness 本身也需要版本管理：

- prompt version；
- tool version；
- middleware version；
- model version；
- environment version；
- policy version。

否则执行 trace 无法复现。

#### 4. 多 Agent 协作风险可以继续展开

Sub-agent 很有用，但也引入风险：

- 上下文丢失；
- 目标漂移；
- 结果不可验证；
- 子代理幻觉污染主代理；
- 并发修改冲突；
- 成本失控。

这些都需要 Harness 设计解决。

---

## 12. 可继续研究的问题

后续可以围绕以下问题继续展开：

1. 如何设计 Agent Harness 的标准架构？
2. Coding Agent 的最小可用 Harness 应该包含哪些组件？
3. Harness 和 MCP、Tools、Function Calling 的关系是什么？
4. Agent Harness 如何做权限控制和安全隔离？
5. 如何设计 Agent trace schema？
6. 如何评估不同 Harness 版本的效果？
7. 如何把失败轨迹转化为 regression benchmark？
8. Harness 是否会成为未来 Agent 产品的核心壁垒？
9. Harness 数据如何反哺模型训练？
10. 在 Kubernetes / PaaS 场景下，Agent Harness 是否可以借鉴 Controller 模式？

---

## 13. 结论

这篇文章最值得记住的不是 Harness 这个新名词，而是一种 Agent 工程方法论。

当 Agent 犯错时，不要只问：

```text
是不是 prompt 没写好？
是不是模型不够强？
```

还要问：

```text
为什么系统允许它这样犯错？
这个错误能不能被工具、hook、sandbox、测试、权限、流程自动阻止？
这次失败能不能变成下一版 Harness 的改进？
```

最终目标不是让模型每次都自觉，而是通过系统设计让正确路径更容易，让错误路径更困难。

可以用一句话总结：

> Prompt Engineering 是教模型怎么回答；Context Engineering 是给模型足够的信息；Harness Engineering 是给模型一个能可靠工作的世界。

---

## 参考链接

- Harness Engineering — Agent 不好用，也许不是模型的问题：<https://yuanchaofa.com/post/harness-engineering-for-ai-agents>
- 文章源文件：<https://raw.githubusercontent.com/bbruceyuan/bbruceyuan.github.io/main/docs/post/LLM/harness-engineering-for-ai-agents.md>
- 作者文章列表：<https://yuanchaofa.com/post>
- OpenAI - Harness engineering: leveraging Codex in an agent-first world：<https://openai.com/index/harness-engineering/>
- LangChain - The Anatomy of an Agent Harness：<https://blog.langchain.com/the-anatomy-of-an-agent-harness/>
- LangChain - Improving Deep Agents with harness engineering：<https://blog.langchain.com/improving-deep-agents-with-harness-engineering/>
- HumanLayer - Skill Issue: Harness Engineering for Coding Agents：<https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents>
- Mitchell Hashimoto - My AI Adoption Journey：<https://mitchellh.com/writing/my-ai-adoption-journey>
