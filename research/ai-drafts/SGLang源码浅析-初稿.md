---
title: SGLang源码浅析
date: 2026-05-09 10:23:00
tags: [AI Infra, SGLang, LLM推理]
category: AI
---

### 前言

在理解 vLLM 时，最容易记住的是 PagedAttention。

它像操作系统里的分页内存管理一样，把原本连续、巨大、容易浪费的 KV Cache 拆成一个个固定大小的块，然后通过块表来管理。这个类比足够直观，所以很容易让人抓住 vLLM 的核心：它不是单纯让模型“跑起来”，而是在重新设计 LLM 推理过程中的显存管理和请求调度。

但 SGLang 给人的第一印象会更复杂一些。

因为它的名字里带有 Lang，很容易让人下意识以为它是一门语言。既然是一门语言，那么我们自然会想到词法分析、语法分析、AST、类型系统、解释器、编译器、运行时、垃圾回收……如果按照这套传统编译器的视角去读 SGLang，很快就会产生一种错位感：源码里并不存在一个独立的 parser，也没有传统意义上的静态类型系统，更没有一个像 JVM 那样的虚拟机。

这并不是因为 SGLang 做得“不完整”，而是因为它从一开始就不是这个问题。

SGLang 更准确的定位是：嵌入在 Python 中的前端 DSL，加上一个面向大模型推理的高性能运行时 SRT。

前端负责表达一个 LLM 程序：提示词如何组织，哪里需要生成，哪里需要选择，哪里有并行分支，哪里要做结构化输出。后端负责真正困难的部分：分词，请求调度，KV Cache 复用，连续批处理，模型执行，并行通信，约束解码，最后把结果返回给调用者。

所以，如果 vLLM 更像是围绕 PagedAttention 和 Continuous Batching 构建的推理运行时，那么 SGLang 更像是试图把“提示词工程”提升成“可编程推理系统”。

这也是本文想讨论的重点：SGLang 到底解决了什么问题？它为什么不是传统意义上的编程语言？它的源码应该从哪里读？以及它背后的核心机制——前端 DSL、TokenizerManager、Scheduler、RadixAttention、ModelRunner 和约束生成——是如何串起来的。



### SGLang 不是传统编程语言

理解 SGLang 的第一步，是先放弃一个误解：它不是一门脱离 Python 独立存在的新语言。

传统编程语言通常会有一套自己的文本语法。源码先经过词法分析变成 token，再经过语法分析变成 AST，然后进行语义分析、类型检查、中间表示生成，最后解释执行或编译成某种目标代码。

但 SGLang 不是这样工作的。

你写的 SGLang 程序本质上还是 Python 代码。它通过 `@sgl.function` 装饰器、`s += ...` 的状态追加、`sgl.gen()`、`sgl.select()`、`fork()` 等 API，把一个 LLM 应用中的生成过程表达出来。

也就是说，Python 解释器已经帮它完成了最基础的词法和语法解析。SGLang 自己并不需要再解析一份独立的 `.sgl` 源文件。它真正做的是：把这些 Python API 调用转换成一组内部表达式和状态操作。

从源码结构上看，前端最值得看的文件是：

```text
python/sglang/lang/ir.py
python/sglang/lang/interpreter.py
python/sglang/lang/tracer.py
python/sglang/lang/backend/
```

其中 `ir.py` 里定义了 SGLang 前端的核心表达式，例如 `SglFunction`、`SglGen`、`SglSelect`、`SglFork`、`SglVariable` 等。它们不太像传统 AST 节点，更像是“提示词程序”中的语义操作。

`SglGen` 表示这里需要模型生成一段内容；`SglSelect` 表示这里需要在几个候选项中选择；`SglFork` 表示从当前状态分叉出多个并行分支；`SglVariable` 表示引用之前生成出来的变量。

如果非要给它写一个概念语法，大概可以理解成：

```python
@sgl.function
def program(s, input):
    s += sgl.system("...")
    s += sgl.user(input)
    s += sgl.assistant(sgl.gen("answer"))
```

这里真正关键的不是 Python 的 `def` 和缩进，而是这些 API 调用背后的 IR 语义。

所以，SGLang 的“语言”并不体现在独立语法上，而体现在它提供了一套适合描述 LLM 推理工作流的抽象。

