---
title: '驾驭工程的三种哲学：Claude Code / Hermes / OpenClaw Prompts 工程深度对比'
date: 2026-06-08 10:28:00
tags:
  - AI Agent
  - Prompt Engineering
  - Context Engineering
  - Claude Code
  - Hermes Agent
  - OpenClaw
  - 架构对比
categories:
  - AI Agent
description: 从 Prompt Engineering、Context Engineering、Harness Engineering 三个维度，系统对比 Claude Code、Hermes Agent 和 OpenClaw 的驾驭工程设计哲学。
---

把大模型当 Agent 用，光靠一个粗放的 System Prompt 远远不够。厉害的团队不会只写一段文字丢给模型，而是在模型外面套一层精心设计的"控制引擎"——决定什么信息按什么顺序注入模型、上下文快满时怎么压缩、怎么在模型想越界时兜底拦截。

这套东西有个名字叫 **驾驭工程**（Prompt/Context/Harness Engineering），三个顶流 Agent 项目——**Claude Code、Hermes Agent、OpenClaw**——各有各的玩法。

这篇文章横切三个维度，把它们的做法拆开讲清楚。

---

## 一、统一框架：三层推进模型

三个项目内部结构差异巨大，但从驾驭视角看，它们都遵循同一个三层模型：

```
Harness Engineering（驾驭工程） ← 外部约束与干预
        ↓ 约束与引导
Context Engineering（上下文工程） ← 给 AI 看什么、什么时候看
        ↓ 组装与格式化
Prompt Engineering（提示词工程） ← 怎么说、以什么身份说
```

- **Prompt Engineering** 管的是"基底语言"——AI 以谁的身份、用什么风格、被赋予什么能力
- **Context Engineering** 管的是"信息窗口"——在有限 Token 预算内，把最有价值的信息塞进窗口
- **Harness Engineering** 管的是"边界护栏"——Hook 拦截、沙箱隔离、人在环路，防止 Agent 跑偏

下面逐个剖开来看。

---

## 二、Claude Code：工业化精密控制

Claude Code 是 Anthropic 的旗舰编程 Agent，商业产品，所有设计围绕"可预测、高可靠、规模化部署"展开。

### 2.1 Prompt Engineering：七段式动态组装

Claude Code 的提示词不是一段文字，而是一个**动态组装流水线**：

```typescript
QueryEngine.ask()
  → fetchSystemPromptParts()     // 并行获取三大组件
  → buildEffectiveSystemPrompt() // 优先级决策
  → query()                      // 发送请求
```

三大组件构成：

| 组件 | 来源 | 作用 |
|------|------|------|
| `defaultSystemPrompt` | `constants/prompts.ts` 硬编码 | 基础行为规则 |
| `systemContext` | `context.ts` 运行时采集 | Git 状态、环境信息 |
| `userContext` | `context.ts` 文件解析 | CLAUDE.md + 日期 + 偏好 |

最终的 System Prompt 由 **7 个核心模块** 按固定顺序组装：

```
模块 1: 身份介绍 — 安全边界
模块 2: 系统行为 — 输出规则 / 权限模式 / 安全防护
模块 3: 任务执行 — 编码风格 / 避免过度工程 / 安全编码
模块 4: 操作安全 — 可逆性评估 + 高风险操作确认
模块 5: 工具使用 — 优先专用工具，拒绝降级为 Bash
模块 6: 语气风格 — 简洁 / 无 emoji / 带行号引用
模块 7: 输出效率 — 直奔主题，拒绝废话
```

**动态部分的注入方式**：通过 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 分隔符划分静态/动态边界。动态部分包含当前会话特定的工具启用状态、MCP 服务器指令、语言偏好、输出风格覆盖等。

**KV Cache 优化**：这是 Claude Code 的一个精妙设计——把不常变的内容标记为可缓存，利用 Anthropic API 的 prompt caching 减少重复计费：

```typescript
splitSysPromptPrefix() → [
  { text: "静态内容", cacheScope: 'global' },  // ✅ 全局缓存
  { text: "动态内容", cacheScope: null },      // ❌ 每轮重新计算
]
```

这意味着使用者每次对话的 Token 成本中，静态底座部分完全不重复计费。一个细节，但日积月累下来对大规模部署的意义巨大。

### 2.2 Context Engineering：四级分层上下文 + 三层压缩

**CLAUDE.md 是 Claude Code 上下文系统的基石，采用四级分层加载：**

