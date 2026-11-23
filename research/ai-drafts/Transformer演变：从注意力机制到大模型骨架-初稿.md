---
title: Transformer演变：从注意力机制到大模型骨架
date: 2026-05-09 10:45:00
tags: [LLM,Transformer,AI Infra,学习笔记]
category: LLM
---

### 前言

2017 年，Google 在论文《Attention Is All You Need》中提出 Transformer。这个标题后来几乎变成了一个预言：在相当长的一段时间里，自注意力机制真的成为了自然语言处理，乃至多模态模型中最重要的基础结构。

但如果只停留在“Transformer = Attention”这个理解上，今天再去看 LLaMA、Mistral、Qwen、Gemma、DeepSeek、Mixtral、Jamba 这些模型时，很快就会发现一个问题：它们都说自己是 Transformer，但内部结构已经和 2017 年的原始 Transformer 有了很多差异。

有的模型把 LayerNorm 换成 RMSNorm；有的把普通前馈网络换成 SwiGLU；有的不用绝对位置编码，而是使用 RoPE 或 ALiBi；有的把多头注意力里的 K/V 共享起来，变成 MQA 或 GQA；有的进一步把 KV cache 压缩成 latent 表示，比如 DeepSeek-V2 的 MLA；还有的模型干脆不再让每个 token 经过同一个前馈网络，而是引入 MoE 专家路由。

这就像一辆汽车。最早我们讨论的是“发动机为什么能让车跑起来”，后来真正进入工业化阶段，工程师关心的问题就变成了：油耗怎么降？散热怎么做？变速箱如何匹配？底盘如何兼顾舒适和操控？车身如何在安全性和轻量化之间取舍？

Transformer 的演变也是如此。

最初的问题是：如何让模型更好地处理序列关系？

后来问题变成：如何在更长上下文、更大参数、更低延迟、更少显存之间做权衡？

本文会沿着这个思路，整理 2021–2025 年开源大模型中 Transformer Block 的主要演变路径。它不是一篇论文综述，也不会试图穷尽所有模型细节，而是希望回答几个更工程化的问题：

1. 今天常见的 LLaMA-like Transformer 到底和原始 Transformer 有什么不同？
2. RMSNorm、RoPE、SwiGLU、GQA、MoE、MLA 这些修改分别解决什么问题？
3. 为什么这些修改最终会汇合成今天开源大模型的主流设计？
4. Transformer 的演变本质上是在优化什么？

需要说明的是，本文以公开论文、模型卡和技术报告为依据，对原始笔记中的信息做了反向核验。部分模型的具体 block 细节并没有完全公开，因此会尽量避免把推断写成确定事实。



### 从原始 Transformer 说起

原始 Transformer 是一个 encoder-decoder 架构。它最初被设计用来做机器翻译：编码器读取源语言句子，解码器生成目标语言句子。这个结构里包含几个关键组件：

1. 多头自注意力机制
2. 前馈神经网络
3. 残差连接
4. LayerNorm
5. 位置编码
6. encoder-decoder attention

但今天讨论大语言模型时，我们更多是在讨论 decoder-only Transformer，也就是 GPT、LLaMA、Qwen 这一类模型。它们保留了解码器侧的自回归生成方式：模型根据已有 token 预测下一个 token，然后把新 token 追加到输入末尾，再继续预测下一个。

如果把一个 decoder-only Transformer Block 抽象出来，它大致是这样：

```text
输入 token embedding
      │
      ▼
自注意力层：让当前 token 从上下文中提取信息
      │
      ▼
前馈神经网络：对每个 token 的表示做非线性变换
      │
      ▼
输出 hidden states
```

更通俗一点说：

- 注意力层负责“看上下文”
- 前馈层负责“加工信息”
- 归一化负责“让数值不要失控”
- 位置编码负责“告诉模型顺序”
- 残差连接负责“让深层网络还能训练”

早期 Transformer 的很多设计并不是错的，而是在后来大模型规模变大、上下文变长、推理服务变复杂以后，暴露出了新的瓶颈。

例如：

- LayerNorm 在大规模训练中是否足够简洁稳定？
- FFN 的表达能力是否还能提升？
- 绝对位置编码能否支持更长上下文？
- 标准多头注意力的 KV cache 是否太占显存？
- 全局注意力在 128K、1M 上下文下是否还能承受？
- 所有 token 都经过同一套 FFN，参数扩展是否太昂贵？

