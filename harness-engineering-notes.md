# Harness Engineering 深度学习笔记

> 综合来源：Mitchell Hashimoto 原文、Martin Fowler / Birgitta Böckeler 分析、OpenAI Codex 团队实战报告、Cobus Greyling 架构综述、中文深度解析文章
>
> 整理时间：2026-03-15

---

## 一、从一个问题说起：为什么你的 Agent 会在第 50 步掉链子？

2026 年的 AI 工程界有一个被反复验证的现象——**模型漂移（Model Drift）**。一个在单次推理测试中表现惊艳的模型，当你让它处理一个长达数百步的复杂工作流时，大约在第 50 步开始出现微小的推理偏差，然后像滚雪球一样越偏越远，最终完全偏离初始指令。

这个现象揭示了一个残酷的事实：**瓶颈不在模型够不够聪明，而在系统够不够耐久。** 顶级模型之间的"智商"差距已经缩小到 1% 以内，但在真实的长时任务中这个差距几乎无意义。真正决定成败的，是模型运行时的"环境工程"。

这就引出了 2026 年 AI 领域最核心的新学科：**Harness Engineering**。

---

## 二、定义：什么是 Harness？

### 2.1 术语溯源

"Harness" 这个词源自马具——缰绳、马鞍、嚼子——整套用于引导一匹强大但不可预测的动物走正确方向的装备。隐喻非常精确：

- **马 = AI 模型**：强大、快速，但自己不知道往哪跑
- **马具 = Harness**：约束、护栏、反馈回路，把模型的力量引向正确方向
- **骑手 = 人类工程师**：提供方向，而不是自己跑

这个术语由 HashiCorp 创始人 Mitchell Hashimoto 在 2026 年 2 月首次正式提出。他在自己的 AI 采纳旅程博文中将第五阶段命名为 "Engineer the Harness"，核心思想是：**每当你发现 Agent 犯了一个错误，你就花时间设计一个方案，让 Agent 永远不再犯同样的错。** 这不是一次性修复，而是系统性免疫。

几乎同时，OpenAI 在一篇关于 Codex 的文章中使用了 "Harness Engineering" 作为标题，描述了他们用零行手写代码构建百万行产品的实验。Martin Fowler 的网站随即发布了 Birgitta Böckeler 的深度分析，确认了这是一个值得认真对待的架构范式。

### 2.2 核心定义

**Harness 不是 Agent 本身，而是管理 Agent 如何运行的软件系统。**

它管理完整的生命周期——工具、记忆、重试、人类审批、上下文工程、子 Agent——让模型可以专注于推理。

用 Philipp Schmid 的计算机类比来理解：

| 概念 | 类比 | 职责 |
|------|------|------|
| 模型 (Model) | CPU | 原始算力，逻辑核心 |
| 上下文窗口 (Context Window) | RAM | 有限且易失的工作内存 |
| **Agent Harness** | **操作系统 (OS)** | **资源调度、内存优化、驱动管理、标准运行环境** |
| AI Agent | 应用程序 (App) | 运行在 Harness 之上，专注于业务逻辑 |

### 2.3 Harness 在架构栈中的位置

之前构建 AI Agent 有三种主流方式：SDK（如 Anthropic Agent SDK）、Framework（如 LangChain）、Scaffolding（如脚手架式工程）。它们回答的问题是 **"how you build an agent"**。

而 Harness 回答的是一个完全不同的问题：**"how the agent runs"**。

你可以用上面三种方式中的任何一种来构建 Harness。Harness 不是它们的替代品，而是它们之上的一层。

| 维度 | SDK | Framework | Scaffolding | **Harness** |
|------|-----|-----------|-------------|-------------|
| 关注点 | 构建 | 结构 | 流程 | **运行时治理** |
| 开发者做什么 | 写代码调用模型 | 在框架内编排 | 定义流程管道 | **设计环境、约束和反馈** |
| 模型做什么 | 响应单次调用 | 在框架内推理 | 执行管道步骤 | **制定计划、自主执行** |
| 核心价值 | 灵活性 | 结构化 | 可复现性 | **长时稳定性** |

---

## 三、为什么 Harness Engineering 是 2026 年最重要的事

### 3.1 模型已是商品，Harness 才是壁垒

