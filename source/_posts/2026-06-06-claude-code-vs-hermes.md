---
title: "Claude Code vs Hermes：两个AI Agent的架构哲学对比"
date: 2026-06-06 15:30:00
tags:
  - AI Agent
  - Claude Code
  - Hermes
  - 架构分析
  - 技术学习
categories: 技术学习
---

## 为什么要聊这个对比

2026 年刚过一半，AI Agent 赛道已经卷得不行。Anthropic 的 Claude Code、OpenAI 的 Codex、Nous Research 的 Hermes Agent、开源社区的 OpenClaw——四款终端 Agent 工具在同一时间段内集中爆发，不是巧合。

根本原因是同一件事：**大模型上下文窗口突破 200K 之后，Agent 从「 Demo 阶段」进入了「工程化阶段」**。窗口够大了，工具调用稳定了，该比的就是架构设计了。

Claude Code 和 Hermes 经常被放在一起比，但说实话它们根本不是同类产品。一个是从内部长出来的编程特化 Agent，一个是从研究项目演化出来的通用自进化框架。把它们放在一起聊，价值不在于「选哪个」，而在于理解**两种完全不同的 Agent 设计哲学**。

---

## 一句话定位

> **Claude Code**：Anthropic 官方出品，深度绑定 Claude 模型，专为编程场景端到端优化的终端 Agent。
>
> **Hermes Agent**：Nous Research 开源发布，模型无关，带三层记忆和自进化 Skills 的通用 AI 数字主脑。

---

## Claude Code 架构深潜

2026 年 4 月，arXiv 上出现了一篇有趣的论文——*Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems*（arxiv 2604.14228）。作者团队直接分析了 Claude Code v2.1.88 的源码：**约 1,900 个 TypeScript 文件，总计约 512K 行代码**。

论文最有价值的洞察是：Claude Code 的架构是由**五项核心价值观**驱动的，这些价值观进一步细化为 13 条设计原则，最终映射到具体的代码实现。

### 五大核心价值观

| 价值观 | 具体体现 |
|--------|----------|
| **人类决策权威** | 关键操作必须人类确认，不自动执行破坏性操作 |
| **安全与安保** | 基于 ML 的分类器判断操作风险，7 种权限模式 |
| **可靠执行** | 工具调用结果可预测，错误可恢复 |
| **能力放大** | 不是替代程序员，而是扩展能力边界 |
| **上下文适应性** | 灵活应对不同项目规模和场景 |

### 核心架构：简单的 while 循环

```
while (true) {
  ① 压缩上下文（五层管道）
  ② 调用 Claude API（带完整工具描述）
  ③ 解析模型返回（工具调用 / 文本回复）
  ④ 执行工具（Shell / 文件编辑 / MCP 工具）
  ⑤ 将结果注入上下文
  ⑥ 等待用户确认（如需要）
}
```

论文指出：**核心循环极其简单，但绝大部分代码存在于围绕这个循环的周边系统**。安全层、上下文压缩、权限管理、MCP 协调——这些「周边系统」才是 Claude Code 的真正壁垒。

### 五层上下文压缩管道

这是 Claude Code 工程化最核心的部分。当对话变长，上下文窗口迟早会满，如何压缩是关键：

1. **自动摘要**：对较早的对话轮次生成摘要替代原文
2. **工具结果截断**：超长工具返回结果只保留关键信息
3. **文件内容懒加载**：只在需要时读取完整文件内容
4. **会话状态快照**：将已完成子任务的状态固化为快照
5. **用户确认点压缩**：用户已确认的操作从历史中移除

这五层管道让 Claude Code 能在超长编程会话中保持上下文不超限。Hermes 目前没有同等复杂的上下文管理方案。

### 扩展机制：四种武器

| 机制 | 用途 | 成熟度 |
|------|------|--------|
| **MCP** | 接入外部工具和数据源 | ⭐⭐⭐⭐⭐ |
| **Plugins** | 官方和社区插件 | ⭐⭐⭐⭐ |
| **Skills** | 项目级可复用指令集 | ⭐⭐⭐ |
| **Hooks** | 生命周期钩子，自动化流程 | ⭐⭐⭐⭐ |