所以后来的演变并不是“推翻 Transformer”，而是在保留其基本骨架的同时，不断替换其中最昂贵、最脆弱、最不适合长上下文的部件。



### 第一条主线：归一化从 LayerNorm 走向 RMSNorm

在深度神经网络里，归一化层的作用很像高速公路上的限速和车道规范。没有它，数值可能在层与层之间不断放大或缩小，最后训练变得不稳定。

原始 Transformer 使用 LayerNorm。LayerNorm 会对一组 hidden states 做均值和方差归一化，大致可以理解为：先减去均值，再除以标准差，让数值分布更稳定。

后来很多大模型开始使用 RMSNorm。RMSNorm 的论文是《Root Mean Square Layer Normalization》，它去掉了均值中心化，只保留均方根缩放。简单说，它不再关心“这组数的平均值是多少”，而是更关心“这组数整体的尺度有多大”。

这看起来只是一个很小的改动，但在大模型里，小改动往往会因为层数和参数规模被放大。

RMSNorm 的优势主要有三个：

1. 计算更简单
2. 训练更稳定
3. 和 pre-norm 结构配合良好

所谓 pre-norm，是指在进入注意力层或 FFN 之前先做归一化，而不是在子层输出之后再做。LLaMA 采用的就是 pre-norm with RMSNorm。这个组合后来几乎成为开源大模型的默认选项之一。

LLaMA 论文中明确提到，它参考 GPT-3 的 pre-normalization 思路，但将 LayerNorm 替换成 RMSNorm。后续 Llama 2、Llama 3、Qwen、Gemma、DeepSeek、OLMo 2 等模型，也都沿着类似方向演进。

这说明一个问题：大模型架构的进步不一定总是来自某个看起来很炫的新模块，有时候来自于“让基础部件更稳定、更简单、更适合规模化”。

归一化层就是这样的部件。它不负责让模型突然变聪明，但它决定了模型能否在更大规模上稳定地学习。



### 第二条主线：前馈网络从 GELU 走向 SwiGLU

在 Transformer Block 中，注意力层往往最受关注，因为它负责 token 之间的信息交互。但如果只看参数量，前馈网络通常占据了很大比例。

注意力层像是在开会：每个 token 去询问其他 token，看看谁和自己有关。

前馈网络则像是每个 token 回到自己的工位上，把刚刚收集到的信息进一步加工。它不直接和其他 token 交互，但它承载了大量模型知识和非线性变换能力。

原始 Transformer 使用的是 ReLU FFN，后来 GPT 系模型常见的是 GELU。再后来，PaLM、LLaMA 等模型开始使用 SwiGLU 一类门控前馈网络。

GLU 的核心思想可以用一个生活化类比理解：普通 FFN 像是一条固定流水线，输入什么都要经过同样的加工流程；门控 FFN 则像是在流水线前加了一个“选择开关”，它会决定哪些信息应该被放大，哪些信息应该被抑制。

SwiGLU 可以粗略理解为：

```text
一条分支负责生成候选信息
另一条分支负责生成门控权重
两者相乘后再输出
```

这种结构让 FFN 拥有更强的表达能力。Noam Shazeer 在《GLU Variants Improve Transformer》中系统比较了多种 GLU 变体，后来 SwiGLU 成为许多大模型中的常见选择。

LLaMA 论文中也明确写到，它使用 SwiGLU 替代 ReLU，并且为了控制参数量，将隐藏维度设置为约 `2/3 * 4d`，而不是简单使用更大的 FFN。

这背后体现的是一个典型工程权衡：

- FFN 太小，模型容量不足
- FFN 太大，参数和计算成本上升
- 门控结构可以在相近成本下提升表达能力

所以 SwiGLU 并不是一个孤立 trick，它和模型规模、训练稳定性、kernel 实现、hidden size 对齐都有关系。很多时候，一个结构是否成为主流，不只看论文中的效果，也取决于它能否被高效实现。



### 第三条主线：位置编码从绝对位置走向 RoPE 和 ALiBi

如果输入一句话：

```text
我 喜欢 你
```

和：

```text
你 喜欢 我
```

两句话包含的 token 一样，但含义完全不同。原因在于顺序变了。

对 Transformer 来说，自注意力本身并不知道 token 的顺序。它看到的是一组向量，如果没有额外的位置信息，模型并不能天然知道谁在前，谁在后。因此，位置编码是 Transformer 里非常关键的一环。

