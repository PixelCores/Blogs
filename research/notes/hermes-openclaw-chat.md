# Hermes 和 OpenClaw 对比，以及 Hermes 持久记忆机制

## 用户

hermnes 和 openclaw 的区别在什么地方？

## 助手

你这里的 “hermnes” 我按 “Hermes Agent” 理解；OpenClaw 我按 openclaw/openclaw 这个个人 AI 助手项目理解。

一句话区别：

Hermes 更像“可扩展的通用 AI Agent 框架/CLI + 多平台网关”；
OpenClaw 更像“本地优先、面向个人助手体验的多端/多频道产品”。

核心差异大概是这些：

### 1. 定位不同

Hermes Agent:
- 重点是让 AI 在你的终端、IDE、聊天平台里做事。
- 更偏“开发者/运维/研究/自动化 agent 框架”。
- 类似 Claude Code、Codex、OpenCode 这一类可执行任务的 agent。
- 强调工具调用、文件/终端操作、技能、记忆、cron、子代理、MCP、插件等。

OpenClaw:
- 重点是“你自己的个人 AI 助手”。
- 更偏“常驻网关 + 多聊天渠道 + 移动/桌面伴随应用 + 语音/Canvas”。
- 目标体验更像一个跨 Telegram、WhatsApp、Slack、Discord、iMessage、手机、桌面等入口的个人助手系统。
- 强调 Gateway、channels、nodes、Live Canvas、移动端/桌面端集成。

### 2. 交互入口不同

Hermes:
- 终端 CLI 是一等入口：`hermes`
- 也支持 Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Email 等 gateway。
- 适合你直接在 shell 里让 agent 改代码、跑命令、查文件、做自动化。

OpenClaw:
- Gateway/多频道是中心。
- 支持大量聊天渠道，并且强调 macOS/iOS/Android companion apps、语音唤醒、Talk Mode、Live Canvas。
- 更像“把 AI 助手接到你日常通信和设备上”。

### 3. 技能/记忆机制侧重点不同

Hermes:
- “Skills” 是核心卖点之一。
- 它会把复杂工作流程沉淀成可复用技能文档，以后自动加载。
- 有持久记忆：用户偏好、环境事实、项目约定等。
- 很适合长期使用后逐渐变成“懂你工作流”的 agent。

OpenClaw:
- 也有 skills / ClawHub / onboarding-driven skills。
- 但整体叙事更偏“个人助手生态 + 多端入口 + gateway 控制面”。
- 技能是产品能力的一部分，不是 Hermes 那样最核心的差异点。

### 4. 工具和自动化能力

Hermes:
- 工具体系很明确：terminal、file、browser、web、vision、image_gen、tts、cronjob、delegation、MCP、session_search、memory 等。
- 子代理 delegate_task、cron 定时任务、webhook、profiles、credential pools 都是内置工作流。
- 更适合复杂任务执行，比如：
  - 改代码
  - 跑测试
  - 调试服务
  - 定时监控
  - 多 agent 并行处理
  - 读写文档/仓库/系统资源

OpenClaw:
- 工具也很多：browser、canvas、nodes、cron、sessions、Discord/Slack actions 等。
- 但特色更多在：
  - 多频道消息路由
  - 多 agent 路由
  - companion app 节点
  - Live Canvas
  - 语音和设备侧能力
  - Gateway 作为本地控制平面

### 5. 技术栈和安装体验

Hermes:
- Python 生态为主。
- 安装方式类似：

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

- 配置命令：

```bash
hermes setup
hermes model
hermes tools
hermes doctor
```

OpenClaw:
- Node/TypeScript 生态为主。
- 推荐 Node 24 或 Node 22.16+。
- 安装方式类似：

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

- 更强调 onboard 流程和 gateway daemon。

### 6. 安全模型侧重点

Hermes:
- 终端/文件/工具权限很强，所以重点在工具启用、审批模式、secret redaction、gateway 授权等。
- 有 yolo、approvals.mode、toolsets、platform pairing 等机制。
- 适合“我明确让 agent 在这台机器上干活”的场景。

OpenClaw:
- README 里特别强调“真实消息入口的 DM 是不可信输入”。
- 默认很多聊天渠道需要 pairing，未知私信不会直接处理。
- 还强调 group/channel sandbox，例如非主会话放进沙箱。
- 因为它更面向常驻、多渠道、远程消息入口，所以安全边界更偏通信/网关/沙箱。