```
~/.claude/CLAUDE.md          ← 用户全局偏好（所有项目生效）
/project/CLAUDE.md           ← 项目共享规范（提交 Git）
/project/CLAUDE.local.md     ← 个人私有指令（不提交，覆盖项目级）
/project/.claude/rules/*.md  ← 按文件类型细分规则
```

优先级规则简洁："离当前任务越近的指令，优先级越高"——项目级覆盖全局级，本地覆盖项目级。

**上下文压缩是上下文工程最考验功夫的部分。Claude Code 用三层递进式策略：**

| 层级 | 名称 | 触发条件 | 策略 | 成本 |
|------|------|----------|------|------|
| L1 | MicroCompact | 工具输出过长 | 规则截断 + 白名单 | 极低 |
| L2 | Session Memory | Token≥10k + 消息≥5 | 复用已有摘要 | 低 |
| L3 | Full LLM Compact | 前两层不足 | LLM 生成 9 段式结构化摘要 | 高 |

**L3 的 9 段式摘要模板非常关键：**

```
1. Primary Request and Intent     — 你最初要干什么
2. Key Technical Concepts         — 用了什么技术
3. Files and Code Sections        — 动了哪些文件
4. Errors and fixes               — 踩了什么坑、怎么修
5. Problem Solving                — 解决路径
6. All user messages              — 所有用户消息
7. Pending Tasks                  — 还有什么事没做
8. Current Work                   — 当前进展
9. Optional Next Step             — 建议下一步
```

这个模板的精妙之处在于：它不是让 LLM 自由发挥写摘要，而是用结构化的框子约束输出格式，确保每次压缩的关键信息维度一致，不会因为模型随机性丢失核心上下文。

**防幻觉的两个技巧：**
1. 先让模型在 `<analysis>` 标签中隐式推理，再输出 `<summary>`，系统剥离分析部分，只保留摘要
2. 压缩过程中禁止调用任何工具（防止副作用）

**Memdir 结构化记忆系统**：用四类文件实现分层记忆——User（用户偏好）、Feedback（纠错记录）、Project（架构决策）、Reference（代码模式）。记忆库过大时用 Sonnet 做语义相关度判断，最多 5 条，在召回率和成本间取平衡。

### 2.3 Harness Engineering：六大 Agent + 双安全体系

Claude Code 的驾驭层是三家中最"重型"的，专为生产环境设计。

**六大内置 Agent 分工：**

| Agent | 职责 | 关键设计 |
|-------|------|----------|
| General | 万能执行者 | 全 tool + 简洁 Prompt |
| Explore | 代码侦察兵 | 只读 + Haiku 省成本 |
| Plan | 软件架构师 | 继承主模型 + 结构化输出 |
| Verification | 质量检验官 | 红蓝对抗思维："搞崩它"而非"看它能不能跑" |
| Guide | 使用说明书 | 查官方文档 + Haiku |
| Statusline | 终端配置员 | 仅 Read/Edit + Sonnet |

**Verification Agent 的设计尤其值得说**：它不是检查功能"能不能跑"，而是想办法"搞崩它"。预设了 10+ 种 AI 偷懒话术（"看起来没问题"、"大概可以了"），一键拆穿。这个设计思路很少在开源 Agent 项目中看到。

**双安全体系：**

```
Permission Engine（规则层）
├─ Allow：低风险自动放行
├─ Deny：高危操作直接阻断（不允许改成 Allow）
└─ Ask：中风险请求用户确认

Sandbox Isolation（系统层，Linux only）
├─ 文件系统：只读挂载 + 白名单
├─ 网络/进程：独立命名空间
└─ 权限：非 root 运行
```

**Hook 系统**：覆盖 20+ 生命周期事件（PreToolUse / PostToolUse / SessionStart / PreFileEdit 等），支持三种干预——阻断、修正参数、注入反馈消息。全局超时 10 分钟兜底，防止外部脚本拖垮主进程。

**异步生成器主循环**：`async function* queryLoop()` 用 Generator 模式设计，内置三类错误自愈——prompt-too-long 自动触发压缩、max-output-tokens 自动续写、网络波动指数退避重试。

---

## 三、Hermes Agent：缓存友好的声明式配置

Hermes Agent 是 Nous Research 的开源项目，设计哲学和 Claude Code 完全不同——它追求**模型无关**、**前缀缓存友好**、**用户可配置**。

### 3.1 Prompt Engineering：三层缓存架构

Hermes 把系统提示词切成三层，核心目标是让 API 的 **prompt caching 命中率最大化**：