原始 Transformer 使用正弦/余弦位置编码，也有一些早期 GPT 模型使用 learned absolute position embedding。绝对位置编码可以理解为：给第 1 个位置、第 2 个位置、第 3 个位置分别贴上标签。

但语言中很多模式并不只依赖绝对位置，而更依赖相对距离。

例如：

- 主语和谓语之间隔了几个词？
- 当前 token 应该关注前面多远的 token？
- 一段代码里的括号、缩进、变量引用相隔多远？

这使得相对位置建模变得重要。

RoPE，也就是 Rotary Position Embedding，来自 RoFormer。它的做法不是把位置向量直接加到 token embedding 上，而是在计算注意力之前，对 Query 和 Key 按位置做旋转。这样 Q 和 K 做点积时，注意力分数中自然会包含相对位置信息。

可以把它想象成：每个 token 进入注意力计算前，都会根据自己所在的位置调整“朝向”。两个 token 是否相关，不仅取决于它们的语义向量，也取决于它们在序列中的相对距离。

RoPE 后来成为 LLaMA-like 模型的主流选择。GPT-J、GPT-NeoX 较早在开源 GPT 模型中使用 RoPE；LLaMA 则把 RoPE、RMSNorm、SwiGLU 组合成了一套影响深远的现代 decoder-only Transformer 骨架。Qwen、Gemma、DeepSeek、OLMo 2 等模型也大量采用 RoPE 或 RoPE 的扩展版本。

与 RoPE 并行的另一条路线是 ALiBi。ALiBi 的论文《Train Short, Test Long》提出直接在 attention score 上加入一个和距离相关的线性 bias。距离越远，惩罚越大。它不需要显式的位置 embedding，因此天然具有一定长度外推能力。

BLOOM、MPT、Baichuan-13B 等模型采用过 ALiBi 路线。BigScience 在 BLOOM 相关工作中也对 learned position、rotary、ALiBi 等方案做过比较。

从今天回看，RoPE 成为更主流的选择，并不意味着 ALiBi 没有价值。更合理的理解是：

- ALiBi 简洁，外推友好
- RoPE 与 LLaMA 生态、现代 attention kernel、长上下文扩展方案结合更紧密

这也是技术演进中常见的现象：一个方案的流行程度，不只取决于它单点是否优雅，还取决于它能否融入整个生态。



### 第四条主线：注意力从 MHA 走向 MQA、GQA 和 MLA

如果说 RoPE 解决的是“位置信息怎么进入注意力”，那么 MQA、GQA、MLA 解决的是另一个更工程化的问题：KV cache 太贵了。

在自回归生成中，模型一次生成一个 token。生成第 1000 个 token 时，前面 999 个 token 的 Key 和 Value 其实已经计算过了。如果每次都重新计算一遍，代价太高。因此推理系统会把历史 token 的 K/V 缓存下来，这就是 KV cache。

KV cache 的价值很大，但它也带来一个问题：上下文越长，层数越多，注意力头越多，缓存就越大。

这也是为什么长上下文模型不只是算法问题，更是系统问题。128K、1M 上下文听起来很诱人，但如果 KV cache 无法承受，模型就很难真正服务化。

标准多头注意力 MHA 中，每个 Query head 都有自己对应的 Key 和 Value。假设有 32 个 attention heads，就会有 32 组 K/V。这样表达能力强，但缓存成本也高。

MQA，也就是 Multi-Query Attention，做法很激进：多个 Query heads 共享同一组 K/V。这样 KV cache 大幅下降。Noam Shazeer 在《Fast Transformer Decoding: One Write-Head is All You Need》中提出了这个方向。

MQA 的直觉很简单：

```text
以前是每个读者都带一套资料库
现在是很多读者共享同一套资料库
```

共享资料库当然省空间，但也可能损失一些表达能力。尤其在更大模型上，所有 heads 共享一组 K/V 可能太过粗糙。

于是 GQA 出现了。GQA，即 Grouped-Query Attention，可以看作 MHA 和 MQA 之间的折中：不是所有 Query heads 共享一组 K/V，而是把 Query heads 分成几组，每组共享一组 K/V。

```text
MHA：每个 head 一套 K/V
MQA：所有 heads 共用一套 K/V
GQA：每组 heads 共用一套 K/V
```

这就是为什么 GQA 后来成为主流：它在能力和推理成本之间找到了更稳妥的平衡。