### 7. 哪个更适合你？

选 Hermes，如果你主要想要：
- 一个强力 CLI agent
- 改代码、跑命令、读写文件、调试项目
- 长期记住你的偏好和工作流
- 用 skills 沉淀经验
- cron 定时任务、webhook、MCP、子代理
- 在终端和聊天平台里都能干活

选 OpenClaw，如果你主要想要：
- 一个常驻的个人 AI 助手
- Telegram/WhatsApp/Slack/Discord/iMessage 等多频道接入
- 手机、桌面、语音、Canvas 体验
- Gateway 统一管理你的 assistant
- 更偏“生活/沟通入口里的 AI 助手”，而不只是终端 agent

我的直观判断：

如果你的使用场景是“开发、自动化、运维、研究、让 AI 在电脑上完成任务”，Hermes 更顺手。

如果你的使用场景是“我要一个长期在线、接入各种聊天软件和设备的个人助手”，OpenClaw 更贴近。

另外 Hermes 里还有一个命令：

```bash
hermes claw migrate
```

说明 Hermes 本身也考虑了从 OpenClaw 迁移过来的路径。也就是说两者有重叠，但 Hermes 更偏 agent framework / CLI execution，OpenClaw 更偏 personal assistant / gateway product。

---

## 用户

有持久记忆 我对这个很感兴趣，那么Hermes是如何实现的呢？为什么会认为越用越好用呢？

## 助手

Hermes 的“持久记忆”大概分两层：

1. Memory：记住事实和偏好
2. Skills：记住可复用的做事方法

这两个加起来，才是“越用越好用”的核心。

### 一、Memory 是什么？

Memory 更像“长期用户档案 + 环境笔记”。

它会保存一些跨会话仍然有价值的信息，例如：

- 你喜欢中文回答
- 你偏好简洁还是详细
- 你常用的项目目录在哪里
- 你的机器是 macOS / Linux / WSL
- 某个项目用 pnpm 而不是 npm
- 某个仓库跑测试要用特定命令
- 你不喜欢 agent 自动做某类操作
- 你习惯用某个模型、某种代码风格、某种提交规范

这些信息不会只存在当前聊天里，而是存进 Hermes 的持久 memory backend。以后新开一个 Hermes session，它仍然可以读到这些事实。

Hermes 的 skill 文档里提到，它支持 pluggable memory backends，比如：

- 内置 memory
- Honcho
- Mem0
- 以及其他可插拔后端

所以它不是简单地把所有历史对话塞进 prompt，而是把“长期有用的事实”抽出来保存。

### 二、Memory 不等于聊天记录

这是一个很关键的区别。

聊天记录是“上次我们说了什么”；
Memory 是“以后还值得记住的事实”。

比如：

用户说：
“以后回答我尽量用中文，不要太啰嗦。”

这适合进 memory。

但如果用户说：
“今天帮我修这个 bug。”

这不适合进 memory，因为这是临时任务进度。

Hermes 的设计原则是：

- 用户偏好：适合记
- 环境事实：适合记
- 项目长期约定：适合记
- 临时任务状态：不适合记
- 大段原始数据：不适合记
- 一次性对话内容：不适合记

这样可以避免 memory 变脏。

### 三、Skills 是什么？

Skills 是 Hermes 更有意思的部分。

Memory 记的是“事实”；
Skills 记的是“流程”。

例如你让 Hermes 多次帮你做类似事情：

- 给 GitHub PR 做 code review
- 部署某个项目
- 处理某种日志报错
- 生成某种格式的报告
- 用某个内部工具跑一套流程
- 给某类代码写测试
- 调试某个框架的常见问题

如果这个流程复杂、步骤多、容易踩坑，Hermes 可以把它沉淀成一个 skill 文档。

Skill 里面通常会包括：

- 什么时候应该加载这个 skill
- 具体操作步骤
- 常用命令
- 验证方式
- 常见坑
- 项目/工具的特殊约定

下次你再提类似任务时，Hermes 会先加载相关 skill，再开始执行。

这就有点像给 agent 建了一个“个人工作手册”。

### 四、为什么说 Hermes 会越用越好用？

因为它不只是“每次从零开始聊天”。

普通 ChatGPT/Claude 类工具通常是：

- 当前对话里知道上下文
- 新开对话后很多东西要重说
- 项目习惯、个人偏好、踩过的坑容易忘