这也解释了为什么 SGLang 更像是一种 EDSL，也就是嵌入式领域特定语言。它借用了 Python 的语法、生态和调试工具，但把特定领域的问题——大模型生成、分支、约束、状态管理——抽象成一组专门的运行时操作。



### 为什么需要把提示词变成程序？

早期使用大模型时，很多人会把 prompt 当成一段字符串。

比如：

```text
你是一个助手，请回答下面的问题：xxx
```

如果问题稍微复杂一点，就继续拼接更多字符串；如果要多轮对话，就把历史消息拼进去；如果要让模型输出 JSON，就在 prompt 里写“请严格按照 JSON 格式返回”。

这当然能工作，但它很快会遇到问题。

第一个问题是可维护性。字符串拼接越多，prompt 就越像一团难以维护的胶水。哪一段是系统提示？哪一段是用户输入？哪一段是模型上次输出？哪里应该停止生成？哪里应该保存变量？这些语义都藏在字符串里。

第二个问题是执行效率。对后端运行时来说，如果它只看到一段扁平字符串，就很难知道哪些部分是共享前缀，哪些部分可以复用 KV Cache，哪些生成可以并行，哪些输出必须满足结构约束。

第三个问题是控制能力。复杂 LLM 应用往往不是一次问答，而是多步推理、并行采样、RAG、ReAct、Tree-of-Thought、结构化抽取、工具调用等组合。如果仍然只靠字符串和 for 循环手写，很容易把程序逻辑和提示词格式搅在一起。

SGLang 的前端 DSL 试图解决这个问题。

它让开发者用更像写程序的方式组织 LLM 推理过程：

```python
@sgl.function
def qa(s, question):
    s += sgl.user(question)
    s += sgl.assistant(sgl.gen("answer", max_tokens=256))
```

这里的 `answer` 不是字符串中一个模糊的位置，而是一个明确的生成变量。运行时可以知道这里发生了一次生成，并且可以把结果绑定到状态里。

如果需要并行生成多个候选答案，可以 fork：

```python
forks = s.fork(3)
forks += sgl.assistant(sgl.gen("answer"))
forks.join()
```

这比手动复制三份 prompt 更明确，也让运行时有机会把并行结构暴露给后端调度。

从这个角度看，SGLang 的前端语言不是为了让开发者少写几行代码，而是为了把 LLM 应用中的结构显式化。

结构一旦显式化，后端才有优化空间。

这和传统软件工程中的抽象很相似。一个系统如果只有字符串和隐式约定，运行时就只能看到一坨文本；一个系统如果有函数、变量、分支、状态和约束，运行时才能理解它可以怎么执行、怎么复用、怎么调度。



### SRT：真正沉重的部分在运行时

虽然 SGLang 名字里有 Lang，但今天的 SGLang 已经明显偏向推理基础设施。

如果只想理解它为什么快，或者它和 vLLM、TensorRT-LLM 这类系统有什么关系，就不能把注意力停留在前端 DSL 上，而要顺着后端运行时 SRT 往下看。

SGLang 后端主链路大致可以概括成：

```text
TokenizerManager -> Scheduler -> TpModelWorker -> ModelRunner
```

一个请求进入系统后，首先到达 `TokenizerManager`。它负责把文本输入变成 token，处理不同输入形态，并把请求规范化。接着请求进入 `Scheduler`，由调度器决定当前哪些请求可以组成 batch，哪些请求处于 prefill 阶段，哪些处于 decode 阶段，哪些可以利用已有前缀缓存。

调度完成后，请求会被交给 `TpModelWorker`。这里的 TP 指 Tensor Parallel，也就是张量并行。`TpModelWorker` 会把调度器组织好的 batch 转换为模型执行需要的数据结构，最终进入 `ModelRunner` 做真正的 forward。

如果把这条链路类比成一条流水线：

- `TokenizerManager` 像入口检查和标准化层。
- `Scheduler` 像生产调度中心。
- `TpModelWorker` 像负责把任务分配到具体机器的车间主管。
- `ModelRunner` 则是真正执行计算的机器。

在这条链路里，最值得关注的是 `Scheduler`。