Mistral 7B 的论文明确写到，它使用 GQA 来加速推理并降低解码内存，同时配合 Sliding Window Attention 支持更长序列。Llama 2 的 70B 模型使用 GQA，Llama 3 进一步把 GQA 扩展到主模型尺寸。Qwen2.5 模型卡也明确写出：7B 模型使用 28 个 Q heads 和 4 个 KV heads。Gemma 2 的技术报告中还给出了一个很有意思的消融：在 9B 规模上，GQA 的平均分略高于 MHA，同时参数更少、推理更快，因此作者选择 GQA。

这说明 GQA 不只是“为了省显存牺牲能力”，它在某些规模和配置下可能是更好的综合选择。

DeepSeek-V2 的 MLA 则把这条路线推得更远。MLA，全称 Multi-head Latent Attention，它不是简单减少 K/V 组数，而是把 K/V 联合压缩到一个低秩 latent 向量中，推理时缓存这个 latent 表示，而不是缓存完整 K/V。

这相当于把“缓存什么”本身变成了一个架构设计问题。

DeepSeek-V2 技术报告称，相比 DeepSeek 67B，它在性能更强的同时，训练成本降低 42.5%，KV cache 降低 93.3%，最大生成吞吐提升 5.76 倍。这个数字不能简单归因于 MLA 单点，因为模型还引入了 DeepSeekMoE 等设计，但它清楚说明：KV cache 已经成为大模型推理中必须正面解决的核心瓶颈。

从 MHA 到 MQA、GQA、MLA，可以看到注意力机制的演变方向并不是“让注意力更复杂”，而是“让注意力更适合推理服务”。



### 第五条主线：长上下文从全局注意力走向局部-全局混合

自注意力的一个经典问题是复杂度与序列长度强相关。训练时完整注意力通常需要计算 token 两两之间的关系，序列长度越长，代价增长越快。推理时虽然有 KV cache，但上下文越长，缓存和读取成本仍然会持续上升。

这就引出一个问题：长上下文是否一定需要每一层、每一个 token 都能看见所有历史 token？

很多模型的答案是：不一定。

Mistral 7B 使用 Sliding Window Attention。它让注意力主要关注局部窗口，而不是每一层都做完整全局注意力。直觉上，这像人读长文：并不是每读一个字都回头扫描全文，而是更多依赖附近上下文；真正需要全局信息时，再通过更高层表示逐步传播。

Gemma 2 和 Gemma 3 则进一步把 local/global attention 做成更系统的设计。Gemma 2 使用局部和全局注意力交替，并引入 logit soft-capping 来限制 attention logits 和最终 logits 的极端值。Gemma 3 技术报告中则明确提到，为了降低长上下文下 KV cache memory 的爆炸，模型增加了 local attention 相对于 global attention 的比例，并支持至少 128K tokens 的上下文。

这背后是一种很实际的判断：

> 长上下文不是把所有层都粗暴扩成全局注意力，而是在信息传播路径和系统成本之间做设计。

局部注意力的代价是，模型不能在每一层都直接访问任意远的 token。它需要依赖多层堆叠，把远距离信息逐步传递过来。好处是，推理成本和显存压力会明显下降。

所以长上下文模型的核心不只是“位置编码能不能外推”，还包括：

1. RoPE 如何 scaling 或 interpolation
2. KV cache 如何压缩
3. attention 是否全局
4. 局部层和全局层如何排列
5. 推理框架能否高效支持这些 pattern

这也是为什么很多模型宣传 128K、1M 上下文时，不能只看上下文长度数字，还要看它在真实任务中的检索能力、长文一致性、生成速度和显存占用。

长上下文不是免费午餐，它只是把成本转移到了更复杂的架构和系统实现中。



### 第六条主线：从 Dense FFN 走向 MoE

Transformer 的另一个重要演变方向是 MoE，也就是 Mixture of Experts。

传统 dense 模型中，每个 token 都会经过同一套前馈网络。模型越大，每个 token 激活的参数也越多，计算成本随之上升。

MoE 的想法是：不让每个 token 都经过所有专家，而是由 router 为每个 token 选择少数几个专家处理。

这很像医院分诊。

如果所有病人都要依次看完内科、外科、眼科、骨科，成本一定很高。更合理的方式是先由分诊系统判断病人可能需要哪个科室，再把他送到对应专家那里。

MoE 也是类似：