```
第一层 stable（稳定层）— 整个会话基本不变
  ├─ SOUL.md 身份 [永远在第一位]
  ├─ 14 个固定引导模块（工具使用、安全、技能索引...）
  └─ 环境提示

第二层 context（上下文层）— 会话之间可能变化
  ├─ 用户自定义 system_message
  └─ AGENTS.md / HERMES.md 等项目上下文文件

第三层 volatile（易变层）— 每次都变
  ├─ MEMORY.md 记忆快照
  ├─ USER.md 用户画像
  ├─ 时间戳与会话 ID
  └─ 平台特定格式提示
```

**为什么 SOUL.md 必须在第一位？** 两个原因：一是后续的所有工具引导、环境提示都以 SOUL.md 定义的身份为前提；二是不变的内容放最前面可以最大化前缀缓存的稳定性。

**stable 层包含的 14 个部分按序拼接（源码 `system_prompt.py` → `build_system_prompt_parts()`）：**

```
 1. SOUL.md 内容（或 DEFAULT_AGENT_IDENTITY 回退）
 2. 引导用户了解 Hermes 自身配置的指令
 3. 通用任务完成/反虚构引导
 4. 工具感知行为引导（按条件注入：记忆、搜索、技能、Kanban）
 5. 计算机使用引导（macOS）
 6. Nous 订阅提示
 7. 工具使用强制引导
 8. 模型特定操作引导（Google/OpenAI）
 9. 技能系统提示
10. 模型身份覆盖（Alibaba 等特殊提供商）
11. 环境提示（WSL、Termux）
12. Python 工具链探针
13. 活跃配置文件提示
14. 平台特定格式提示（CLI 模式禁止 Markdown）
```

系统提示词在每个会话中**只构建一次**，缓存到 `agent._cached_system_prompt`，只有在上下文压缩事件后才会触发重建。

**和 Claude Code 最大的不同：** Claude Code 在每轮 API 调用时构建 prompt（因为有动态注入），Hermes 追求"一次构建、全程复用"——代价是动态性较弱，但缓存命中率最高。

### 3.2 Context Engineering：身份与项目分离

Hermes 把 Agent 的"身份"和"项目知识"彻底拆开：

| 维度 | SOUL.md | AGENTS.md |
|------|---------|-----------|
| 管理对象 | 身份、语气、沟通风格 | 项目架构、编码规范、工具偏好 |
| 作用域 | 所有项目、所有会话 | 仅当前项目 |
| 文件位置 | `~/.hermes/SOUL.md`（固定） | `./AGENTS.md`（从 cwd 向上遍历） |
| 加载层 | stable 层（永远第一位） | context 层 |

**判断准则很直观**：关闭所有项目，只开空白对话，还希望 Agent 保持这个行为吗？→ 是则放 SOUL.md，否则放 AGENTS.md。

**项目上下文文件优先级链（`build_context_files_prompt()`）：**

```
1. .hermes.md / HERMES.md  → 从 cwd 向上遍历到 git root
2. AGENTS.md               → 仅限 cwd
3. CLAUDE.md               → 仅限 cwd（兼容 Claude Code 迁移）
4. .cursorrules            → 仅限 cwd（兼容 Cursor 迁移）
```

**首次匹配获胜**——只加载一种类型，不会叠加。这避免了 Claude Code 那种"多层级 CLAUDE.md 同时生效导致冲突"的问题。

**Personality Overlay 叠加机制**：SOUL.md 定义"是谁"，`/personality` 命令定义"这次用什么语气"，叠加而非替换。14 个内置人格从 `helpful` 到 `catgirl` 到 `noir` 应有尽有。

**安全扫描与截断：**

```python
def load_soul_md():
    content = soul_path.read_text()
    content = _scan_context_content(content)   # 检测 prompt injection / C2 模式
    content = _truncate_content(content)        # 20,000 字符，保留头尾
    return content
```

检测到威胁不是警告，是**完全阻止**，返回 `[BLOCKED: ...]` 占位符。

**特殊执行模式下的继承规则：**

| 模式 | 继承 SOUL.md | 原因 |
|------|-------------|------|
| Cron 任务 | ✅ 继承 | 定时任务也是用户派出去的 |
| 子代理/委托 | ❌ 不继承，用 `DEFAULT_AGENT_IDENTITY` | 子代理是工具，不应有人格 |
| `HERMES_IGNORE_RULES=1` | ❌ 跳过所有上下文文件 | 调试和隔离测试 |