因为 LLM 推理不是传统 Web 请求那样“进来一个请求，处理完返回”。每个请求都包含两个完全不同的阶段：prefill 和 decode。

prefill 阶段要把整段 prompt 输入模型，计算量大，像一次密集的矩阵计算；decode 阶段则是一个 token 一个 token 地生成，单步计算量较小，但需要频繁读取 KV Cache。不同请求的 prompt 长度不同，生成长度不同，到达时间不同，缓存命中情况也不同。

如果用传统静态批处理，必须等一批请求全部结束后再处理下一批，很容易造成 GPU 空转。短请求要等长请求，新请求要等当前 batch 结束，GPU 利用率自然上不去。

所以现代 LLM serving 系统都会走向连续批处理。batch 不再是一个固定集合，而是一个动态变化的执行队列。每一轮调度都要重新决定：哪些请求继续 decode，哪些新请求可以插入 prefill，当前 token budget 还够不够，显存是否足够，是否需要淘汰缓存。

这就是 SRT 作为运行时的意义。

它不是简单地调用一次 `model.generate()`，而是在不断组织请求、缓存、显存和 GPU 时间片。



### RadixAttention：SGLang 的缓存视角

如果说 vLLM 的核心类比是分页内存，那么 SGLang 的核心类比更像是前缀树。

SGLang 的 RadixAttention 使用 radix tree 来管理 token 前缀与 KV Cache 之间的映射。它关注的不是单个请求如何分配显存块，而是多个请求之间是否共享前缀。

这在 LLM 应用中非常常见。

例如一个系统提示可能在所有请求中都相同；few-shot 示例可能在一批请求中重复；多轮对话中，后续请求天然包含前面的上下文；RAG 场景里，不同问题可能共享相同文档片段；Agent 工作流中，不同分支可能从同一个中间状态 fork 出来。

如果每次都从头计算这些前缀，显然会浪费大量计算。RadixAttention 的思路就是把已经计算过的前缀缓存下来，当新的请求到来时，先在 radix tree 中做前缀匹配，如果发现前面一段 token 已经存在，就直接复用对应的 KV Cache。

这和操作系统中的缓存思想很像：如果同一批数据已经被读取并保存在更快的存储层里，就没有必要每次都重新从慢路径加载。

只是对于 LLM 来说，这个“数据”不是普通文件，而是 token 前缀对应的 KV Cache。

这也解释了一个源码中的边界：如果请求直接传入 `input_embeds`，SGLang 会要求关闭 radix cache。

因为 RadixAttention 的前提是 token 前缀可以比较。两个请求的 token 序列是否相同，可以用来判断它们是否共享前缀。但如果输入已经变成了 embedding，那么“前缀相等”这件事就不再天然成立。此时继续使用 token 级前缀缓存就会破坏内部假设。

这个细节很小，但非常能说明系统设计中的隐含前提：缓存机制不是魔法，它依赖清晰的数据表示。

在 SGLang 中，RadixAttention 通常还会和 paged KV cache、连续批处理、chunked prefill、overlap schedule 组合使用。也就是说，它不是孤立优化，而是和调度器共同工作。

调度器要知道哪些请求命中了前缀缓存，哪些请求需要新算；缓存系统要管理哪些 KV 可以复用，哪些应该淘汰；模型执行层则要在正确的位置读取这些 KV。

这也体现出推理系统的复杂性：真正决定性能的往往不是某个单独技巧，而是多个机制之间能否形成闭环。



### 零开销调度：CPU 和 GPU 不该互相等待

在大模型推理中，GPU 是最昂贵的资源。

如果 GPU 正在等待 CPU 准备 batch metadata、等待调度器整理请求、等待缓存系统查找前缀，那么系统吞吐就会下降。反过来，如果 CPU 能在 GPU 执行当前 batch 时，提前准备下一个 batch，那么 GPU 空转时间就会减少。

SGLang v0.4 中强调的 zero-overhead batch scheduler，本质上就是这个方向：让 CPU 调度工作和 GPU 执行尽可能重叠。

这件事听起来简单，但在真实系统里并不容易。

因为调度器需要处理的不是一个简单队列，而是一组持续变化的请求状态。它要考虑：