```text
输入 token
   │
   ▼
router 判断该 token 适合哪些 expert
   │
   ▼
只激活少数 expert
   │
   ▼
合并 expert 输出
```

Mixtral 8x7B 是开源社区中非常有代表性的 MoE 模型。它有 8 个 experts，每个 token 选择 top-2 experts。论文中提到，Mixtral 总参数约 46.7B，但每个 token 只使用约 12.9B active parameters。也就是说，模型拥有更大的总容量，但每次推理只激活其中一部分。

DeepSeekMoE 则进一步提出细粒度 expert 和 shared experts。细粒度 expert 让专家划分更细，shared experts 用来捕获通用知识，避免所有知识都被路由到专门专家中。

DBRX、DeepSeek-V2/V3、Llama 4 等模型也说明，MoE 已经从研究分支进入主流模型家族。

但 MoE 并不是白捡参数。它带来的复杂度也非常明显：

1. router 训练是否稳定？
2. token 是否会集中到少数专家，导致负载不均？
3. expert 并行如何做？
4. sparse kernel 是否高效？
5. serving 时如何控制延迟抖动？
6. 多机通信是否会成为瓶颈？

这也是为什么 MoE 很适合大厂和强工程团队，但对个人或小团队并不一定友好。它用架构复杂度换取了更大的参数容量和更低的 active compute。

换句话说，MoE 的本质不是“模型突然拥有多个大脑”，而是“在固定计算预算下，允许模型存储更多可选择的知识和能力”。



### 第七条支线：Transformer 与 Mamba 的混合

除了在 Transformer 内部做修改，还有一些模型开始尝试把 Transformer 与其他序列模块混合。

Jamba 是一个代表案例。它把 Transformer 和 Mamba 层交替组合，并在部分层中加入 MoE。Mamba 属于状态空间模型路线，相比标准注意力，它在长序列处理上具有不同的计算和内存特性。

这类 hybrid 架构的动机很清楚：

- Transformer 质量强，生态成熟
- Mamba 等状态空间模型在长上下文吞吐和内存上有潜力
- MoE 可以继续扩大容量

于是一个自然想法是：能不能把它们组合起来？

但混合架构的问题也很明显。它不再是纯 Transformer，训练 recipe、推理 kernel、并行策略、缓存机制都更复杂。它是否会成为通用 LLM 的主流，目前还没有像 LLaMA-like Transformer 那样形成压倒性共识。

所以更稳妥的判断是：Hybrid 架构代表了“后 Transformer 时代”的探索，但还没有真正取代 Transformer 主线。



### 一条时间线：从 RoPE 到 MLA 和 MoE

如果把 2021 年之后的开源大模型结构演变压缩成一条时间线，大致可以这样理解：

```text
2021
  GPT-J：较早在开源 GPT 模型中使用 RoPE
  RoFormer：系统提出 Rotary Position Embedding
  ALiBi：提出 attention linear biases，用于长度外推

2022
  GPT-NeoX：延续 RoPE 和 GPT-J 风格残差结构
  BLOOM：采用 ALiBi 和 embedding layer norm
  OPT：更像 GPT-3 风格的开放复现基线

2023
  LLaMA：RMSNorm + SwiGLU + RoPE 成为现代开源主线
  Llama 2 70B：引入 GQA
  Falcon / ChatGLM2：展示 MQA 或 KV 共享对推理的价值
  Mistral 7B：GQA + Sliding Window Attention 成为高效长上下文模板

2024
  Mixtral：MoE 在开源主流模型中爆发
  DeepSeekMoE：细粒度专家 + shared experts
  DeepSeek-V2：MLA + MoE，重点压缩 KV cache
  Gemma 2：local/global attention + GQA + soft-capping
  Qwen2.5：RoPE + SwiGLU + RMSNorm + GQA + QKV bias，支持长上下文
  Jamba：Transformer-Mamba hybrid + MoE
  OLMo 2：RMSNorm、RoPE、QK-Norm 等稳定性改造

2025
  Gemma 3：进一步提高 local attention 比例，面向 128K 长上下文降低 KV cache 压力
  DeepSeek-V3：继续放大 MLA + DeepSeekMoE 路线
  Llama 4：Llama 主线进入 MoE 和原生多模态方向
```

需要注意的是，“首次出现”在技术史中很难绝对判断。很多技术可能先出现在论文、小模型、内部系统或未公开模型中，后来才在开源大模型中被广泛使用。更重要的不是争论谁第一个提出，而是理解哪些设计最终变成了主流组合。