**容器写入保护**：检测到通过容器路径对 `profiles/*/SOUL.md` 的写入尝试时直接拦截——防止 Agent 自己修改自己的人设约束。这是一个细思恐极但非常有前瞻性的安全设计。

### 3.3 Harness Engineering：轻量但精准

Hermes 的驾驭层比 Claude Code 薄很多，但几处设计很精准：

**Prompt Injection 扫描器**：独创的 scope 分级——`context` scope 检测经典注入模式和角色扮演劫持，`strict` scope 检测 SSH 后门和数据泄露 URL。对 SOUL.md 等上下文文件用 `context` 级（避免误杀正常指令），对用户输入用 `strict` 级。

**Skills 自进化闭环**：自动生成 → 检索增强 → 自我优化，遵循 agentskills.io 标准——这层不是传统 Harness，但起到了类似"行为约束"的效果：把成功的工作流固化成可复用的模板，变相减少了 Agent 的自主发挥空间。

**RL 数据飞轮**：执行轨迹 → ShareGPT 格式导出 → Atropos 框架 RL 微调 → 更好的模型 → 更强的 Hermes。这层设计让 Hermes 不仅是一个 Agent 框架，还是一个数据生成管道——长期来看这是它的隐藏杀手锏。

**记忆系统**：三层记忆（L1 会话/L2 持久 FTS5 + LLM 摘要/L3 技能 SKILL.md）。L2 采用了 FTS5 全文检索 + LLM 摘要的混合召回，比 Claude Code 的纯文件系统方案在检索精度上更优。

---

## 四、OpenClaw：模块化与渐进式披露

OpenClaw 是一个自托管的 AI 助手网关，设计哲学是"把所有东西都变成 Markdown 文件，让用户和工具都能操作它"。

### 4.1 Prompt Engineering：23 模块装配线

OpenClaw 的提示词由 `buildAgentSystemPrompt()` 函数（`src/agents/system-prompt.ts`）组装，按固定顺序动态拼接 **23 个模块**。

**三种模式裁剪：**

| 模式 | 适用场景 | 加载策略 |
|------|----------|----------|
| full | 主 Agent 与用户直接对话 | 全部 23 个模块 |
| minimal | 子 Agent 执行独立任务 | 只保留核心模块 |
| none | 极简场景 | 基本只有一行身份 |

**23 个模块的完整结构（full 模式）：**

```
模块 1:  身份标识 [永远存在] — "You are OpenClaw, a personal AI assistant."
模块 2:  工具清单 [full/minimal] — 列出所有可用工具
模块 3:  工具调用风格 [full] — 简单任务直接调工具不解释
模块 4:  安全准则 [full] — Safety Guidelines：服从人类、不越权
模块 5:  CLI 操作指令 [full] — /status, /new, /compact 等
模块 6:  Agent Skills [有条件] — 渐进式披露：先列名字+描述，按需读 SKILL.md
模块 7:  记忆召回 [有条件] — 有搜索工具时才加载
模块 8:  自更新管理 [有条件]
模块 9:  模型别名 [有条件]
模块 10: 工作区信息
模块 11: 参考文档
模块 12: 沙箱信息 [有条件]
模块 13: 授权发送者 [有条件] — 用户身份哈希处理
模块 14: 时间信息
模块 15: Workspace 文件注入 — AGENTS.md/SOUL.md/USER.md/IDENTITY.md/TOOLS.md
模块 16: 回复标签 — [[reply_to_current]] 实现引用回复
模块 17: 消息系统 — 跨 Session、子 Agent 编排指令
模块 18: 语音合成 [有条件]
模块 19: 群聊回复 — 表情反应 vs 文字回复的决策逻辑
模块 20: 推理格式 [有条件]
模块 21: 静默回复 — 后台任务完成时输出 [SILENT]
模块 22: 心跳机制 — HEARTBEAT_OK
模块 23: 运行时信息 [永远存在] — agentId/host/os/model/shell/channel
```

**Markdown 文件驱动的注入机制**——这是 OpenClaw 最独特的设计：

| 文件 | 角色 | 说明 |
|------|------|------|
| AGENTS.md | 总纲 | 核心规范要求、根本目标、交互原则 |
| SOUL.md | 灵魂 | 人格特质、性格倾向、说话风格 |
| IDENTITY.md | 身份证 | 名字、类型、头像风格 |
| USER.md | 主人档案 | 用户偏好、厌恶、习惯 |
| TOOLS.md | 工具清单 | 当前环境可用工具及说明 |
| HEARTBEAT.md | 心跳任务 | 定时任务逻辑 |
| BOOTSTRAP.md | 出生证明 | 首次启动后自动删除 |
| BOOT.md | 启动文件 | 配合 Hook 在启动时运行 |
| MEMORY.md | 长期记忆 | 跨会话持久记忆 |