---

## Hermes Agent 架构深潜

Hermes Agent 是 Nous Research（Hermes 系列大模型背后的团队）于 2026 年 2 月发布的开源项目，MIT 许可证。发布四个多月，GitHub 斩获 61K+ Stars——这个数在纯 Agent 框架里相当夸张。

它的核心设计目标只有一个：**让 Agent 越用越聪明**。不是通过更大模型，而是通过记忆和技能沉淀。

### 三层记忆系统

这是 Hermes 最本质的架构差异：

```
┌─────────────────────────────────────────────┐
│            Session Memory (会话记忆)          │
│   当前对话上下文，存储在内存，会话结束即丢失      │
│   作用：理解当前任务的连贯性                  │
└─────────────────────────────────────────────┘
                    ↓ 任务完成
┌─────────────────────────────────────────────┐
│         Persistent Memory (持久记忆)          │
│   跨会话存储用户偏好、项目背景、历史操作         │
│   技术实现：FTS5 全文索引 + LLM 摘要         │
│   作用：下次对话时自动召回相关背景              │
└─────────────────────────────────────────────┘
                    ↓ 经验固化
┌─────────────────────────────────────────────┐
│          Skill Memory (技能记忆)              │
│   解决方案模式和方法论，以 SKILL.md 存储       │
│   遵循 agentskills.io 标准，可复用可分享      │
│   作用：把「做过一次的事」变成「永远会做的事」    │
└─────────────────────────────────────────────┘
```

三层记忆的精髓在于：**会话记忆管当下，持久记忆管跨会话连续性，技能记忆管可复用的方法论**。Claude Code 只有第一层（会话内存）和项目级的 `CLAUDE.md`（近似第二层但很粗糙）。

### Skills 自进化：Hermes 的真正杀手锏

传统 Agent 的 Skills 是人工写的 Markdown 文件，写一次就固定了。Hermes 的 Skills 是**活的**：

```
用户：帮我部署这个 Hexo 博客到 GitHub Pages

  ↓ Hermes 执行（调用 git / npm / GitHub API 等工具）
  ↓ 任务完成

  ↓ Hermes 自动反思：这个任务有哪些可复用的步骤？
  ↓ 生成 SKILL.md：hexo-blog-deploy.md
  ↓ 存入技能库（FTS5 索引）

下次用户说：发一篇新博客
  ↓ Hermes 检索技能库 → 找到 hexo-blog-deploy
  ↓ 自动应用该技能的流程
  ↓ 执行过程中根据实际效果调整技能内容（自优化）
```

这套闭环有三个阶段：

1. **自动生成**：任务完成后，LLM 自动提炼可复用步骤，写成 `SKILL.md`
2. **检索增强**：新任务开始时，先从技能库检索相关 Skills，注入上下文
3. **自我优化**：技能在实际使用中被不断修正，形成「越用越准」的飞轮

### 模型无关架构

Claude Code 深度绑定 Anthropic API，换个模型？不可能。Hermes 从第一天就是模型无关的：

```yaml
# ~/.hermes/config.yaml 示例
llm:
  provider: openrouter    # nous / openrouter / custom / local
  model: anthropic/claude-opus-4
  api_key: ${OPENROUTER_API_KEY}

  # 也可以接 30+ 提供商中的任意一个：
  # provider: custom
  # base_url: https://api.deepseek.com/v1
  # model: deepseek-chat
  # api_key: ${DEEPSEEK_API_KEY}
```

支持的接入方式：

| 接入方式 | 说明 | 典型场景 |
|----------|------|----------|
| **Nous Portal** | 原生 OAuth，一键登录 | 快速开始 |
| **OpenRouter** | 一个 API Key 访问 200+ 模型 | 模型切换 / 成本优化 |
| **自定义端点** | 任何 OpenAI 兼容 API | 接国内模型（Kimi/智谱/GLM） |
| **本地 vLLM** | 完全本地，零 API 成本 | 隐私敏感场景 |

