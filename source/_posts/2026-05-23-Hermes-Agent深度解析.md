---
title: "Hermes Agent 深度解析：从安装到原理，以及与 OpenClaw 的终极对比"
date: 2026-05-23 20:44:00
tags: [AI, Agent, Hermes, OpenClaw, 开源, 工具对比]
categories: 项目分析
---

> Hermes Agent 是 Nous Research 于 2026 年 2 月开源的「自进化 AI Agent」。截至 2026 年 5 月，GitHub 已突破 **164k Stars**，短短三个月超越了绝大多数 AI 开源项目数年积累。本文从安装部署、常用场景、技术原理、核心架构四个维度全面拆解，并与 OpenClaw 做一次硬碰硬的对比。

---

## 一、项目速览

| 项目信息 | 详情 |
|---------|------|
| **项目名称** | Hermes Agent (hermes-agent) |
| **开发团队** | [Nous Research](https://nousresearch.com)（Hermes、Nomos、Psyche 等知名开源模型的创造者） |
| **开源协议** | MIT |
| **GitHub** | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |
| **Stars** | 164k+ |
| **Forks** | 26.8k+ |
| **Commits** | 9,274+ |
| **最新版本** | v0.14.0（2026.5.16） |
| **技术栈** | Python 88.6% + TypeScript 8.5% |
| **发布周期** | 约一周一个大版本，两周一个小版本 |

一句话定位：**"The agent that grows with you"**——用得越久，越懂你的 AI Agent。

---

## 二、核心定位：Hermes 到底是什么？

市面上 Agent 框架分两类：

1. **IDE 编程助手**（如 Cursor、GitHub Copilot）——只能在写代码时用
2. **聊天机器人包装器**（如各种"你的 AI 助手"）——换皮 GPT，短期记忆

Hermes 走了第三条路：**一个能自我进化、跨平台运行、长期记忆的自主 Agent**。

它不依附于任何 IDE，也不只是一个聊天前端。你可以在命令行里跟它聊，也可以通过 Telegram/Discord/飞书/钉钉随时唤醒它——而且它会记住之前每一次对话的细节，从中提取规律，优化自己的行为。

---

## 三、安装部署：全平台指南

### 3.1 支持的平台一览

| 平台 | 安装方式 | 推荐度 |
|------|---------|--------|
| **Linux** | `curl \| bash` 一行命令 | ⭐⭐⭐⭐⭐ 最佳 |
| **macOS** | 同上 | ⭐⭐⭐⭐⭐ |
| **WSL2（Windows）** | 同上 | ⭐⭐⭐⭐ 最推荐 Windows 用户 |
| **Windows PowerShell** | `irm \| iex` 一键 | ⭐⭐⭐ 快速体验 |
| **Docker** | 容器化部署 | ⭐⭐⭐⭐ 生产环境 |
| **VPS** | 同 Linux | ⭐⭐⭐⭐ $5/月即可 |
| **Termux（Android）** | 独立指南 | ⭐⭐⭐ 手机实验 |

### 3.2 Linux / macOS / WSL2 安装

```bash
curl -fsSL https://res1.hermesagent.org.cn/install.sh | bash
```

国内网络会自动走镜像加速（CNB.cool + Cloudflare R2），无需手动翻墙。

### 3.3 Windows PowerShell 安装

```powershell
irm https://res1.hermesagent.org.cn/install.ps1 | iex
```

> ⚠️ 快速体验可以，但长期使用建议回到 WSL2，稳定性更好。

### 3.4 Docker 部署

适合作为服务端 7×24 运行，接入消息平台后可以随时通过 Telegram/飞书唤醒。

```bash
# 参考官方文档
# https://hermesagent.org.cn/docs/user-guide/docker
```

### 3.5 安装后三步走

```bash
# 1. 验证安装
hermes --version

# 2. 配置模型（以 OpenRouter 为例）
hermes config set provider.openrouter.api_key "your-key"
hermes config set model "openrouter/anthropic/claude-sonnet-4"

# 3. 启动对话
hermes
```

### 3.6 一键 Portal 配置（最省事）

如果你不想一个个配置 API Key，可以走 Nous Portal 一站式方案：

```bash
hermes setup --portal
```

单个订阅覆盖 **300+ 模型** + 网页搜索 + 图片生成 + TTS + 云浏览器，全部零额外账号。

---

## 四、常用场景

### 场景一：个人 AI 助理（命令行）

```bash
hermes
> 帮我写一个 Python 脚本，每天从央行官网拉 M2 数据并存入 SQLite
```

Hermes 会自己写代码、运行、调试、把成功的工作流提炼成 Skill——下次你再提类似需求，它直接用 Skill 秒级响应。

### 场景二：消息平台接入（Telegram/飞书/钉钉）

```bash
hermes gateway start
```

配置好 `gateway.yaml` 后，你的 Hermes Agent 就变成了一个 7×24 在线助理：
- **Telegram**：直接 @Bot 发消息
- **飞书/钉钉/企业微信**：接入工作群，处理任务
- **Discord/Slack**：团队协作场景

### 场景三：定时自动化任务

```yaml
# 在对话中对 Hermes 说：
"每天早上 8 点帮我从央行官网抓取最新 M2 数据，
  整理成表格，通过 Telegram 发给我"
```

Hermes 内置 cron 调度器，用自然语言描述就能创建定时任务。日报、监控、备份——全自动化。

### 场景四：代码审查 + 学习助手

```bash
hermes
> 读取当前目录的 Python 项目，分析代码结构，指出潜在的性能问题
```

配合 FTS5 记忆系统，Hermes 会记住你的代码风格偏好，审查越来越精准。

### 场景五：科研 / 批量处理

Hermes 支持「批量轨迹生成」和「轨迹压缩」——一次让 Agent 处理 100 个相似任务，导出完整执行轨迹，用于训练 RL 模型或做 Agent 行为分析。

---

## 五、技术原理

### 5.1 六层架构骨架

Hermes 采用「自底向上」六层架构设计，每层职责清晰：

```
┌───────────────────────────────────────────────────────┐
│ L6 入口/UI   │ cli.py / TUI (Ink) / gateway 网关       │
├───────────────────────────────────────────────────────┤
│ L5 编排      │ run_agent.py — 同步 ReAct 推理循环       │
├───────────────────────────────────────────────────────┤
│ L4 工具      │ 40+ 内置工具 + MCP 协议扩展               │
├───────────────────────────────────────────────────────┤
│ L3 推理      │ Provider 适配器 / 缓存 / 压缩 / Curator  │
├───────────────────────────────────────────────────────┤
│ L2 持久化    │ SQLite + FTS5 全文索引 + 记忆插件         │
├───────────────────────────────────────────────────────┤
│ L1 系统      │ ~/.hermes/ 配置 + 7 种执行后端            │
└───────────────────────────────────────────────────────┘
```

各层含义：

- **L1 系统层**：管理配置文件、API Key、7 种后端（Local/Docker/SSH/Modal/Daytona等）
- **L2 持久化层**：跨会话记忆的核心——SQLite + FTS5 实现毫秒级全文检索，无需向量数据库
- **L3 推理层**：适配 300+ 模型，统一为 OpenAI message 格式，实现 provider 零代码切换
- **L4 工具层**：工具注册中心 + 自动发现，只需写 `registry.register()` 即可被 Agent 调用
- **L5 编排层**：核心推理循环，约 12,000 行代码，同步 while 循环驱动
- **L6 入口层**：支持命令行 TUI、Web、Telegram/飞书/钉钉等 22+ 平台

<svg viewBox="0 0 680 500" width="100%" xmlns="http://www.w3.org/2000/svg">
<defs>
<linearGradient id="l6g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#667eea"/><stop offset="100%" stop-color="#764ba2"/></linearGradient>
<linearGradient id="l5g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#f093fb"/><stop offset="100%" stop-color="#f5576c"/></linearGradient>
<linearGradient id="l4g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#4facfe"/><stop offset="100%" stop-color="#00f2fe"/></linearGradient>
<linearGradient id="l3g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#43e97b"/><stop offset="100%" stop-color="#38f9d7"/></linearGradient>
<linearGradient id="l2g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#fa709a"/><stop offset="100%" stop-color="#fee140"/></linearGradient>
<linearGradient id="l1g" x1="0" y1="0" x2="1" y2="0"><stop offset="0%" stop-color="#a18cd1"/><stop offset="100%" stop-color="#fbc2eb"/></linearGradient>
<filter id="shadow"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-opacity="0.15"/></filter>
</defs>
<rect width="680" height="500" rx="12" fill="#1a1a2e"/>
<text x="340" y="32" text-anchor="middle" fill="#e0e0e0" font-size="18" font-weight="bold" font-family="sans-serif">Hermes Agent 六层架构（自底向上）</text>
<!-- L6 -->
<rect x="40" y="55" width="600" height="56" rx="8" fill="url(#l6g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="80" fill="#fff" font-size="13" font-weight="bold" font-family="sans-serif">L6 入口/UI层</text>
<text x="60" y="98" fill="#fff" font-size="11" font-family="sans-serif">cli.py · TUI终端(Ink) · Gateway网关 · 22+消息平台适配(Telegram/飞书/钉钉/微信/Discord/Slack)</text>
<!-- L5 -->
<rect x="40" y="126" width="600" height="56" rx="8" fill="url(#l5g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="151" fill="#fff" font-size="13" font-weight="bold" font-family="sans-serif">L5 编排层 — 同步ReAct推理循环</text>
<text x="60" y="169" fill="#fff" font-size="11" font-family="sans-serif">run_agent.py (~12k LOC) · while循环+预算驱动 · 可中断 · 统一OpenAI消息格式</text>
<!-- L4 -->
<rect x="40" y="197" width="600" height="56" rx="8" fill="url(#l4g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="222" fill="#fff" font-size="13" font-weight="bold" font-family="sans-serif">L4 工具层 — 注册中心+自动发现</text>
<text x="60" y="240" fill="#fff" font-size="11" font-family="sans-serif">40+内置工具 · tools/registry.py零依赖 · MCP协议集成 · 代码执行/浏览器/子Agent委托</text>
<!-- L3 -->
<rect x="40" y="268" width="600" height="56" rx="8" fill="url(#l3g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="293" fill="#1a1a2e" font-size="13" font-weight="bold" font-family="sans-serif">L3 推理层 — 20+Provider适配器矩阵</text>
<text x="60" y="311" fill="#1a1a2e" font-size="11" font-family="sans-serif">OpenAI · Anthropic · Gemini · Bedrock · Copilot ACP · Curator自进化引擎 · context_compressor</text>
<!-- L2 -->
<rect x="40" y="339" width="600" height="56" rx="8" fill="url(#l2g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="364" fill="#1a1a2e" font-size="13" font-weight="bold" font-family="sans-serif">L2 持久化层 — SQLite+FTS5 记忆系统</text>
<text x="60" y="382" fill="#1a1a2e" font-size="11" font-family="sans-serif">毫秒级BM25全文检索 · LLM摘要压缩 · Honcho/mem0/supermemory插件扩展</text>
<!-- L1 -->
<rect x="40" y="410" width="600" height="56" rx="8" fill="url(#l1g)" filter="url(#shadow)" opacity="0.9"/>
<text x="60" y="435" fill="#1a1a2e" font-size="13" font-weight="bold" font-family="sans-serif">L1 系统层 — 配置+执行后端</text>
<text x="60" y="453" fill="#1a1a2e" font-size="11" font-family="sans-serif">~/.hermes/ · .env · 7种后端: Local/Docker/SSH/Daytona/Modal/Singularity/Vercel Sandbox</text>
<!-- arrows between layers -->
<line x1="340" y1="111" x2="340" y2="126" stroke="#555" stroke-width="1.5" stroke-dasharray="4,3"/>
<line x1="340" y1="182" x2="340" y2="197" stroke="#555" stroke-width="1.5" stroke-dasharray="4,3"/>
<line x1="340" y1="253" x2="340" y2="268" stroke="#555" stroke-width="1.5" stroke-dasharray="4,3"/>
<line x1="340" y1="324" x2="340" y2="339" stroke="#555" stroke-width="1.5" stroke-dasharray="4,3"/>
<line x1="340" y1="395" x2="340" y2="410" stroke="#555" stroke-width="1.5" stroke-dasharray="4,3"/>
</svg>

### 5.2 推理循环：同步 ReAct 范式

Hermes **没有使用 LangGraph/AutoGPT 的复杂图编排**，而是极简的同步 while 循环：

```python
while (api_call_count < max_iterations and
       iteration_budget.remaining > 0) or _budget_grace_call:
    if _interrupt_requested: break

    # 1. 发送消息 + 工具定义给模型
    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas
    )

    # 2. 如果有 tool_calls，执行并追加结果
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(
                tool_call.name, tool_call.args, task_id
            )
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        # 3. 没有工具调用了，返回最终文本
        return response.content
```

**四大设计要点**：

1. **同步循环**：循环体本身同步，异步只在 IO 边界（流式 API、网关），避免 async/await 嵌套
2. **预算驱动**：`max_iterations`（默认 90）硬上限 + `iteration_budget` 软预算 + `_budget_grace_call` 一次"恩典调用"做收尾
3. **可中断**：每轮检查 `_interrupt_requested`，支持 Ctrl+C / `/stop` 即时退出
4. **统一消息格式**：所有 provider 统一为 OpenAI message 格式，推理内容存入 `assistant_msg["reasoning"]`，切换模型零代码

### 5.3 自进化机制：Curator 闭环

这是 Hermes 最核心的差异化能力，由 `agent/curator.py`（约 75KB）实现：

```
Agent 完成复杂任务
       ↓
Curator 后台提炼（空闲时间窗触发）
       ↓
将成功模式写成 Skill（技能文件）
       ↓
下次同类任务直接命中 Skill → 效率指数级提升
```

**Curator 关键配置**：

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `curator.enabled` | true | 自进化开关 |
| `interval_hours` | 24 | 提炼周期 |
| `min_idle_hours` | 2 | 仅空闲时工作，不打扰主对话 |
| `stale_after_days` | 30 | 技能过时标记 |
| `archive_after_days` | 90 | 自动归档老技能 |

> **本质**：Hermes 自己在后台总结自己的成功经验，写成可复用的技能。这跟人类「做完一个项目写复盘文档」是一个逻辑，只是它是自动的。

<svg viewBox="0 0 680 420" width="100%" xmlns="http://www.w3.org/2000/svg">
<defs>
<linearGradient id="cg1" x1="0" y1="0" x2="1" y2="1"><stop offset="0%" stop-color="#667eea"/><stop offset="100%" stop-color="#764ba2"/></linearGradient>
<linearGradient id="cg2" x1="0" y1="0" x2="1" y2="1"><stop offset="0%" stop-color="#f093fb"/><stop offset="100%" stop-color="#f5576c"/></linearGradient>
<linearGradient id="cg3" x1="0" y1="0" x2="1" y2="1"><stop offset="0%" stop-color="#43e97b"/><stop offset="100%" stop-color="#38f9d7"/></linearGradient>
<linearGradient id="cg4" x1="0" y1="0" x2="1" y2="1"><stop offset="0%" stop-color="#fa709a"/><stop offset="100%" stop-color="#fee140"/></linearGradient>
<marker id="arrowGreen" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#43e97b"/></marker>
<marker id="arrowPink" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto"><polygon points="0 0, 10 3.5, 0 7" fill="#f5576c"/></marker>
<filter id="sh"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-opacity="0.2"/></filter>
</defs>
<rect width="680" height="420" rx="12" fill="#1a1a2e"/>
<text x="340" y="32" text-anchor="middle" fill="#e0e0e0" font-size="18" font-weight="bold" font-family="sans-serif">Curator 自进化闭环</text>
<!-- Step 1 -->
<rect x="60" y="60" width="240" height="72" rx="10" fill="url(#cg1)" filter="url(#sh)"/>
<text x="180" y="86" text-anchor="middle" fill="#fff" font-size="14" font-weight="bold" font-family="sans-serif">1. Agent执行复杂任务</text>
<text x="180" y="108" text-anchor="middle" fill="#e0e0ff" font-size="11" font-family="sans-serif">完成多步骤操作后触发提炼</text>
<text x="180" y="124" text-anchor="middle" fill="#e0e0ff" font-size="10" font-family="sans-serif">例：从央行拉M2数据→清洗→画图</text>
<!-- Step 2 -->
<rect x="380" y="60" width="240" height="72" rx="10" fill="url(#cg2)" filter="url(#sh)"/>
<text x="500" y="86" text-anchor="middle" fill="#fff" font-size="14" font-weight="bold" font-family="sans-serif">2. Curator后台提炼</text>
<text x="500" y="108" text-anchor="middle" fill="#ffe0f0" font-size="11" font-family="sans-serif">空闲时间窗触发（默认2h）</text>
<text x="500" y="124" text-anchor="middle" fill="#ffe0f0" font-size="10" font-family="sans-serif">提取成功的操作模式</text>
<!-- Arrow 1→2 -->
<line x1="300" y1="96" x2="375" y2="96" stroke="#f5576c" stroke-width="2" marker-end="url(#arrowPink)"/>
<!-- Step 3 -->
<rect x="380" y="170" width="240" height="72" rx="10" fill="url(#cg3)" filter="url(#sh)"/>
<text x="500" y="196" text-anchor="middle" fill="#1a1a2e" font-size="14" font-weight="bold" font-family="sans-serif">3. 生成Skill文件</text>
<text x="500" y="218" text-anchor="middle" fill="#1a4020" font-size="11" font-family="sans-serif">写入~/.hermes/skills/目录</text>
<text x="500" y="234" text-anchor="middle" fill="#1a4020" font-size="10" font-family="sans-serif">兼容agentskills.io标准</text>
<!-- Arrow 2→3 -->
<line x1="500" y1="132" x2="500" y2="165" stroke="#43e97b" stroke-width="2" marker-end="url(#arrowGreen)"/>
<!-- Step 4 -->
<rect x="60" y="170" width="240" height="72" rx="10" fill="url(#cg4)" filter="url(#sh)"/>
<text x="180" y="196" text-anchor="middle" fill="#1a1a2e" font-size="14" font-weight="bold" font-family="sans-serif">4. 下次任务命中Skill</text>
<text x="180" y="218" text-anchor="middle" fill="#402010" font-size="11" font-family="sans-serif">用户说"M2数据"→秒级响应</text>
<text x="180" y="234" text-anchor="middle" fill="#402010" font-size="10" font-family="sans-serif">效率从8步→1步，跳过探索</text>
<!-- Arrow 3→4 -->
<line x1="380" y1="206" x2="305" y2="206" stroke="#fee140" stroke-width="2" marker-end="url(#arrowGreen)"/>
<!-- Arrow 4→1 (loop) -->
<path d="M 60 206 C 30 206, 30 96, 55 96" stroke="#667eea" stroke-width="2" fill="none" marker-end="url(#arrowGreen)" stroke-dasharray="6,3"/>
<!-- 中间循环说明 -->
<rect x="220" y="290" width="240" height="36" rx="18" fill="none" stroke="#38f9d7" stroke-width="1.5" stroke-dasharray="4,3"/>
<text x="340" y="313" text-anchor="middle" fill="#38f9d7" font-size="13" font-weight="bold" font-family="sans-serif">♻ 技能在使用中持续改进</text>
<!-- 底部说明 -->
<text x="340" y="355" text-anchor="middle" fill="#888" font-size="11" font-family="sans-serif">stale_after_days=30 → 过时标记 | archive_after_days=90 → 自动归档 | 生命周期完整闭环</text>
<text x="340" y="380" text-anchor="middle" fill="#666" font-size="10" font-family="sans-serif">curator.py (~75KB) · 仅在空闲窗口工作，绝不干扰主对话</text>
</svg>

### 5.4 记忆系统：SQLite + FTS5 的工程化选择

Hermes 的记忆系统没有用向量数据库（Pinecone/Weaviate/Milvus），而是选了 **SQLite + FTS5 全文索引**。

**为什么不用向量数据库？**

| 方案 | 优势 | 劣势 |
|------|------|------|
| SQLite FTS5 | 零依赖、$5 VPS 跑得动、毫秒级 BM25 检索 | 不支持语义相似度 |
| 向量数据库 | 语义检索精准 | 需要 GPU/向量模型、内存开销大、运维复杂 |

Hermes 的折中方案：**FTS5 做关键词召回 + LLM 摘要做语义压缩**。

```
用户输入 "上次聊到 X，继续"
       ↓
FTS5 命中相关历史片段（毫秒级）
       ↓
LLM 摘要压缩后塞回当前 context
       ↓
预算可控地恢复上下文
```

当确实需要语义检索时，可以通过插件接入 Honcho（辩证用户建模）、mem0（键值记忆）或 supermemory（多模态记忆）。

### 5.5 工具系统：注册中心 + 自动发现

```python
# 开发者只需在 tools/ 目录下写：
from tools.registry import registry

@registry.register("my_custom_tool")
def my_custom_tool(query: str) -> str:
    """搜索我的内部知识库"""
    return search_kb(query)
```

写完之后不需要手动 import——Hermes 的 `tools/registry.py` 会自动发现所有注册的工具。

**工具暴露两阶段**：自动发现 → 显式启用。这样即使工具代码存在，也不会被 Agent 误用（安全设计）。

**40+ 核心工具分类**：

| 类别 | 代表工具 |
|------|---------|
| 代码与文件 | `code_execution_tool.py`、`file_operations.py` |
| 浏览器与 GUI | `browser_tool.py`、`computer_use_tool.py` |
| 代理协作 | `delegate_tool.py`（子 Agent 并行）、`mixture_of_agents_tool.py` |
| MCP 协议 | `mcp_tool.py`、`managed_tool_gateway.py` |
| Agent 元能力 | `memory_tool.py`、`checkpoint_manager.py`、`cronjob_tools.py` |

---

## 六、核心架构深度拆解

### 6.1 Provider 抽象层：300+ 模型统一适配

Hermes 支持 300+ 模型，但不依赖任何单一供应商：

| 适配器 | 对应服务 |
|--------|---------|
| `openai_adapter.py` | OpenAI / 兼容接口（通义千问、GLM、Kimi、MiniMax） |
| `anthropic_adapter.py` | Claude Messages API（含 extended thinking） |
| `bedrock_adapter.py` | AWS Bedrock |
| `gemini_native_adapter.py` | Google Gemini 原生 API |
| `copilot_acp_client.py` | GitHub Copilot ACP |

每个适配器只处理一件事：把模型输出统一为 OpenAI message 格式。切换模型只需改一行配置：

```bash
hermes model set openrouter/anthropic/claude-sonnet-4
hermes model set nous-portal/nous-hermes-3
hermes model set openai/gpt-4o
```

### 6.2 多平台消息网关：22+ 渠道统一路由

Hermes 的 `gateway/` 模块实现了一个统一的消息路由层：

```
Telegram  ─┐
Discord  ─┤
飞书     ─┼──→ gateway/run.py ──→ AIAgent 推理循环 ──→ 统一响应
钉钉     ─┤
微信     ─┘
```

每个平台只需实现 `send_message` / `receive_event` 接口，其余逻辑完全复用。支持 **Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Mattermost、Email、SMS、飞书、钉钉、企业微信、微信、QQBot、BlueBubbles、Yuanbao、Webhook、API Server** 等。

### 6.3 七种执行后端

| 后端 | 特点 | 适用场景 |
|------|------|---------|
| `local` | 本地 shell | 个人电脑 |
| `docker` | 容器隔离 | 生产环境 |
| `ssh` | 远程主机 | VPS 部署 |
| `daytona` | Serverless 持久化沙箱 | 空闲休眠，零费用 |
| `modal` | Serverless GPU | 按需唤醒，按量计费 |
| `singularity` | HPC 容器 | 科研计算集群 |
| `vercel sandbox` | 边缘运行时 | 低延迟场景 |

> 这意味着 Hermes 可以在你笔记本上跑，也可以在 $5/月的 VPS 上跑，甚至可以在几乎免费的 Serverless 平台上跑。

### 6.4 技能系统：程序性记忆的标准化

Hermes 的技能（Skill）是一个 `.md` 文件，描述了一个「场景 → 操作 → 结果」的固定工作流。兼容 [agentskills.io](https://agentskills.io) 开放标准。

**技能的生命周期**：
```
Agent 完成任务 → Curator 提炼 → 生成 Skill → 下次命中 → 执行中改进 → 过时标记 → 自动归档
```

这意味着一开始你告诉 Hermes "帮我查央行 M2 数据并画图"，它可能要花 8 步操作。等它提炼成 Skill 后，下次说"M2 数据"它就秒回——因为它已经记住怎么做了。

---

## 七、Hermes Agent vs OpenClaw：终极对比

在 AI Agent 圈子里，这两个是目前讨论最多的开源框架。它们都很好，但设计哲学完全不同。

### 7.1 一句话区别

- **Hermes** → "我得自己变聪明"（自进化 + 主动记忆）
- **OpenClaw** → "我给你最多工具和渠道"（生态 + 技能市场 + 多平台）

### 7.2 12 维度对比表

| 对比维度 | Hermes Agent | OpenClaw |
|---------|-------------|---------|
| **定位** | 自进化 AI 伙伴 | IDE 深度集成 + 通用 Agent 平台 |
| **核心卖点** | 用得越久越聪明 | 开箱即用，生态最大 |
| **记忆方式** | **主动记忆**（自动提取+SQLite存储） | **文件记忆**（Markdown 手动/程序写入） |
| **记忆存储** | SQLite + FTS5 全文索引 | Markdown 文件（MEMORY.md / YYYY-MM-DD.md） |
| **技能系统** | Agent 自主创建技能 + agentskills.io 标准 | 技能市场 13,000+ / Skill 安装制 |
| **推理循环** | 同步 while + 预算驱动（极简） | 异步编排（复杂，但更灵活） |
| **模型支持** | 300+（OpenRouter / Nous Portal / 自定义） | 多模型切换（需配置 Skill） |
| **消息平台** | 22+（Telegram/飞书/钉钉/微信/Discord 等） | 50+（更广的 IoT/智能家居支持） |
| **执行后端** | 7 种（含 Serverless） | 本地 / Docker |
| **安装方式** | `curl \| bash` 一行命令 | npm / pip / Docker |
| **GitHub Stars** | 164k（3个月） | 更早积累，生态更成熟 |
| **技术门槛** | 较高（需命令行 + 手动配置） | 较低（有 Web UI + 图形界面） |
| **隐私** | ⭐⭐⭐⭐⭐（纯本地 SQLite） | ⭐⭐⭐（文件系统 + 可本地部署） |

### 7.3 记忆系统的本质区别

这是两个框架最核心的差异：

**OpenClaw 的模式**（"你告诉我记什么"）：
```
用户明确要求 → 写入 MEMORY.md / YYYY-MM-DD.md → 下次加载上下文时读取
```
问题：如果用户忘了说「记住这个」，这个信息就丢了。

**Hermes 的模式**（"我替你记"）：
```
每轮对话 → Curator 后台自动提炼 → 存入 SQLite 索引 → FTS5 检索 → 下次自动命中
```
优势：你不需要告诉它"记住这个"，它自己会判断什么值得记住。

**一个直观的例子**：

你在 Hermes 里说 "我不喜欢蓝色主题的图表，用深色背景配黄色线条"，三周后你再让它画图，它自动用深色+黄色——因为它已经把"你的图表偏好"存入了记忆，并且通过 FTS5 自动关联到了当前任务。

在 OpenClaw 里，除非你明确让它"把这个偏好记到 MEMORY.md"，否则它下次可能忘了。

### 7.4 适用场景选择

**选 Hermes Agent，如果你**：
- 每天都在用 Agent 处理大量相似任务，希望它越用越聪明
- 有技术背景，不介意命令行和手动配置
- 高度关注隐私，希望所有记忆和对话数据完全本地存储
- 需要一个「长期 AI 伙伴」而非「一次性工具」
- 习惯飞书/钉钉办公，希望 Agent 集成到工作流

**选 OpenClaw，如果你**：
- 刚入门 AI Agent，想要开箱即用
- 需要同时管理 Telegram + 微信 + Discord 多个平台
- 偏好图形界面操作（Web UI）
- 想做**金融日报自动化**这类已有成熟 Skill 的标准化任务
- 需要一个全平台 AI 秘书管理日程、消息、提醒

### 7.5 进阶玩法：两者组合

理论上可以把两者结合：
- **OpenClaw** 做「前台」→ 管理所有聊天渠道，统一接收/回复
- **Hermes** 做「大脑」→ 处理复杂任务、调用深度记忆、自我进化推理

目前没有官方集成方案，需要自行编写对接脚本。但方向上很值得期待。

---

## 八、总结

### Hermes Agent 的核心价值

1. **自进化闭环**：Agent 自己从成功经验中提炼 Skill，下次更快更好——这是目前唯一做到「越用越聪明」的开源框架
2. **记忆系统工程化**：SQLite + FTS5 的方案极其务实——不追求向量数据库的"高级感"，而是在 $5 VPS 上实现真正的跨会话记忆
3. **模型无锁定**：300+ 模型任意切换，不绑定任何供应商
4. **架构极简**：同步 while 循环 + 预算驱动的设计哲学——拒绝过度工程化
5. **社区活跃**：164k Stars、每周千级增长、平均一周一个版本——这不是昙花一现的项目

### 适合谁

- 每天重度使用 AI 的**开发者、运维、研究人员**
- 关注**隐私和本地化部署**的技术团队
- 需要 AI 成为「长期伙伴」而非「一次性工具」的人

### 不适合谁

- 想要**图形界面一键操作**的新手
- 只需要**偶尔问 AI 几个问题**的轻度用户
- 依赖 OpenClaw 已有完善 Skill 生态（如金融日报、微信发布）的用户

---

### 参考链接

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [Hermes Agent 中文社区](https://hermesagent.org.cn/)
- [Hermes Agent 架构深度解析](https://www.cnblogs.com/qiniushanghai/p/20012754)
- [Nous Portal](https://portal.nousresearch.com)
- [OpenClaw 官网](https://openclaw.ai)

---

*本文基于 Hermes Agent v0.14.0（2026.5.16）撰写，架构细节参考了官方源码和社区分析文章。GitHub 数据截至 2026 年 5 月 23 日。*