- 当前有哪些 running requests？
- 哪些请求已经完成？
- 哪些新请求可以加入？
- decode 是否应该优先？
- prefill 是否需要切块？
- 当前显存还能容纳多少 KV？
- radix cache 是否命中？
- 是否启用了 speculative decoding？
- 是否有 grammar 约束？
- 是否处于 TP / DP / EP / PD 等并行模式？

这些判断都要在极短时间内完成，否则调度本身就会变成瓶颈。

这让我想到 Kubernetes 中调度器的角色。Kubernetes 调度器也不是简单地把 Pod 随便放到一个节点上，它要考虑资源、亲和性、污点、拓扑、优先级、抢占等因素。只不过在 LLM serving 中，调度粒度变成了请求、token、KV cache 和 GPU batch。

从这个角度看，SGLang 的 Scheduler 并不是一个普通队列，而是一个面向 GPU 推理的资源调度器。

它要做的不是“公平地处理请求”，而是在延迟、吞吐、显存、缓存命中率和硬件利用率之间不断做权衡。



### 结构化输出：让生成空间变小

LLM 的输出是开放的，这是它强大的地方，也是它麻烦的地方。

如果我们只是问一个开放问题，模型可以自由生成自然语言。但在很多工程场景中，我们并不希望模型自由发挥，而是希望它输出一个可解析、可验证、可落库的结构。

例如：

```json
{
  "name": "...",
  "age": 18,
  "tags": ["..."]
}
```

如果只在 prompt 里写“请严格输出 JSON”，模型仍然可能多输出解释文本、漏掉字段、类型不匹配，或者在字符串里塞入不符合预期的内容。

SGLang 支持结构化输出，背后可以接 XGrammar、Outlines、Llguidance 等 grammar backend。它可以通过 JSON Schema、正则或 EBNF 来约束生成空间。

这件事的意义在于：不要只靠模型“自觉遵守格式”，而是把格式约束下沉到解码过程里。

从工程角度看，这和类型系统、schema 校验、API 合约都有相似之处。区别在于，传统程序是在输出之后校验，而约束解码希望在 token 生成过程中就把非法路径剪掉。

这可以减少很多后处理成本。

当然，它也不是免费的。约束越复杂，解码过程需要维护的状态就越多；不同 grammar backend 的性能和能力也不一样。SGLang 默认选择 XGrammar，也说明结构化输出已经不是一个外围功能，而是 LLM 应用运行时的重要组成部分。

尤其在 Agent、Function Calling、RAG 抽取、自动化运维这类场景中，结构化输出几乎是必要条件。因为下游系统无法长期依赖“看起来像 JSON”的自然语言文本。



### 和 vLLM 的差异在哪里？

vLLM 和 SGLang 经常被放在一起比较，因为它们都属于 LLM 推理系统，也都关注吞吐、延迟、KV Cache 和调度。

但两者的出发点并不完全一样。

vLLM 的核心故事更清晰：PagedAttention + Continuous Batching。它把 LLM 推理中的 KV Cache 管理抽象成类似操作系统分页的机制，从而提升显存利用率，并通过连续批处理提高 GPU 利用率。

SGLang 的故事则更偏“前后端协同”。

它不仅仅关心一个请求如何被高效执行，还关心复杂 LLM 程序如何表达，表达出来的结构又如何被后端利用。

可以粗略理解为：

- vLLM 更像一个通用高性能 LLM serving engine。
- SGLang 更像一个面向复杂 LLM 程序的 serving framework。

当然，这种区分不是绝对的。vLLM 也在不断扩展生态和上层能力，SGLang 也在不断强化底层 runtime 和 kernel。两者的边界会随着项目演进而变化。

但从源码阅读角度看，这个区分很有帮助。

读 vLLM 时，可以优先抓 PagedAttention、KV Cache、Scheduler、Block Manager、Worker 这些主线；读 SGLang 时，则应该同时抓两条线：

第一条是前端 DSL：`lang/ir.py`、`interpreter.py`、`tracer.py`。

第二条是后端 SRT：`server_args.py`、`tokenizer_manager.py`、`scheduler.py`、`tp_worker.py`、`model_executor/`、`mem_cache/`、`constrained/`。

如果只读前端，会误以为 SGLang 只是一个 prompt 编排库；如果只读后端，又会忽略它为什么需要前端 DSL 暴露程序结构。