LangChain 在 Terminal Bench 2.0 上的数据给了我们一记反直觉的重击：**底层模型参数完全不动，仅通过优化运行环境（Harness），任务得分从 52.8% 飙升至 66.5%，排名从全球第 30 位跃升至第 5 位。**

他们做了什么？

- 增加了 self-verification loop（预提交检查清单中间件）
- 做了 context engineering（启动时映射目录结构）
- 加了 loop detection（跟踪重复文件编辑，防止"死循环"）
- 用了 reasoning sandwich（规划/验证用高推理力模型，实现用中等推理力模型）

同一个模型，不同的 Harness，天壤之别的结果。

### 3.2 OpenAI 的百万行代码实验

OpenAI 团队在 2025 年 8 月底开始了一个极端实验：从空 Git 仓库开始，用零行手写代码构建一个完整的内部产品。5 个月后，这个产品超过了一百万行代码，有内部日活用户和外部 Alpha 测试者，完整地经历了部署、崩溃和修复的全周期。

他们的核心发现：**早期进展比预期慢，不是因为 Codex 不够能干，而是因为环境描述不足（underspecified）。** Agent 缺乏工具、抽象层和内部结构来完成高层目标。当出问题时，修复方案几乎从来不是"再试一次"，而是反复追问一个问题："缺了什么能力？怎么让这个能力对 Agent 既可读（legible）又可执行（enforceable）？"

人类工程师的主要工作变成了：**设计环境、明确意图、构建反馈回路——让 Codex Agent 能可靠地工作。**

### 3.3 Framework 层正在塌缩进 Harness

一个重要的结构性变化正在发生：模型正在吸收过去由多 Agent 框架处理的能力。Agent 定义、消息路由、任务生命周期、依赖管理、Worker 生成……大约 80% 的框架功能，模型现在可以原生处理。

**剩下的 20%——持久化、确定性重放、成本控制、可观测性、错误恢复——恰好就是 Harness 提供的东西。**

智能迁入模型，基础设施迁入 Harness。这不是框架在消失，而是框架在分裂。

---

## 四、Harness 的三大支柱（深度解析）

Birgitta Böckeler 在 Martin Fowler 站点上对 OpenAI 文章的分析，将 Harness 的组件归为三类。这是当前业界最被认可的分类框架。

### 4.1 支柱一：上下文工程（Context Engineering）

**这是 Harness Engineering 中最深、最值得花时间理解的部分。**

上下文工程不等于"写好 prompt"。它是一个动态的、多层次的信息管理系统，目标是确保 Agent 在每次模型调用时，能获取到恰好正确的信息——不多不少。

#### 4.1.1 从"静态提示模板"到"动态上下文选择"

传统 prompt engineering 是静态的：你写一段 system prompt，然后祈祷它在所有情况下都管用。**上下文工程是动态的：它根据当前任务状态，主动选择什么信息应该出现在这次模型调用中。**

OpenAI 团队明确发现了"大一统 AGENTS.md"的失败模式：

1. **上下文是稀缺资源**。一个巨大的指令文件会挤占任务本身、代码和相关文档的空间，导致 Agent 要么遗漏关键约束，要么朝错误方向优化。
2. **当一切都"重要"时，什么都不重要。** Agent 最终只是在本地做模式匹配，而不是有目的地导航。
3. **它会立即腐烂。** 一个单体手册变成陈旧规则的坟场。Agent 分不清什么还有效，人类也懒得维护它。
4. **它很难验证。** 你怎么知道这些规则是否真的在被遵循？

解决方案是分层、可导航、可机器验证的上下文结构。

#### 4.1.2 静态上下文 vs 动态上下文

**静态上下文**指那些相对稳定、预先准备好、存放在仓库中的知识：

- **仓库本地文档**：架构规范、API 契约、风格指南
- **AGENTS.md / CLAUDE.md 文件**：编码项目特定规则——但要精简、分层、可验证
- **交叉链接的设计文档**：通过 linter 验证文档间的引用一致性
- **结构化的 docs/ 目录**：包含 map（全局导航）、execution plan（执行计划）和 design specification（设计规范），作为 Agent 的"唯一事实来源"

**动态上下文**指那些在运行时才产生、需要实时注入的信息：