**特别机制**：如果 OpenClaw 要修改 SOUL.md，会主动通知用户——这点和 Hermes 的容器写入保护异曲同工，都是在防止 AI 自我修改人设。

**极简主义措辞**：能用一个词绝不用句子——比如群聊回复策略用 `Quality > quantity` 两个词搞定，比写一段几百字的规则指令大幅节省 Context Window 空间。

### 4.2 Context Engineering：自适应压缩 + 双层记忆

**Skills 渐进式披露**——这是 Anthropic 提出的理念在 OpenClaw 中的最佳实践：

```
初始注入：只列出技能名称和一句话描述
用户触发某技能 → 系统读取完整的 SKILL.md → 注入到当前上下文
任务完成后 → 技能详细内容从上下文中清除
```

这样一来，Agent 有几十个技能也不会撑爆 Context Window——不用的技能只占一行空间。

**上下文压缩的工程实现非常扎实（`src/agents/compaction.ts`）：**

```
关键常量:
  BASE_CHUNK_RATIO  = 0.4    → 每块占上下文 40%
  SAFETY_MARGIN     = 1.2    → 20% 安全缓冲
  SUMMARIZATION_OVERHEAD = 4096 → 摘要指令预留 Token
```

**三层降级策略：**

```
第一招: summarizeInStages()
  ├── 消息少 → 走兜底 summarizeWithFallback()
  └── 消息多 → 按 Token 比例分割 → summarizeChunks()

第二招: summarizeChunks()
  ├── 处理单个消息块
  ├── 最多 3 次重试
  └── 生成分块摘要后合并

第三招: summarizeWithFallback()
  ├── 先尝试完整摘要
  ├── 失败 → 排除过大消息后再试
  └── 还失败 → 返回 "No prior history."
```

自适应分块：消息越多、每块越大，小消息可以多装几个，大消息则少装。不是粗暴定死"每 10 条压一次"，而是根据实际 Token 数动态调整。

**摘要保留规则**很重要：当前活跃的任务、重要的决策和结论、待办事项、做过的承诺、**所有不透明标识符**（UUID、哈希值）必须原文保留，不得修改。

**双层记忆系统：**

| 特性 | MEMORY.md | memory/日期.md |
|------|-----------|----------------|
| 文件数量 | 只有一个 | 每天一个 |
| 写入方式 | 整理后覆盖/编辑 | 追加写入 |
| 内容类型 | 持久的事实和偏好 | 每日上下文笔记 |
| 注入方式 | 每次对话自动注入 | 只通过搜索访问 |
| 时间衰减 | 不衰减 | `e^(-λ × 天数)`，半衰期 30 天 |

时间衰减的设计很优雅：1 天前的记忆权重 0.977，30 天前 0.500，90 天前只剩 0.125——模拟人的自然遗忘曲线，避免老旧信息占据检索结果的头部位置。

### 4.3 Harness Engineering：全生命周期 Hook

OpenClaw 的驾驭层设计介于 Claude Code（重）和 Hermes（轻）之间：

**全生命周期 Hook：**

| Hook | 触发时机 | 典型用途 |
|------|----------|----------|
| `before_prompt_build` | 构建提示词之前 | 注入额外上下文 |
| `before_tool_call` | 执行工具之前 | 拦截/修改工具参数 |
| `after_tool_call` | 工具执行之后 | 处理工具结果 |
| `before_compaction` | 上下文压缩之前 | 观察压缩过程 |
| `after_compaction` | 上下文压缩之后 | 后处理 |
| `message_received` | 收到消息时 | 消息预处理 |
| `message_sending` | 发送消息前 | 消息后处理 |

**三层安全沙箱：**

```
第一层: 文件系统沙箱  → 限制 Workspace 访问范围
第二层: 命令执行沙箱  → Security/Ask/safeBins 三种模式
第三层: 网络访问沙箱  → 白名单域名 + 防泄露机制
底层:   操作系统最小权限兜底
```

**心跳强制巡检**：HEARTBEAT.md 强制 Agent 定期完成固定任务，防止空闲太久丢失上下文。

---

## 五、三维度全景对比