Hermes 的逻辑是：

第一次：
你告诉它你的偏好、项目结构、工具链。
它可能会犯错，需要你纠正。

第二次：
它可以记住：
“这个用户偏好中文。”
“这个项目用 pnpm。”
“这个仓库测试命令是 pnpm test。”
“不要直接 npm install。”
“这个用户希望先给计划再改代码。”

第三次：
如果它曾经解决过复杂问题，可以有 skill：
“调试这个项目时先检查 config.yaml，再跑 hermes doctor，再看 ~/.hermes/logs/gateway.log。”

所以后续任务里，它可以少问你很多问题，少走弯路。

### 五、举个具体例子

假设你第一次让 Hermes 帮你维护一个项目。

你告诉它：

- 项目在 `~/code/myapp`
- 包管理器是 pnpm
- 测试命令是 `pnpm test`
- lint 命令是 `pnpm lint`
- 不要直接改 main 分支
- 提交信息要用 conventional commits
- 你希望回答用中文

这些可以变成 memory。

然后某次它帮你解决了一个很难的部署问题，流程是：

1. 检查 `.env`
2. 跑 `docker compose config`
3. 看 `gateway.log`
4. 发现端口冲突
5. 修改 compose override
6. 验证 health endpoint

这个复杂流程可以变成 skill。

下次你说：

“这个项目又起不来了，帮我看一下。”

Hermes 就不用从零问你：

- 项目在哪里？
- 用 npm 还是 pnpm？
- 日志在哪？
- 怎么验证？
- 你要中文还是英文？

它可以直接基于 memory 和 skill 开始排查。

### 六、Hermes 的持久记忆大概怎么进入上下文？

高层理解是：

每次新会话开始或执行任务时，Hermes 会构建 system prompt / agent context。

里面可能包含：

- 当前用户 profile
- 相关 memory
- 相关 skills
- 当前启用的工具
- 平台信息
- 会话状态
- 必要的环境约定

所以模型不是“真的大脑永久变聪明”，而是 Hermes 在每次调用模型前，把跟你相关的长期知识重新注入给模型。

这就是 agent 系统层面的“持久记忆”。

### 七、Memory 和 Skills 的区别总结

Memory:
- 记事实
- 短小
- 长期有效
- 例如：用户偏好、项目约定、环境信息

Skills:
- 记流程
- 结构化文档
- 可复用
- 例如：如何调试某系统、如何发布某项目、如何做某类 review

Session history:
- 记对话
- 用于回忆过去发生了什么
- 不一定每次自动注入
- 适合查“上次我们做了什么”

### 八、为什么这比普通“聊天历史”更好？

因为普通聊天历史有几个问题：

1. 太长
   全部塞进上下文会爆 token。

2. 噪声太多
   大量临时信息、废话、失败尝试会污染判断。

3. 不结构化
   模型很难从几百页历史里稳定提取“真正该记住的规则”。

4. 不可维护
   错误信息一旦混进去，后面可能反复误用。

Hermes 的设计更像：

- 把“事实”压缩成 memory
- 把“流程”沉淀成 skill
- 把“历史细节”留在 session search 里按需检索

这比单纯保存聊天记录更可靠。

### 九、它会不会记错？

会有这个风险。

所以 Hermes 的设计里有几个原则：

- 只保存长期有价值的信息
- 不保存临时任务进度
- memory 要简洁
- skill 如果过时，可以 patch 更新
- 如果用户纠正了 agent，优先更新 memory/skill
- skills 有 curator 机制做生命周期管理

换句话说，Hermes 的“越用越好用”不是魔法，而是靠长期维护一套个人化知识库。

### 十、我的理解

Hermes 的持久记忆不是单纯“我记得你聊过什么”，而是：

“我知道你是谁、你喜欢怎么工作、你的环境是什么样、你的项目有哪些固定约定、以前哪些流程被证明有效。”

所以它越用越好用的原因是：

- 更少重复解释
- 更少犯同样错误
- 更贴合你的偏好
- 更了解你的项目和机器
- 能复用过去成功的复杂流程
- 能从一次性经验沉淀成长期能力

如果你主要把 AI 当“个人长期工作助理”而不是“一次性聊天机器人”，Hermes 这套 memory + skills 的设计确实是它最有价值的地方之一。

---

## 用户

将刚才我们的聊天记录输出成一个md文件