- **可观测性数据**：日志、指标、链路追踪（traces）——Agent 可以直接读取线上遥测数据来定位问题
- **目录结构映射**：Agent 启动时自动扫描代码库结构，建立心智地图
- **CI/CD 管道状态**：当前构建是否通过？哪些测试失败了？
- **浏览器状态**：对于 CUA（Computer Use Agent），屏幕截图 → 动作 → 验证的循环
- **终端实时回传**：Agent 执行命令后的实时输出

#### 4.1.3 "给地图，不给千页手册"

OpenAI 团队的核心经验可以浓缩为一句话："Give Codex a map, not a 1,000-page instruction manual."

具体操作方式：

- **仓库即知识系统（Repository as System of Record）**：从 Agent 的角度，任何它在上下文中访问不到的东西等于不存在。Google Docs 里的决策、Slack 里的讨论、人脑里的知识——对系统来说都是不可见的。所有 Agent 需要的信息必须在仓库中，版本化、可发现。
- **为 Agent 可读性（Legibility）而设计**：就像为新入职工程师优化代码可导航性一样，为 Agent 优化仓库的可推理性。代码设计本身就是上下文的核心组成部分。
- **渐进式上下文加载**：不是一次塞给 Agent 所有信息，而是建立"导航 → 发现 → 深入"的层级结构，让 Agent 按需获取。

#### 4.1.4 上下文工程与传统 RAG 的区别

早期的 RAG（Retrieval Augmented Generation）是被动的：用户提问 → 检索相关文档 → 拼接到 prompt。

上下文工程是主动的、架构级的：

| 维度 | 传统 RAG | 上下文工程 |
|------|---------|-----------|
| 触发 | 用户查询 | 任务状态变化 |
| 粒度 | 文档片段 | 多层上下文（静态+动态） |
| 来源 | 外部知识库 | 仓库、遥测、CI、浏览器、终端 |
| 维护 | 手动更新索引 | 自动化验证和清理 |
| 目标 | 回答问题 | 赋能 Agent 持续自主工作 |

### 4.2 支柱二：架构约束（Architectural Constraints）

这是 Harness Engineering 与传统 AI prompting 分歧最大的地方。不是告诉 Agent "写好代码"，而是**机械性地强制执行好代码的标准。**

#### 4.2.1 依赖分层（Dependency Layering）

OpenAI 的 Codex 团队建立了严格的依赖流向：

```
Types → Config → Repo → Service → Runtime → UI
```

每一层只能从左边的层导入。这不是建议，而是通过结构性测试和 CI 验证强制执行的。跨层依赖会被立即拒绝。

跨切关注点（认证、遥测、特性开关）通过一个单一的显式接口 `Providers` 进入系统，而不是随意散落。

#### 4.2.2 约束执行的四种工具

1. **确定性 Linter（Deterministic Linters）**：自定义规则自动标记违规。这是传统软件工程的工具，但在 Harness 中变得更加关键——因为 Agent 生成代码的速度远超人类，如果没有自动化检查，质量债务会瞬间失控。

2. **LLM 驱动的审计 Agent（LLM-based Auditors）**：用一个 Agent 审查另一个 Agent 的代码是否符合架构约束。这是"Agent 对 Agent"闭环的核心。

3. **结构性测试（Structural Tests）**：类似 ArchUnit，但专门为 AI 生成代码设计。验证模块边界、命名规范、文件大小限制。

4. **Pre-commit Hooks**：代码提交前的自动化检查门。

#### 4.2.3 反直觉的发现：约束释放潜能

一个普遍的直觉误区是"限制越多，AI 越笨"。但数据证明恰恰相反：

- 当 Agent 可以生成任何东西时，它浪费 token 探索死胡同
- 当 Harness 定义了清晰的边界时，Agent 更快地收敛到正确解

Vercel 团队曾尝试移除 Agent 中 80% 的手工预设工具，结果不仅降低了 Token 消耗，还因为减少了模型的认知负荷，获得了更快的响应速度。

**更高的 AI 自主化，必须以约束其运行时为代价。** 这是 2026 年最重要的工程直觉之一。

#### 4.2.4 "为删除而建"（Build for Removal）

Manus 团队在 6 个月内将其 Harness 架构重构了 5 次，核心目的只有一个：移除基于人类经验的"刚性假设"。

这个理念呼应了 AI 发展史上的"苦涩的教训"（The Bitter Lesson）：利用通用计算的技术，最终总会击败依赖人类经验手工编码的技术。