从这个角度看，影响最大的组合可能不是最激进的 MLA 或 MoE，而是下面这套基础栈：

```text
RMSNorm / Pre-Norm
      +
SwiGLU
      +
RoPE
      +
GQA
```

这套组合成为 2023 年之后大量开源模型的共同底座。其他设计，比如 local-global attention、QK-Norm、MLA、MoE，则是在不同目标下继续叠加的优化。



### 为什么这些修改会汇合？

现在可以回到最初的问题：Transformer 为什么会变成今天这个样子？

一个很直接的答案是：因为大模型的瓶颈变了。

早期我们更关心训练能否跑起来，模型能否学会语言规律。后来当模型参数变大、上下文变长、用户规模变大以后，问题变成了：

1. 训练是否稳定？
2. 推理是否足够快？
3. 显存是否能放下 KV cache？
4. 长上下文是否真的可用？
5. 参数继续扩大时，计算成本是否还能承受？
6. serving 系统能否支撑真实业务负载？

于是不同修改分别对应不同瓶颈：

| 修改 | 主要解决的问题 |
|---|---|
| RMSNorm / Pre-Norm | 大规模训练稳定性 |
| SwiGLU | FFN 表达能力和参数效率 |
| RoPE | 相对位置信息和长上下文扩展基础 |
| ALiBi | 简洁的长度外推 |
| MQA | 极致压缩 KV cache |
| GQA | 能力与 KV cache 的折中 |
| MLA | 重新设计 attention cache 形态 |
| Sliding Window / Local-Global | 降低长上下文注意力成本 |
| QK-Norm / Soft-capping | 控制 attention logits 数值稳定性 |
| MoE | 固定 active compute 下扩大总参数容量 |
| Hybrid Transformer-Mamba | 探索长序列建模的新骨架 |

如果用一句话概括：

> Transformer 的演变，不是为了让结构看起来更复杂，而是为了让它在更大规模、更长上下文、更低延迟和更低显存下仍然可训练、可推理、可部署。

这也是 AI Infra 和模型架构越来越难分开的原因。很多架构选择，本质上不是纯算法问题，而是系统问题。

GQA 是否值得使用，要看推理吞吐和 KV cache；

MoE 是否值得使用，要看路由、负载均衡和并行通信；

长上下文是否真实可用，要看 attention pattern、KV cache、RoPE scaling 和推理框架；

MLA 是否能发挥作用，要看底层 serving 系统是否能适配新的缓存格式。

模型结构和基础设施正在互相塑造。



### 对开发者有什么意义？

如果只是调用 API，似乎并不需要理解这些细节。但对开发者来说，理解 Transformer 的演变至少有三个价值。

第一，能更好理解模型能力边界。

例如，当一个模型支持 128K 上下文，不代表它真的能在 128K 内稳定找出所有细节。长上下文能力受到位置编码、注意力模式、训练数据、检索任务分布和推理实现共同影响。

第二，能更好理解推理成本。

同样是 7B 模型，MHA、MQA、GQA 的 KV cache 成本可能完全不同。上下文长度、batch size、并发请求、显存占用都会受到影响。对于部署者来说，这些结构差异不是论文细节，而是机器成本。

第三，能避免把模型名字当作全部。

很多模型都叫 Transformer，但内部差异很大。LLaMA-like dense model、Mixtral MoE、DeepSeek MLA、Gemma local-global、Jamba hybrid，它们的推理行为和部署约束并不一样。

这就像都叫数据库，MySQL、Redis、ClickHouse、Elasticsearch 的适用场景完全不同。模型也是一样。

当我们选择模型时，不应该只看参数量和榜单分数，还要看：

1. attention 结构是什么？
2. 是否使用 GQA/MQA/MLA？
3. 上下文长度如何实现？
4. 是否 MoE？active parameters 多少？
5. 推理框架是否原生支持？
6. 目标任务是否真的需要长上下文或超大参数？

这些问题比“哪个模型更强”更接近工程现实。



### 总结

Transformer 从 2017 年发展到今天，已经不再只是论文中那个 encoder-decoder 架构。它更像是一套可以不断替换部件的通用计算骨架。

在这套骨架里：

- 注意力机制负责上下文交互
- FFN 承载大量知识和非线性变换
- 位置编码让模型理解顺序
- 归一化保证训练稳定
- KV cache 决定推理成本
- MoE 决定参数扩展方式
- 局部/全局注意力决定长上下文是否可部署