<svg viewBox="0 0 680 620" width="100%" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Claude Code / Hermes / OpenClaw 驾驭工程三维度对比"><defs><style>text{font-family:sans-serif}.title{font-size:15px;font-weight:bold}.h2{font-size:13px;font-weight:bold}.body{font-size:11px}.cap{font-size:10px;fill:#64748B}.tag-red{font-size:9px;fill:#fff}.tag-blue{font-size:9px;fill:#fff}.tag-green{font-size:9px;fill:#fff}</style></defs><rect width="680" height="620" fill="#FAFAFA" rx="8"/><text x="340" y="26" text-anchor="middle" class="title" fill="#222">驾驭工程三维度全景对比</text><!-- Column headers --><rect x="185" y="38" width="150" height="26" fill="#D97706" rx="4"/><text x="260" y="56" text-anchor="middle" class="body" fill="#fff" font-weight="bold">Claude Code</text><rect x="350" y="38" width="150" height="26" fill="#7C3AED" rx="4"/><text x="425" y="56" text-anchor="middle" class="body" fill="#fff" font-weight="bold">Hermes Agent</text><rect x="515" y="38" width="150" height="26" fill="#3B82F6" rx="4"/><text x="590" y="56" text-anchor="middle" class="body" fill="#fff" font-weight="bold">OpenClaw</text><!-- Section: Prompt Engineering --><rect x="8" y="74" width="162" height="24" fill="#FEF3C7" rx="4"/><text x="89" y="90" text-anchor="middle" class="h2" fill="#92400E">Prompt Engineering</text><!-- Row 1: Identity --><text x="12" y="118" class="cap">身份定义</text><rect x="185" y="106" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="120" text-anchor="middle" class="body" fill="#92400E">7段静态模块</text><text x="260" y="137" text-anchor="middle" class="cap">hardcoded in prompts.ts</text><rect x="350" y="106" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="120" text-anchor="middle" class="body" fill="#5B21B6">SOUL.md 驱动</text><text x="425" y="137" text-anchor="middle" class="cap">14内置人格可选</text><rect x="515" y="106" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="120" text-anchor="middle" class="body" fill="#1D4ED8">Markdown多文件</text><text x="590" y="137" text-anchor="middle" class="cap">SOUL+IDENTITY+USER+TOOLS</text><!-- Row 2: Assembly --><text x="12" y="168" class="cap">组装策略</text><rect x="185" y="156" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="170" text-anchor="middle" class="body" fill="#92400E">动态装配线</text><text x="260" y="187" text-anchor="middle" class="cap">buildEffectiveSystemPrompt()</text><rect x="350" y="156" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="170" text-anchor="middle" class="body" fill="#5B21B6">会话一次构建</text><text x="425" y="187" text-anchor="middle" class="cap">_cached_system_prompt</text><rect x="515" y="156" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="170" text-anchor="middle" class="body" fill="#1D4ED8">23模块装配</text><text x="590" y="187" text-anchor="middle" class="cap">3种模式 full/minimal/none</text><!-- Row 3: Cache --><text x="12" y="218" class="cap">缓存优化</text><rect x="185" y="206" width="150" height="40" fill="#FEF2F2" rx="4" stroke="#FECACA" stroke-width="0.5"/><text x="260" y="220" text-anchor="middle" class="body" fill="#991B1B">KV Cache标记</text><text x="260" y="237" text-anchor="middle" class="cap">splitSysPromptPrefix()</text><rect x="350" y="206" width="150" height="40" fill="#F0FDF4" rx="4" stroke="#BBF7D0" stroke-width="0.5"/><text x="425" y="220" text-anchor="middle" class="body" fill="#15803D">三层切分天生友好</text><text x="425" y="237" text-anchor="middle" class="cap">stable层放最前面</text><rect x="515" y="206" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="220" text-anchor="middle" class="body" fill="#1D4ED8">无显式缓存优化</text><text x="590" y="237" text-anchor="middle" class="cap">依赖模型通用缓存</text><!-- Section: Context Engineering --><rect x="8" y="258" width="162" height="24" fill="#EFF6FF" rx="4"/><text x="89" y="274" text-anchor="middle" class="h2" fill="#1E40AF">Context Engineering</text><!-- Row 4 --><text x="12" y="302" class="cap">项目配置</text><rect x="185" y="290" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="304" text-anchor="middle" class="body" fill="#92400E">四级分层加载</text><text x="260" y="321" text-anchor="middle" class="cap">全局→项目→本地→规则</text><rect x="350" y="290" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="304" text-anchor="middle" class="body" fill="#5B21B6">优先级链首次匹配</text><text x="425" y="321" text-anchor="middle" class="cap">不叠加，防冲突</text><rect x="515" y="290" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="304" text-anchor="middle" class="body" fill="#1D4ED8">多文件平等注入</text><text x="590" y="321" text-anchor="middle" class="cap">全部注入+条件加载</text><!-- Row 5 --><text x="12" y="352" class="cap">记忆系统</text><rect x="185" y="340" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="354" text-anchor="middle" class="body" fill="#92400E">Memdir四类+语义检索</text><text x="260" y="371" text-anchor="middle" class="cap">Sonnet做相关度判断</text><rect x="350" y="340" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="354" text-anchor="middle" class="body" fill="#5B21B6">三层记忆系统</text><text x="425" y="371" text-anchor="middle" class="cap">L1会话+L2 FTS5+L3技能</text><rect x="515" y="340" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="354" text-anchor="middle" class="body" fill="#1D4ED8">双层记忆+时间衰减</text><text x="590" y="371" text-anchor="middle" class="cap">BM25+向量双路召回</text><!-- Row 6 --><text x="12" y="402" class="cap">上下文压缩</text><rect x="185" y="390" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="404" text-anchor="middle" class="body" fill="#92400E">三层递进式压缩</text><text x="260" y="421" text-anchor="middle" class="cap">L1截断+L2复用+L3 LLM</text><rect x="350" y="390" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="404" text-anchor="middle" class="body" fill="#5B21B6">较简的compact机制</text><text x="425" y="421" text-anchor="middle" class="cap">压缩后重建系统提示词</text><rect x="515" y="390" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="404" text-anchor="middle" class="body" fill="#1D4ED8">自适应分块+三级降级</text><text x="590" y="421" text-anchor="middle" class="cap">动态chunk+20%安全缓冲</text><!-- Section: Harness Engineering --><rect x="8" y="442" width="162" height="24" fill="#F0FDF4" rx="4"/><text x="89" y="458" text-anchor="middle" class="h2" fill="#15803D">Harness Engineering</text><!-- Row 7 --><text x="12" y="486" class="cap">Agent编排</text><rect x="185" y="474" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="488" text-anchor="middle" class="body" fill="#92400E">6大内置Agent</text><text x="260" y="505" text-anchor="middle" class="cap">Verification红蓝对抗</text><rect x="350" y="474" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="488" text-anchor="middle" class="body" fill="#5B21B6">子代理+委托模式</text><text x="425" y="505" text-anchor="middle" class="cap">子代理不继承SOUL.md</text><rect x="515" y="474" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="488" text-anchor="middle" class="body" fill="#1D4ED8">多Agent路由+隔离</text><text x="590" y="505" text-anchor="middle" class="cap">per-session独立workspace</text><!-- Row 8 --><text x="12" y="536" class="cap">安全体系</text><rect x="185" y="524" width="150" height="40" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="538" text-anchor="middle" class="body" fill="#92400E">双安全体系</text><text x="260" y="555" text-anchor="middle" class="cap">Allow/Deny/Ask + Sandbox</text><rect x="350" y="524" width="150" height="40" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="538" text-anchor="middle" class="body" fill="#5B21B6">Prompt注入扫描</text><text x="425" y="555" text-anchor="middle" class="cap">context+strict双scope</text><rect x="515" y="524" width="150" height="40" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="538" text-anchor="middle" class="body" fill="#1D4ED8">三层沙箱+全周期Hook</text><text x="590" y="555" text-anchor="middle" class="cap">文件+命令+网络三层</text><!-- Row 9 --><text x="12" y="586" class="cap">Hook机制</text><rect x="185" y="574" width="150" height="35" fill="#FFFBEB" rx="4" stroke="#FDE68A" stroke-width="0.5"/><text x="260" y="590" text-anchor="middle" class="body" fill="#92400E">20+事件+3种干预</text><text x="260" y="604" text-anchor="middle" class="cap">阻断/修正/注入反馈</text><rect x="350" y="574" width="150" height="35" fill="#F5F3FF" rx="4" stroke="#DDD6FE" stroke-width="0.5"/><text x="425" y="590" text-anchor="middle" class="body" fill="#5B21B6">容器写入保护</text><text x="425" y="604" text-anchor="middle" class="cap">防AI篡改自身SOUL.md</text><rect x="515" y="574" width="150" height="35" fill="#EFF6FF" rx="4" stroke="#BFDBFE" stroke-width="0.5"/><text x="590" y="590" text-anchor="middle" class="body" fill="#1D4ED8">心跳强制巡检</text><text x="590" y="604" text-anchor="middle" class="cap">HEARTBEAT.md定时唤醒</text><!-- Footer --><text x="340" y="618" text-anchor="middle" class="cap">基于源码分析：Claude Code v4.x / Hermes Agent v2.x / OpenClaw v2.x</text></svg>