实操原则：**今天费尽心机编写的复杂控制逻辑，明天极有可能被一个更强大的模型提示词取代。保持架构高度模块化，让每个组件可以被独立替换或移除。**

### 4.3 支柱三：熵管理 / 垃圾回收（Entropy Management / Garbage Collection）

这是最被低估的组件。AI 生成的代码库随时间会积累熵——文档与现实脱节、命名规范偏离、死代码堆积。如果不治理，Agent 会开始复制已有的坏模式（因为 Codex 通过参考仓库中已有的模式来实现功能），形成系统性退化。

OpenAI 团队最初每周五花一整天手动清理"AI Slop"（低质量 AI 生成物）。这显然无法规模化。

解决方案是建立定期运行的"清扫 Agent"：

- **文档一致性 Agent**：验证文档是否与当前代码匹配
- **约束违规扫描器**：查找之前漏过检查的代码
- **模式执行 Agent**：识别并修复偏离既定模式的代码
- **依赖审计器**：追踪和解决循环或不必要的依赖

运作方式：

1. 后台 Codex 任务定期扫描偏差
2. 更新质量评级
3. 开出针对性的重构 Pull Request
4. 大多数 PR 一分钟内审查完毕，自动合并

这本质上是把技术债务视为高利贷——以小额日频增量偿还，而不是让它复利积累后痛苦地一次性处理。

**人类品味只需捕获一次，然后在每一行代码上持续强制执行。**

---

## 五、Mitchell Hashimoto 的六步采纳旅程

Mitchell Hashimoto（Vagrant、Terraform 的创造者）的博文之所以重要，是因为它提供了一个**从个人实践者视角**的渐进式采纳路径——从怀疑到精通。

### Step 1: 抛弃聊天界面（Drop the Chatbot）

停止通过 ChatGPT/Gemini 网页界面做正经编码工作。聊天界面的问题是你在"祈祷模型凭训练数据猜对"，然后反复说"不对，再试"。效率极低。

**必须转向 Agent**——一个能够读文件、执行程序、发 HTTP 请求的 LLM 循环。

### Step 2: 复现你自己的工作（Reproduce Your Own Work）

刻意练习阶段。Mitchell 强迫自己把每个手动提交都用 Agent 重做一遍——同一件事做两次。极其痛苦，但形成了三个核心认知：

1. 把会话拆成独立的、清晰的、可执行的任务。不要试图一次"画完猫头鹰"。
2. 对模糊请求，把工作分成"规划"和"执行"两个独立会话。
3. 如果给 Agent 一个验证自己工作的方法，它在大多数情况下能自己修复错误。

同等重要的是**负空间**：理解什么时候不该用 Agent。把 Agent 用在它大概率失败的事情上，是巨大的时间浪费。

### Step 3: 下班前 Agent（End-of-Day Agents）

每天最后 30 分钟启动 Agent。核心假设：在你无法工作的时间里获得增量进展。

有效的类别：
- **深度研究**：让 Agent 调研某个领域的所有库，产出多页摘要
- **并行探索**：让多个 Agent 尝试你有但还没时间开始的模糊想法
- **Issue/PR 分流**：Agent 不回复，只产出报告，第二天给你"热启动"

### Step 4: 外包确定性任务（Outsource the Slam Dunks）

把 Agent 高确信度能搞定的任务在后台运行，自己做另一件事。关键细节：**关掉 Agent 的桌面通知。** 上下文切换非常昂贵，要由人类控制何时去检查 Agent，而不是被 Agent 打断。

### Step 5: 工程化 Harness（Engineer the Harness）⭐

这是核心阶段。两种形式：

**形式一：隐式提示（AGENTS.md）**

对于简单问题——比如 Agent 反复运行错误的命令或找错 API——更新 AGENTS.md。Mitchell 在 Ghostty 项目中维护的 AGENTS.md 里，每一行都来自一个过去的 Agent 错误行为。写进去后，几乎完全解决了对应问题。

**形式二：编程工具**

比如自动截图的脚本、过滤运行测试的脚本等。通常配合 AGENTS.md 的更新，让 Agent 知道这些工具的存在。

核心心态：**每当 Agent 做了一件坏事，就投入工程努力让它永远不再做。每当 Agent 需要验证它在做好事，就给它工具。** 这像疫苗一样——每次错误变成一条免疫规则，随时间复利。