SGLang 的关键就在这两者之间。



### 源码阅读顺序

如果是第一次阅读 SGLang 源码，我不建议从目录树顺序开始读。

因为这种基础设施项目的代码通常会有大量 feature flag、硬件后端、兼容逻辑、历史路径和性能优化。如果从入口文件一路追 import，很容易迷失在细节里。

更好的方式是先建立主干，再看分支。

我会建议按下面顺序阅读：

第一步，读 `server_args.py`。

这个文件虽然很大，但它是理解系统能力边界的入口。模型路径、tokenizer、量化、显存比例、调度策略、attention backend、grammar backend、speculative decoding、数据并行、LoRA、HiCache、debug 选项，几乎都要在这里汇聚和校验。

它不像一个普通配置文件，更像 SGLang 运行时的“启动语义”。很多 feature 能不能一起开，默认值如何推导，硬件后端如何选择，都可以在这里找到线索。

第二步，读前端的 `ir.py` 和 `interpreter.py`。

这一步是为了理解 SGLang 的“Lang”到底是什么。重点不是语法，而是 `gen`、`select`、`fork`、role、variable 这些操作如何被表示，又如何交给 `StreamExecutor` 执行。

第三步，顺着请求链路读 `TokenizerManager -> Scheduler -> TpModelWorker -> ModelRunner`。

这一步才是 SRT 的主干。重点关注请求从文本变成 token，从 token 变成 batch，从 batch 变成 forward，再从 forward 变成输出的过程。

第四步，读 `mem_cache/*`。

这里是 RadixAttention、KV cache、内存池、前缀复用、淘汰策略的核心区域。只读调度器而不读缓存，很难理解 SGLang 的性能来源。

第五步，读 `constrained/*` 和 grammar backend。

这部分对应结构化输出。它关系到 JSON Schema、regex、EBNF 等约束如何进入生成过程。

第六步，再看 `sgl-kernel/*`。

这是更底层的性能临界路径，包含 C++/CUDA/Torch extension 等内容。除非目标是做性能优化或 kernel 调试，否则可以放在第二轮深入阅读。

这个顺序的好处是，先建立“系统怎么运转”的骨架，再逐步进入性能细节。



### 可以动手验证的几个实验

只读源码很容易停留在概念层。对于推理系统来说，最好一边读一边跑。

最小实验可以从 offline engine 开始：

```python
import sglang as sgl

llm = sgl.Engine(model_path="qwen/qwen2.5-0.5b-instruct")
outputs = llm.generate(["Hello"], {"temperature": 0})
print(outputs[0]["text"])
```

这个路径可以绕开 HTTP 服务，直接验证 runtime 主链路。

然后再启动服务：

```bash
python -m sglang.launch_server \
  --model-path qwen/qwen2.5-0.5b-instruct \
  --host 0.0.0.0 \
  --log-level warning
```

如果想观察调度和缓存，可以做几组对照：

1. 默认启动和 `--disable-radix-cache` 对比，发送共享前缀请求，观察 TTFT 和 cached tokens。
2. 默认启动和 `--disable-overlap-schedule` 对比，观察 CPU/GPU 是否存在调度空洞。
3. 普通输出和 JSON Schema 结构化输出对比，观察约束解码带来的开销和稳定性。
4. 开启 speculative decoding，对比 decode 阶段吞吐。
5. 调整 `--chunked-prefill-size`，观察长 prompt 下 prefill 对延迟和显存的影响。

这些实验的目的不是追求某个漂亮数字，而是把源码中的机制和运行时现象对应起来。

例如，RadixAttention 不应该只停留在“使用 radix tree 复用 KV Cache”这句话上，而应该能看到共享前缀请求下 TTFT 的变化；overlap schedule 也不应该只停留在“CPU/GPU 重叠”这句话上，而应该能在 profiler 时间线里看到 GPU 空洞是否减少。

系统类源码阅读最怕的是只读静态结构，不看动态行为。

SGLang 这种推理运行时尤其如此。很多代码的意义只有放在吞吐、延迟、显存和缓存命中率这些指标里才会变得清晰。



### SGLang 的复杂性在哪里？