---

### 核心差异总结

| 维度 | Claude Code | Hermes Agent | OpenClaw |
|------|------------|-------------|----------|
| **Prompt 组装** | 每轮动态装配，7 模块 + 动态边界 | 一次构建全程复用，三层 stable/context/volatile | 23 模块装配线，3 种模式裁剪 |
| **身份定义** | 代码硬编码 + CLAUDE.md 偏好 | 单一的 SOUL.md，固定路径不认 cwd | 多个 Markdown 文件（SOUL+IDENTITY+USER+TOOLS） |
| **缓存策略** | 显式 KV Cache 标记，工程化最优 | 三层切分天生缓存友好，架构级优化 | 无显式缓存优化，依赖模型通用缓存 |
| **上下文压缩** | 三层递进（截断→复用→LLM），9 段式模板 | 较简单的 compact，触发后重建系统提示词 | 自适应分块 + 三级降级，工程最扎实 |
| **记忆系统** | 四类文件 + Sonnet 语义检索（最多 5 条） | 三层记忆 L1/L2/L3，FTS5 全文检索 | 双层 + 时间衰减 + BM25/向量双路召回 |
| **Agent 编排** | 6 个内置 Agent，Verification 红蓝对抗 | 子代理/委托模式，子代理不继承人格 | 多 Agent 路由隔离，per-session 独立 |
| **Hook** | 20+ 事件，阻断/修正/注入三种干预 | 较薄，容器写入保护是亮点 | 全生命周期 7 个节点 + 心跳强制巡检 |
| **安全体系** | 权限引擎（Allow/Deny/Ask）+ Linux 沙箱 | Prompt 注入扫描（双 scope 分级） | 三层沙箱（文件/命令/网络） |
| **模型绑定** | ⚠️ 仅 Claude | ✅ 30+ 提供商 / 本地模型 | ✅ 多模型支持 |