### Step 6: 永远有一个 Agent 在运行

目标状态：如果没有 Agent 在跑，就问自己"现在有什么任务可以委托给 Agent？" 当前 Mitchell 大约 10-20% 的工作时间有后台 Agent 在运行。

关键态度：**不为跑 Agent 而跑 Agent。** 只在有真正有帮助的任务时才启动。

---

## 六、Martin Fowler / Böckeler 的批判性视角

Birgitta Böckeler 的分析带来了几个 OpenAI 文章中缺失的关键思考：

### 6.1 功能性验证的缺失

OpenAI 描述的所有措施都聚焦于**长期内部质量和可维护性**。但 Böckeler 指出，文章中**缺少对功能和行为的验证**——也就是说，代码维护得很好，但你怎么知道它做的是对的事？

这是一个重要的 gap。

### 6.2 Harness 会成为新的"服务模板"吗？

大多数组织只有两三个主要技术栈。Böckeler 设想了一个未来：团队从一组预建的 Harness 中选择，针对常见应用拓扑开箱即用，然后随着时间推移根据应用特性微调。

但这也引出了经典的 Fork 同步问题：团队定制了 Harness 后，如何把改进回馈给共享模板？

### 6.3 运行时约束与自主性的悖论

早期的 AI 编码炒作假设 LLM 会给我们无限的运行时灵活性——用任何语言、任何模式生成代码，LLM 会搞定一切。但 OpenAI 的实践证明，**增加信任和可靠性需要约束解决方案空间**：特定架构模式、强制边界、标准化结构。

这意味着放弃一些"生成任何东西"的灵活性，换取 prompt、规则和充满技术细节的 Harness。

### 6.4 两个世界：AI 前 vs AI 后

如果我们开发出好的 Harness 技术，哪些可以应用到现有应用？哪些只适用于从零开始就有 Harness 的应用？

对老代码库，改造 Harness 是否值得？就像在一个从未用过静态分析工具的代码库上首次运行——你会被告警淹没。

这预示着 **双轨制架构** 的出现：AI 原生应用从第一行代码就融入 Harness，享受 AI 自动维护红利；传统应用在系统熵增中举步维艰。

---

## 七、Harness 的六大组件（parallel.ai 框架）

parallel.ai 团队识别的六个核心组件与 OpenAI 和 Anthropic 发布的内容高度一致：

### 7.1 工具集成层（Tool Integration Layer）

通过定义好的协议（如 MCP）把模型连接到外部 API、数据库、代码执行环境和自定义工具。

关键设计原则：**构建原子化的、健壮的工具。** 每个工具做好一件事，有清晰的输入输出契约。让模型来决定何时、如何组合这些工具。

### 7.2 记忆与状态管理（Memory and State Management）

多层记忆架构：

- **工作上下文（Working Context）**：当前任务的即时信息
- **会话状态（Session State）**：当前工作会话中积累的中间结果
- **长期记忆（Long-term Memory）**：跨会话持久化的知识

Anthropic 的方案使用进度文件（progress files）和 Git 历史来桥接会话。OpenAI 发现用 JSON 做特性跟踪比 Markdown 更好——因为 Agent 不太会随意编辑或覆盖结构化数据。

这本质上解决的问题是：**每个新的 Agent 会话启动时对之前发生的事情毫无记忆。** 进度文件让新 Agent 快速理解工作状态，就像工程师之间的交班报告。

### 7.3 上下文工程与提示管理（Context Engineering & Prompt Management）

已在第四章深度展开。核心要点重申：**不是静态提示模板，而是基于当前任务状态的主动上下文选择。**

### 7.4 规划与分解（Planning and Decomposition）

引导模型通过结构化任务序列执行，而不是试图一次搞定所有事。

实践中，这意味着：
- 大目标 → 小构建块（设计、编码、审查、测试）
- 先让 Agent 构建这些构建块
- 再用它们解锁更复杂的任务
- **深度优先**，而不是广度优先

### 7.5 验证与护栏（Verification and Guardrails）

格式验证、安全过滤、自我修正循环。当 Agent 挣扎时，Harness 把它当作一个信号——去识别**缺少什么**，而不是要求 Agent"再努力一点"。

每一次 Agent 失败都是一个正向信号：失败点被反馈回系统，触发自动编写修复代码，实现系统的自我进化。