SGLang 的复杂性不在于它写了一套多么复杂的语言语法。

恰恰相反，前端 DSL 的形式并不难理解。真正复杂的是：它要把一个高层 LLM 程序映射到一个动态、高并发、高显存压力、强硬件相关的推理系统中。

这其中至少有几类复杂性。

第一类是状态复杂性。

LLM 请求不是无状态函数调用。每个请求都有 prompt、已生成 token、KV Cache、采样参数、约束状态、streaming 状态、是否完成、是否命中缓存等信息。多轮对话和 fork 会进一步增加状态分支。

第二类是调度复杂性。

不同请求处在不同阶段，prefill 和 decode 的计算特征完全不同。调度器既要尽可能提高吞吐，又要控制延迟，还要考虑显存上限和缓存复用。

第三类是内存复杂性。

KV Cache 很大，而且生命周期不一。哪些缓存应该保留，哪些应该淘汰，哪些可以复用，哪些必须新分配，这些决策都会影响性能和稳定性。

第四类是硬件复杂性。

GPU、CPU、NPU、TPU，NVIDIA、AMD，不同 attention backend，不同 kernel，不同并行策略，都会带来大量条件分支。推理系统不是纯算法，它最终要落在硬件上。

第五类是工程复杂性。

OpenAI 兼容 API、原生 API、offline engine、LoRA、量化、结构化输出、多模态、分布式部署、安全边界、监控、profiling……这些能力叠加起来，SGLang 就不再只是一个学术原型，而是一套完整基础设施。

也正因为如此，阅读 SGLang 源码时不能只问“这个函数做什么”，还要问：它处在什么系统压力下？它试图优化哪个指标？它牺牲了什么？它依赖什么前提？

这比单纯读一个算法实现要难，但也更接近真实工程。



### 小结

SGLang 最有意思的地方在于，它把两个原本容易分离的问题放到了一起：如何表达复杂 LLM 程序，以及如何高效执行这些程序。

前端 DSL 让开发者可以用 Python 组织生成、选择、分支、角色和结构化输出；后端 SRT 则通过 TokenizerManager、Scheduler、TpModelWorker、ModelRunner、RadixAttention、连续批处理和约束解码，把这些程序尽可能高效地跑在硬件上。

如果把它当成传统编译器，会发现它缺少独立 parser、静态类型系统和虚拟机；但如果把它当成可编程推理系统，就会发现这些并不是重点。

它真正关心的是：

- 如何让 LLM 应用不再只是字符串拼接；
- 如何把生成过程中的结构暴露给运行时；
- 如何复用共享前缀的 KV Cache；
- 如何在动态请求中持续组织 batch；
- 如何让 CPU 调度和 GPU 执行尽量重叠；
- 如何在开放生成中加入结构化约束；
- 如何把复杂 LLM workload 变成一个可服务、可调度、可优化的系统。

从这个意义上看，SGLang 并不是“另一种调用大模型的 Python 库”，而是一种面向 LLM 时代的运行时抽象。

它提醒我们，大模型应用的复杂性正在从“如何写 prompt”转向“如何组织推理程序、如何调度请求、如何管理 KV Cache、如何压榨 GPU、如何保证输出结构”。

这和云原生时代的变化很像。最开始人们只是想把程序跑起来，后来才发现真正困难的是部署、调度、扩缩容、观测、故障恢复和资源隔离。LLM 应用也是如此。最开始只要能调用模型就足够了，后来才发现真正困难的是低延迟、高吞吐、低成本、强约束、多租户和稳定运行。

SGLang 之所以值得读，不只是因为它快，而是因为它把这些问题集中在了一套源码里。

它既能看到 Python DSL 的表达力，也能看到 GPU 推理系统的工程压力；既能看到前缀树这种数据结构，也能看到连续批处理这种调度思想；既能看到结构化输出的产品需求，也能看到底层 kernel 和硬件后端的现实约束。

对于想理解 AI Infra 的开发者来说，这比单纯学会一个启动命令更重要。

因为真正的变化不在命令行参数里，而在系统边界的重新划分里。

SGLang 试图回答的问题不是“怎样调用一个模型”，而是“当大模型调用变成复杂程序以后，运行时应该长什么样”。

这也是它最值得研究的地方。