硬性要求：模型上下文窗口至少 **64K tokens**，否则多步骤工具调用的 working memory 会撑爆。

### RL 数据飞轮：隐藏最深的杀手锏

这是大部分介绍文章完全没提到的功能。Hermes 每次任务执行的**完整轨迹**（每轮对话、每次工具调用、每个中间结果）都会被记录，可以导出为标准的 RL 训练数据格式：

```bash
# 导出 ShareGPT 格式的训练数据
hermes export --format sharegpt --output training_data.json

# 支持轨迹压缩到指定 token 预算
hermes export --format sharegpt --max-tokens 4096
```

配合 Nous Research 自己的 Atropos 框架，可以直接用这些轨迹数据做 RL 微调：

```python
from atropos import TrajectoryTrainer

trainer = TrajectoryTrainer(
    trajectories="training_data.json",
    model="your-base-hermes-model",
    parsers=11  # 支持 11 种工具调用解析器
)
trainer.train()
```

这个设计的意义：**用 Hermes 产生的数据，反过来微调出更好的 Hermes 系列模型，再用更好的模型跑 Hermes——形成闭环**。Nous Research 在模型训练和 Agent 框架之间打通了数据飞轮，这是 Anthropic 也好、OpenAI 也好，都没有做的一件事。

---

## 架构对比：一张图说清楚