### 7.6 模块化与可扩展性（Modularity and Extensibility）

可插拔组件，可以独立启用、禁用或替换。

这回到了"为删除而建"的原则：今天你写的复杂控制逻辑，明天可能被一个更强模型的 prompt 取代。如果你的架构不允许你轻松移除组件，你就被自己的工程决策锁死了。

---

## 八、实战中的 Harness：三种模式

### 8.1 OpenAI 模式：零人工代码

- 5-7 名工程师，5 个月
- 平均每人每天合并 3.5 个 Pull Request
- 代码审查通过"Agent 对 Agent"闭环自动完成
- 工程师的工作 100% 转向环境设计和反馈

### 8.2 Stripe 模式：Minions 规模化

- 内部编码 Agent 每周合并超过 1000 个 PR
- 开发者在 Slack 发布任务 → Minion 写代码 → 通过 CI → 开 PR → 人类审查合并
- Agent 在隔离的、预热的 "devbox" 中运行——与人类工程师相同的开发环境，但与生产和互联网隔离
- 通过 MCP 服务器访问 400+ 内部工具

### 8.3 Anthropic 模式：Markdown/Prompt Harness

Claude Code 是一个 Harness。它读取整个代码库，管理文件系统访问，生成子 Agent，处理工具编排，维护跨会话记忆，并实现护栏。

CLAUDE.md / Skills 架构代表了另一种 Harness 形态：**把编排指令直接嵌入系统提示或结构化 Markdown 文件中。LLM 自身成为循环控制器——它读取 Harness 规则并遵循它们。**

这种方式适合：LLM 能力足够强，能自我引导；你想快速迭代，不想改代码。

---

## 九、从 2023 到 2026：认知的三级跳

| 阶段 | 关注点 | 核心问题 |
|------|--------|---------|
| Prompt Engineering (2023) | "怎么说" | 如何写出让模型给出好回答的提示词？ |
| Context Engineering (2025) | "知道什么" | 如何确保模型有正确的信息来做决策？ |
| **Harness Engineering (2026)** | **"在什么环境下做事"** | **如何构建让 Agent 长时稳定运行的完整系统？** |

这标志着 AI 开发已从"面向模型"彻底转向"面向系统"。竞争不再是智商较量，而是工程化能力的较量。

---

## 十、对 Recomby 的启示

基于以上所有理解，Harness Engineering 与 Recomby 当前架构有直接映射关系：

**你已经在做 Harness Engineering**——只是可能还没用这个名字。CLAUDE.md 作为编排大脑、file-based memory system、四层 API 计费控制、PreCompact hook for crash recovery、Skills 的 IP 保护——这些都是 Harness 的组件。

几个值得思考的方向：

1. **上下文工程的深化**：当前的 CLAUDE.md + reference files 的渐进披露架构（progressive disclosure architecture）已经是一种上下文工程实践。下一步可以考虑动态上下文注入——根据当前执行的 Skill（geo-miner vs geo-writer），动态加载不同的上下文片段，而不是让 Agent 面对所有信息。

2. **架构约束的机械化执行**：目前的 setReadonlyRecursive() 是一种约束。可以考虑增加更多确定性检查——比如 Skill 输出格式验证、API 调用频率的自动限制、跨 Skill 依赖方向的强制执行。

3. **熵管理**：随着客户数量增长，memory/ 目录中的数据会积累。定期运行清理 Agent 保持数据一致性和质量会变得越来越重要。

4. **"为删除而建"的心态**：模型能力在快速迭代。今天在 Skill 指令中精心设计的某个工作流步骤，可能在下一代模型中可以用一句话替代。保持架构可拆卸性。

---

## 参考资料

- Mitchell Hashimoto, "My AI Adoption Journey" (2026-02-05): https://mitchellh.com/writing/my-ai-adoption-journey
- Birgitta Böckeler / Martin Fowler, "Harness Engineering" (2026-02-17): https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html
- OpenAI, "Harness engineering: leveraging Codex in an agent-first world" (2026-02): https://openai.com/index/harness-engineering/
- Cobus Greyling, "The Rise of AI Harness Engineering" (2026-03-13)
- Birgitta Böckeler / Martin Fowler, "Humans and Agents in Software Engineering Loops" (2026-03-04)
- NxCode, "Harness Engineering: The Complete Guide" (2026-03-01)