---

## 六、设计哲学总结

三个项目代表了三种截然不同的工程哲学：

**Claude Code = 工业化精密控制**。它的一切设计都是为了大规模部署——7 段式模板是精心打磨过的"最优解"，三层压缩是经过无数次生产事故迭代出的工程方案，Verification Agent 的"红蓝对抗"是为杜绝 AI 幻觉而生的机制。代价是它只能绑定 Claude，是个封闭王国。

**Hermes Agent = 缓存友好的声明式配置**。它的所有设计都围绕一个核心目标：让用户通过 Markdown 文件而非代码来控制 Agent。三层切分、SOUL.md 固定路径、Personality Overlay——这些设计让非技术用户也能"驯服"AI，同时给开发者留了足够的扩展空间。它的弱项在于压缩和 Hook 相对单薄，Harness 层的工程深度不够。

**OpenClaw = 模块化与渐进式披露**。23 个模块像乐高一样按需拼接，Skills 渐进式披露是大规模工具集的终极 Answer，三层自适应压缩的工程实现是三家中最扎实的。但它太依赖 Node.js 生态，对非 JavaScript 开发者的门槛偏高。

---

**一句话结论：**

- 让我选一个**给团队用、要稳定**的 Agent → Claude Code
- 让我选一个**个人用、要灵活**的 Agent → Hermes Agent
- 让我选一个**大规模多工具、要扩展**的 Agent → OpenClaw

三者不是竞争对手，是三套互补的工具箱。厉害的不是选哪一个，是知道什么时候该用哪一套。

---

> **参考来源**
> - arXiv 2604.14228: Claude Code Reverse Engineering
> - [Claude Code Handbook](https://github.com/ThamJiaHe/claude-code-handbook)
> - [Hermes Agent 中文文档](https://hermes-agent.lzw.me/docs/developer-guide/prompt-assembly)
> - [OpenClaw 源码](https://github.com/openclaw/openclaw) / `src/agents/system-prompt.ts` / `src/agents/compaction.ts`
> - [Claude Code CLAUDE.md 完全指南](https://www.wangjun.dev/2026/05/complete-guide-to-claude-code-claude-md/)
> - [Hermes Agent SOUL.md 人格系统](https://cloud.tencent.com/developer/article/2682032)