<svg viewBox="0 0 680 520" width="100%" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Claude Code vs Hermes 架构对比">
  <defs>
    <style>
      text { font-family: sans-serif; }
      .title { font-size: 18px; font-weight: bold; }
      .section-title { font-size: 14px; font-weight: bold; }
      .body { font-size: 12px; }
      .caption { font-size: 11px; fill: #666; }
    </style>
  </defs>

  <!-- Background -->
  <rect width="680" height="520" fill="#FAFAFA" rx="8"/>

  <!-- Title -->
  <text x="340" y="30" text-anchor="middle" class="title" fill="#222">Claude Code vs Hermes：架构对比</text>

  <!-- === LEFT: Claude Code === -->
  <text x="20" y="60" class="section-title" fill="#D97706">Claude Code（Anthropic）</text>

  <!-- Core Loop -->
  <rect x="20" y="75" width="300" height="60" fill="#FEF3C7" rx="6" stroke="#D97706" stroke-width="1.5"/>
  <text x="170" y="95" text-anchor="middle" class="body" fill="#92400E" font-weight="bold">While 循环核心</text>
  <text x="170" y="112" text-anchor="middle" class="caption">调用模型 → 解析工具调用 → 执行 → 压缩上下文 → 循环</text>
  <text x="170" y="128" text-anchor="middle" class="caption">~1,900 TS 文件 / ~512K 行代码</text>

  <!-- Context Compression -->
  <rect x="20" y="155" width="300" height="75" fill="#EFF6FF" rx="6" stroke="#3B82F6" stroke-width="1.5"/>
  <text x="170" y="175" text-anchor="middle" class="body" fill="#1E40AF" font-weight="bold">五层上下文压缩管道</text>
  <text x="30" y="193" class="caption">① 自动摘要较早对话轮次</text>
  <text x="30" y="209" class="caption">② 工具结果截断</text>
  <text x="170" y="209" text-anchor="middle" class="caption">③ 文件内容懒加载</text>
  <text x="30" y="225" class="caption">④ 会话状态快照</text>
  <text x="170" y="225" text-anchor="middle" class="caption">⑤ 用户确认点压缩</text>

  <!-- Permission System -->
  <rect x="20" y="250" width="300" height="50" fill="#F0FDF4" rx="6" stroke="#16A34A" stroke-width="1.5"/>
  <text x="170" y="270" text-anchor="middle" class="body" fill="#15803D" font-weight="bold">七种权限模式 + ML 分类器</text>
  <text x="170" y="287" text-anchor="middle" class="caption">关键操作强制人类确认，安全边界由 ML 判定</text>

  <!-- Extension Mechanisms -->
  <rect x="20" y="315" width="300" height="65" fill="#FAF5FF" rx="6" stroke="#9333EA" stroke-width="1.5"/>
  <text x="170" y="335" text-anchor="middle" class="body" fill="#6B21A8" font-weight="bold">四种扩展机制</text>
  <text x="30" y="353" class="caption">MCP（工具接入）| Plugins（官方插件）</text>
  <text x="30" y="369" class="caption">Skills（项目指令）| Hooks（生命周期钩子）</text>

  <!-- Model Binding -->
  <rect x="20" y="395" width="300" height="45" fill="#FEF2F2" rx="6" stroke="#DC2626" stroke-width="1.5"/>
  <text x="170" y="415" text-anchor="middle" class="body" fill="#991B1B" font-weight="bold">⚠️ 模型绑定：仅 Claude</text>
  <text x="170" y="432" text-anchor="middle" class="caption">Opus / Sonnet / Haiku，无法切换其他模型</text>

  <!-- Deployment -->
  <rect x="20" y="455" width="300" height="40" fill="#FFFBEB" rx="6" stroke="#D97706" stroke-width="1.5"/>
  <text x="170" y="475" text-anchor="middle" class="body" fill="#92400E" font-weight="bold">部署：终端 CLI + IDE 插件</text>
  <text x="170" y="492" text-anchor="middle" class="caption">VS Code / JetBrains / 桌面 App</text>

  <!-- === RIGHT: Hermes Agent === -->
  <text x="360" y="60" class="section-title" fill="#7C3AED">Hermes Agent（Nous Research）</text>

  <!-- Core: 3-Layer Memory -->
  <rect x="360" y="75" width="300" height="90" fill="#F5F3FF" rx="6" stroke="#7C3AED" stroke-width="1.5"/>
  <text x="510" y="95" text-anchor="middle" class="body" fill="#5B21B6" font-weight="bold">三层记忆系统（核心差异）</text>
  <rect x="370" y="105" width="280" height="18" fill="#EDE9FE" rx="3"/>
  <text x="380" y="118" class="caption" fill="#5B21B6">L1 会话记忆：内存，管当下</text>
  <rect x="370" y="126" width="280" height="18" fill="#EDE9FE" rx="3"/>
  <text x="380" y="139" class="caption" fill="#5B21B6">L2 持久记忆：FTS5 + LLM 摘要，管跨会话</text>
  <rect x="370" y="147" width="280" height="18" fill="#EDE9FE" rx="3"/>
  <text x="380" y="160" class="caption" fill="#5B21B6">L3 技能记忆：SKILL.md，管可复用方法论</text>

  <!-- Self-Evolving Skills -->
  <rect x="360" y="180" width="300" height="60" fill="#ECFDF5" rx="6" stroke="#059669" stroke-width="1.5"/>
  <text x="510" y="200" text-anchor="middle" class="body" fill="#065F46" font-weight="bold">Skills 自进化闭环</text>
  <text x="510" y="217" text-anchor="middle" class="caption">自动生成 → 检索增强 → 自我优化</text>
  <text x="510" y="233" text-anchor="middle" class="caption">遵循 agentskills.io 标准，可分享</text>

  <!-- Model Agnostic -->
  <rect x="360" y="250" width="300" height="50" fill="#F0FDF4" rx="6" stroke="#16A34A" stroke-width="1.5"/>
  <text x="510" y="270" text-anchor="middle" class="body" fill="#15803D" font-weight="bold">✅ 模型无关：30+ 提供商</text>
  <text x="510" y="287" text-anchor="middle" class="caption">OpenRouter(200+模型) / 自定义 / 本地 vLLM</text>

  <!-- RL Data Flywheel -->
  <rect x="360" y="315" width="300" height="65" fill="#FFF7ED" rx="6" stroke="#EA580C" stroke-width="1.5"/>
  <text x="510" y="335" text-anchor="middle" class="body" fill="#9A3412" font-weight="bold">RL 数据飞轮（隐藏杀手锏）</text>
  <text x="370" y="353" class="caption">执行轨迹 → ShareGPT 格式导出</text>
  <text x="370" y="369" class="caption">→ Atropos 框架 RL 微调 → 更好的模型 → 更强的 Hermes</text>

  <!-- Multi-Platform -->
  <rect x="360" y="395" width="300" height="45" fill="#EFF6FF" rx="6" stroke="#3B82F6" stroke-width="1.5"/>
  <text x="510" y="415" text-anchor="middle" class="body" fill="#1E40AF" font-weight="bold">多平台网关：15+ 消息平台</text>
  <text x="510" y="432" text-anchor="middle" class="caption">Telegram / Discord / Slack / 微信 / 飞书</text>

  <!-- Deployment -->
  <rect x="360" y="455" width="300" height="40" fill="#FFFBEB" rx="6" stroke="#D97706" stroke-width="1.5"/>
  <text x="510" y="475" text-anchor="middle" class="body" fill="#92400E" font-weight="bold">部署：服务器常驻 + 多平台</text>
  <text x="510" y="492" text-anchor="middle" class="caption">支持 Docker / Modal / Daytona / 本地</text>

  <!-- Bottom note -->
  <text x="340" y="510" text-anchor="middle" class="caption">数据来源：arxiv 2604.14228 | Hermes Agent 官方文档 | 实测体验</text>
</svg>

---

## 编程能力：残酷的真相

先把话说明白：**Hermes 的编程能力不如 Claude Code，这是事实，不是偏见**。

但这件事的原因值得拆解，不是简单一句「Hermes 不行」能概括的。

### 原因一：模型质量的差异

Hermes 的编程能力上限 = 你挂载的底层模型的质量上限。你接 Claude Opus 4，编程能力就是 Opus 4 的水平；接 DeepSeek V3，就是 DeepSeek V3 的水平。

Claude Code 的编程能力上限 = Anthropic 对 Claude 模型的**深度编排**。Anthropic 在 Claude Code 里对工具调用格式、上下文注入方式、错误恢复策略做了大量专有优化，这些优化是跟 Claude 模型本身耦合的，别的模型复制不了。

打个比方：Claude Code 是原厂调校的赛车，Hermes 是你可以随意换发动机的改装车架子。架子本身不错，但原厂调校的赛道表现就是更好。

### 原因二：上下文管理的工程深度

前面提到的「五层上下文压缩管道」，Claude Code 这是真刀真枪的工程积累。Hermes 的记忆系统擅长跨会话持久化，但在**单次超长编程会话的上下文管理**上，目前没有同等复杂的方案。

这意味着：你要让 Agent 连续工作几小时、涉及几十个文件的大型重构任务，Claude Code 更不容易「失忆」或产出垃圾输出。

### 原因三：工具调用的稳定性

Claude Code 的工具调用格式是 Anthropic 自己定义的，跟 Claude 模型训练时对齐得很好。Hermes 要兼容 30+ 提供商的各种工具调用格式（OpenAI 格式、Anthropic 格式、Cohere 格式……），客观上更难做到每个模型都完美。

---

## 那 Hermes 的优势到底在哪？

既然编程能力打不过，Hermes 凭什么活下来而且还火了？

### 优势一：自进化 Skills（Claude Code 完全没有的东西）

这是最根本的差异。Claude Code 每次执行任务都是从零开始（除了 `CLAUDE.md` 里的项目约定），Hermes 每次执行任务都会**变得更好一点**。

具体场景：你让 Hermes 帮你搭一个 Redis MCP 服务器，它搞定了。下次你说「帮我搭一个 MySQL MCP 服务器」，Hermes 会从技能库里召回上次的 Redis 搭建经验，知道哪些坑要避免、哪些步骤可以复用。Claude Code 不会，它每次都是从零学。

### 优势二：持久记忆（跨会话不「失忆」）

Claude Code 的上下文在会话结束后就没了（存在本地文件里但下次不会自动召回），Hermes 的持久记忆层会在下次对话时**自动召回相关背景**。

比如你两个月前跟 Hermes 聊过你们公司的部署规范，下次打开它还记得。Claude Code 不会，除非你手动把规范写进 `CLAUDE.md`。

### 优势三：定时任务和主动式工作

Claude Code 是「你叫它它才动」的被动工具。Hermes 内置 cron 调度器，可以**主动干活**：

```bash
# 让 Hermes 每天早上 8 点自动生成日报
hermes cron add "0 8 * * *" "生成今日科技早报，发送到 Telegram"
```

Claude Code 要做同样的事，需要外面的 cron + 脚本调用，它自己不会「醒来」。

### 优势四：模型无关 = 成本可控

Claude Opus 4 的 API 成本不是开玩笑的。Hermes 可以随时切换到便宜的模型干简单活，贵的模型只用在真正需要的地方。还有 Credential Pool（多 API Key 轮换），一个 Key 用完了自动换下一个。

---

## 批评：两个系统各自的问题

不搞「两边讨好」的写法，分别说问题。

### Claude Code 的问题

1. **模型绑定太死**：你不喜欢 Anthropic 的数据政策？不喜欢 Claude 的回复风格？没得选。这是商业产品的本质，不是 Bug，但确实是限制。
2. **记忆系统太弱**：`CLAUDE.md` 是静态文件，不会自己进化。跨会话的上下文连续性基本靠用户手动维护。
3. **不能主动工作**：没有内置调度，不能做定时任务，不能 24/7 常驻巡检。
4. **黑盒**：闭源，你不知道它到底怎么处理你的代码的，数据有没有上传到 Anthropic 的服务器（官方说不会，但你只能信）。

### Hermes 的问题

1. **编程语言是 Python**：不是说 Python 不好，而是 TypeScript/JavaScript 对 Agent 框架来说更适合做工具生态。这也是为什么 MCP 生态里 TypeScript SDK 最成熟。
2. **记忆系统的语义检索不够准**：依赖 FTS5 全文搜索，不是向量检索，语义理解偶有偏差。官方也承认这点，建议用户用 `hermes memory` 命令手动管理。
3. **Skills 自进化还比较早期**：自动生成的 SKILL.md 质量不稳定，有时候生成的是废话，反而污染技能库。需要用户干预修剪。
4. **编程专项优化不足**：前面说过了，复杂编程任务的表现不如 Claude Code。

---

## 结论：不是二选一，是组合拳

把两个工具放在同一个擂台上比「哪个更好」本身就是错误的问题。正确的问题是「哪个更适合我当前的任务」。

| 场景 | 推荐工具 | 理由 |
|------|----------|------|
| 复杂代码库重构、Debug | **Claude Code** | 代码理解深度、工具调用稳定性完胜 |
| 需要跨会话记忆的个人助理 | **Hermes** | 三层记忆 + 自进化 Skills |
| 定时任务、自动巡检 | **Hermes** | 内置 cron，Claude Code 做不到 |
| 成本敏感，需要灵活切换模型 | **Hermes** | 模型无关，可以用便宜模型干简单活 |
| 企业环境，需要稳定官方支持 | **Claude Code** | Anthropic 背书，生态成熟 |
| 隐私敏感，不能把代码发到云端 | **Hermes（本地模型）** | 可以完全本地部署，零数据外泄 |

我自己的用法：**Claude Code 干编程的脏活累活，Hermes 管自动化流程和长周期任务**。两个一起用，不冲突——CLI 入口分别是 `claude` 和 `hermes`，装在同一个环境里完全没问题。

---

*参考资料：Liu et al. "Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems", arXiv 2604.14228, 2026. Hermes Agent 官方文档及各技术社区实测报告。*