过去几年开源大模型的演变，大体可以概括为四个方向：

1. 更稳定：LayerNorm 到 RMSNorm，post-norm 到 pre-norm，soft-capping 和 QK-Norm
2. 更高效：MHA 到 MQA/GQA/MLA，减少 KV cache 成本
3. 更长上下文：RoPE、ALiBi、RoPE scaling、Sliding Window、Local-Global Attention
4. 更大容量：SwiGLU 提升 FFN 表达，MoE 在固定 active compute 下扩大总参数

如果说早期 Transformer 的成功来自“注意力机制让序列建模可以并行化”，那么今天 Transformer 的生命力则来自“它足够模块化，能够不断被工程化改造”。

这也是为什么很多人不断讨论“Transformer 之后是什么”，但 Transformer 仍然没有被轻易取代。因为它已经不只是一个单一架构，而是一套被硬件、数据、训练框架、推理系统和开源生态共同打磨出来的工程标准。

当然，这并不意味着 Transformer 会永远占据中心。Mamba、RWKV、RetNet、Jamba 这类路线都在探索新的序列建模方式。但至少从目前开源大模型的发展来看，更现实的路径并不是突然抛弃 Transformer，而是继续把它改造成更可缓存、更可稀疏、更可局部化，也更容易和其他序列模块混合的基础骨架。

所以，今天我们再回看《Attention Is All You Need》时，也许可以稍微改写一下这个标题：

> Attention 是起点，但真正支撑大模型走向工业化的，是围绕 Attention 进行的一整套工程权衡。



### 参考资料

1. Vaswani et al., Attention Is All You Need, 2017, https://arxiv.org/abs/1706.03762
2. Su et al., RoFormer: Enhanced Transformer with Rotary Position Embedding, 2021, https://arxiv.org/abs/2104.09864
3. Zhang and Sennrich, Root Mean Square Layer Normalization, 2019, https://arxiv.org/abs/1910.07467
4. Shazeer, GLU Variants Improve Transformer, 2020, https://arxiv.org/abs/2002.05202
5. Shazeer, Fast Transformer Decoding: One Write-Head is All You Need, 2019, https://arxiv.org/abs/1911.02150
6. Ainslie et al., GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints, 2023, https://arxiv.org/abs/2305.13245
7. Press et al., Train Short, Test Long: Attention with Linear Biases Enables Input Length Extrapolation, 2021, https://arxiv.org/abs/2108.12409
8. Touvron et al., LLaMA: Open and Efficient Foundation Language Models, 2023, https://arxiv.org/abs/2302.13971
9. Touvron et al., Llama 2: Open Foundation and Fine-Tuned Chat Models, 2023, https://arxiv.org/abs/2307.09288
10. Jiang et al., Mistral 7B, 2023, https://arxiv.org/abs/2310.06825
11. Jiang et al., Mixtral of Experts, 2024, https://arxiv.org/abs/2401.04088
12. Gemma Team, Gemma: Open Models Based on Gemini Research and Technology, 2024, https://arxiv.org/abs/2403.08295
13. Gemma Team, Gemma 2: Improving Open Language Models at a Practical Size, 2024, https://arxiv.org/abs/2408.00118
14. DeepSeek-AI, DeepSeekMoE: Towards Ultimate Expert Specialization in Mixture-of-Experts Language Models, 2024, https://arxiv.org/abs/2401.06066
15. DeepSeek-AI, DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model, 2024, https://arxiv.org/abs/2405.04434
16. DeepSeek-AI, DeepSeek-V3 Technical Report, 2024, https://arxiv.org/abs/2412.19437
17. Lieber et al., Jamba: A Hybrid Transformer-Mamba Language Model, 2024, https://arxiv.org/abs/2403.19887
18. Team OLMo, 2 OLMo 2 Furious, 2024/2025, https://arxiv.org/abs/2501.00656
19. Gemma Team, Gemma 3 Technical Report, 2025, https://arxiv.org/abs/2503.19786
20. Qwen2.5-7B Model Card, https://huggingface.co/Qwen/Qwen2.5-7B
21. Mistral-7B-v0.1 Model Card, https://huggingface.co/mistralai/Mistral-7B-v0.1
22. DeepSeek-V2 GitHub README, https://github.com/deepseek-ai/DeepSeek-V2
