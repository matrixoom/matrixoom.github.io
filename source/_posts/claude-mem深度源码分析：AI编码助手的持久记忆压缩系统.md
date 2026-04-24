---
title: claude-mem深度源码分析：AI编码助手的持久记忆压缩系统
cover: /images/cover-claudemem.png
sticky: 0
comments: true
categories:
  - AI技术
  - 开源项目
tags:
  - Claude
  - AI记忆
  - Hook系统
  - ChromaDB
  - MCP协议
  - 开源
date: 2026-04-24 20:00:00
---

# claude-mem深度源码分析：AI编码助手的持久记忆压缩系统

## 引言：AI编码助手的"失忆症"

你有没有这样的经历——和Claude Code讨论了一个复杂的架构方案，做完决策后关闭会话，第二天打开新会话时，Claude对昨天的讨论一无所知，你不得不重新解释所有背景？

这不是bug，而是LLM的天生局限：**无状态**。每个会话都是一张白纸。

**claude-mem** 正是为了解决这个问题而生的。它不是简单的聊天记录保存工具，而是一套完整的**记忆压缩与检索系统**——通过Hook机制拦截Claude Code的每个操作事件，将工具调用结果压缩为精炼的observation，存入SQLite+ChromaDB双引擎存储，在新会话开始时自动注入相关上下文。

项目由Alex Newman（@thedotmack）开发，在GitHub上拥有超过3万星标，已成为Claude Code生态中最受欢迎的记忆增强方案。

> 项目地址：https://github.com/thedotmack/claude-mem

---

## 一、快速上手：三步开启持久记忆

### 1.1 安装

```bash
# 一行命令安装
npx claude-mem install
```

安装过程会自动完成以下操作：
- 在`~/.claude/settings.json`中注册7个Hook
- 创建数据目录`~/.claude-mem/`
- 初始化SQLite数据库
- 配置环境变量

### 1.2 系统要求

| 组件 | 最低要求 | 推荐 |
|------|----------|------|
| **Claude Code** | 最新版 | 最新版 |
| **Bun** | ≥1.0 | 最新版 |
| **Node.js** | ≥18 | ≥20 |
| **磁盘空间** | 100MB | 500MB+ |
| **API Key** | Anthropic API Key | 支持多provider |

### 1.3 核心配置

安装后，你可以在`~/.claude-mem/settings.json`中进行配置。关键设置项：

```json
{
  "model": "claude-sonnet-4-6",
  "mode": "code",
  "semantic_inject": true,
  "chroma_enabled": true,
  "worker_port": 37700,
  "max_concurrent_agents": 3,
  "observation_dedup_window_seconds": 30
}
```

### 1.4 工作模式

claude-mem内置了多种工作模式（位于`plugin/modes/`目录）：

| 模式 | 说明 | observation类型 |
|------|------|-----------------|
| `code` | 通用编码（默认） | decision, finding, action, pattern, error, insight |
| `code--zh` | 中文编码模式 | 继承code，输出语言为中文 |
| `email-investigation` | 邮件调查 | person, event, evidence, timeline, connection, finding |
| `raw` | 原始模式 | 无类型过滤 |

模式之间支持继承，例如`code--zh`继承`code`的所有类型定义，仅覆盖语言设置。

---

## 二、整体架构：四层系统设计

claude-mem采用四层架构设计，各层职责清晰、松耦合：

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 680" style="background:#1a1a2e;border-radius:12px">
<defs>
<linearGradient id="g1" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" style="stop-color:#e94560"/><stop offset="100%" style="stop-color:#c23152"/></linearGradient>
<linearGradient id="g2" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" style="stop-color:#0f3460"/><stop offset="100%" style="stop-color:#16213e"/></linearGradient>
<linearGradient id="g3" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" style="stop-color:#533483"/><stop offset="100%" style="stop-color:#3a1f5e"/></linearGradient>
<linearGradient id="g4" x1="0%" y1="0%" x2="100%" y2="0%"><stop offset="0%" style="stop-color:#1a8a5c"/><stop offset="100%" style="stop-color:#145a3e"/></linearGradient>
<filter id="shadow"><feDropShadow dx="0" dy="2" stdDeviation="3" flood-color="#000" flood-opacity="0.3"/></filter>
</defs>
<text x="450" y="38" text-anchor="middle" fill="#e0e0e0" font-size="20" font-weight="bold" font-family="sans-serif">claude-mem 四层架构全景图</text>
<!-- Layer 1: Claude Code Host -->
<rect x="40" y="58" width="820" height="110" rx="10" fill="url(#g1)" filter="url(#shadow)" opacity="0.95"/>
<text x="60" y="85" fill="#fff" font-size="16" font-weight="bold" font-family="sans-serif">Layer 1: Claude Code 宿主层</text>
<rect x="70" y="100" width="150" height="50" rx="8" fill="#e94560" stroke="#ff6b81" stroke-width="1.5"/>
<text x="145" y="130" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">用户输入</text>
<rect x="260" y="100" width="150" height="50" rx="8" fill="#e94560" stroke="#ff6b81" stroke-width="1.5"/>
<text x="335" y="130" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">工具调用</text>
<rect x="450" y="100" width="150" height="50" rx="8" fill="#e94560" stroke="#ff6b81" stroke-width="1.5"/>
<text x="525" y="130" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">AI响应</text>
<rect x="640" y="100" width="190" height="50" rx="8" fill="#e94560" stroke="#ff6b81" stroke-width="1.5"/>
<text x="735" y="130" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">会话生命周期</text>
<!-- Arrow down -->
<path d="M450,168 L450,188" stroke="#aaa" stroke-width="2" marker-end="url(#arrow)"/>
<defs><marker id="arrow" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="8" markerHeight="8" orient="auto"><path d="M0,0 L10,5 L0,10 Z" fill="#aaa"/></marker></defs>
<!-- Layer 2: CLI Hook Layer -->
<rect x="40" y="188" width="820" height="130" rx="10" fill="url(#g2)" filter="url(#shadow)" opacity="0.95"/>
<text x="60" y="215" fill="#fff" font-size="16" font-weight="bold" font-family="sans-serif">Layer 2: CLI Hook 拦截层</text>
<rect x="55" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="102" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">Setup</text>
<rect x="158" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="205" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">SessionStart</text>
<rect x="261" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="308" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">PreToolUse</text>
<rect x="364" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="411" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">PostToolUse</text>
<rect x="467" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="514" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">UserPrompt</text>
<rect x="570" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="617" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">Stop</text>
<rect x="673" y="230" width="95" height="45" rx="6" fill="#0f3460" stroke="#4a90d9" stroke-width="1.5"/>
<text x="720" y="257" text-anchor="middle" fill="#8ec8f0" font-size="11" font-family="sans-serif">SessionEnd</text>
<text x="60" y="300" fill="#8899aa" font-size="11" font-family="sans-serif">hook-command.ts → normalizeInput → execute(handler) → formatOutput</text>
<!-- Arrow down -->
<path d="M450,318 L450,338" stroke="#aaa" stroke-width="2" marker-end="url(#arrow)"/>
<!-- Layer 3: Worker Daemon -->
<rect x="40" y="338" width="820" height="140" rx="10" fill="url(#g3)" filter="url(#shadow)" opacity="0.95"/>
<text x="60" y="365" fill="#fff" font-size="16" font-weight="bold" font-family="sans-serif">Layer 3: Worker 守护进程层 (Express HTTP :37777)</text>
<rect x="55" y="380" width="120" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="115" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">SessionManager</text>
<rect x="185" y="380" width="120" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="245" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">SDKAgent</text>
<rect x="315" y="380" width="120" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="375" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">SearchManager</text>
<rect x="445" y="380" width="120" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="505" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">ContextBuilder</text>
<rect x="575" y="380" width="130" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="640" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">PendingMsgStore</text>
<rect x="715" y="380" width="120" height="40" rx="6" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="775" y="405" text-anchor="middle" fill="#d4a5f0" font-size="12" font-family="sans-serif">ModeManager</text>
<text x="60" y="450" fill="#8899aa" font-size="11" font-family="sans-serif">ChromaSync · SSEBroadcaster · ProcessRegistry · KnowledgeAgent</text>
<!-- Arrow down -->
<path d="M450,478 L450,498" stroke="#aaa" stroke-width="2" marker-end="url(#arrow)"/>
<!-- Layer 4: Storage -->
<rect x="40" y="498" width="820" height="110" rx="10" fill="url(#g4)" filter="url(#shadow)" opacity="0.95"/>
<text x="60" y="525" fill="#fff" font-size="16" font-weight="bold" font-family="sans-serif">Layer 4: 存储层</text>
<rect x="70" y="540" width="250" height="50" rx="8" fill="#1a8a5c" stroke="#2ecc71" stroke-width="1.5"/>
<text x="195" y="563" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">SQLite (WAL + FTS5)</text>
<text x="195" y="580" text-anchor="middle" fill="#a5d6b0" font-size="10" font-family="sans-serif">sessions · observations · summaries · pending_messages</text>
<rect x="360" y="540" width="250" height="50" rx="8" fill="#1a8a5c" stroke="#2ecc71" stroke-width="1.5"/>
<text x="485" y="563" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">ChromaDB (向量检索)</text>
<text x="485" y="580" text-anchor="middle" fill="#a5d6b0" font-size="10" font-family="sans-serif">narrative · facts · summary → 语义搜索</text>
<rect x="650" y="540" width="180" height="50" rx="8" fill="#1a8a5c" stroke="#2ecc71" stroke-width="1.5"/>
<text x="740" y="563" text-anchor="middle" fill="#fff" font-size="13" font-family="sans-serif">文件系统</text>
<text x="740" y="580" text-anchor="middle" fill="#a5d6b0" font-size="10" font-family="sans-serif">~/.claude-mem/</text>
<!-- Footer -->
<text x="450" y="640" text-anchor="middle" fill="#666" font-size="11" font-family="sans-serif">数据流方向：Claude Code → Hook拦截 → Worker处理 → 双引擎存储 → 上下文注入回Claude Code</text>
</svg>

### 架构要点解读

**宿主层（Layer 1）**是Claude Code本身，claude-mem通过Hook机制与之集成，不需要修改Claude Code的源码。

**CLI Hook拦截层（Layer 2）**是claude-mem的入口，7个Hook分别拦截Claude Code生命周期中的关键事件。每个Hook收到stdin中的JSON数据，经过`normalizeInput → execute → formatOutput`的管道处理后，将结果通过stdout返回给Claude Code。

**Worker守护进程层（Layer 3）**是核心业务逻辑所在，以Express HTTP服务的形式运行在端口37777上。它管理着会话生命周期、SDK子进程调度、搜索策略、上下文构建等全部核心功能。

**存储层（Layer 4）**采用SQLite+ChromaDB双引擎设计。SQLite负责结构化存储和元数据查询，ChromaDB负责向量语义搜索。

---

## 三、Hook生命周期：Claude Code的事件拦截

claude-mem的核心接入机制是Claude Code的Hook系统。每个Hook在Claude Code的特定生命周期节点被触发，接收stdin中的JSON数据，返回处理结果。

### 3.1 七个Hook一览

| Hook | 触发时机 | claude-mem处理器 | 超时 | 核心功能 |
|------|----------|-----------------|------|----------|
| **Setup** | 插件安装时 | 确保Worker运行 | 300s | 初始化环境 |
| **SessionStart** | 新会话开始 | context handler | 60s | 注入历史上下文 |
| **UserPromptSubmit** | 用户发送消息 | session-init handler | 60s | 初始化会话+语义检索 |
| **PreToolUse** | 工具调用前 | file-context handler | 2000s | 文件级上下文注入 |
| **PostToolUse** | 工具调用后 | observation handler | 120s | 观测数据压缩 |
| **Stop** | AI响应结束 | summarize handler | 120s | 会话摘要生成 |
| **SessionEnd** | 会话关闭 | session-complete handler | 30s | 清理资源 |

### 3.2 Hook生命周期流程

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 520" style="background:#1a1a2e;border-radius:12px">
<defs>
<marker id="arrowW" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 Z" fill="#e0e0e0"/></marker>
</defs>
<text x="430" y="32" text-anchor="middle" fill="#e0e0e0" font-size="18" font-weight="bold" font-family="sans-serif">Hook 生命周期与数据流</text>
<!-- Session Start -->
<rect x="30" y="55" width="140" height="55" rx="8" fill="#e94560" opacity="0.9"/>
<text x="100" y="78" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SessionStart</text>
<text x="100" y="98" text-anchor="middle" fill="#ffcdd2" font-size="10" font-family="sans-serif">注入历史上下文</text>
<line x1="170" y1="82" x2="205" y2="82" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- User Prompt -->
<rect x="205" y="55" width="140" height="55" rx="8" fill="#0f3460" opacity="0.9"/>
<text x="275" y="78" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">UserPromptSubmit</text>
<text x="275" y="98" text-anchor="middle" fill="#90caf9" font-size="10" font-family="sans-serif">初始化会话+语义检索</text>
<line x1="345" y1="82" x2="380" y2="82" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- PreToolUse -->
<rect x="380" y="55" width="140" height="55" rx="8" fill="#533483" opacity="0.9"/>
<text x="450" y="78" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">PreToolUse</text>
<text x="450" y="98" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">文件级上下文注入</text>
<line x1="450" y1="110" x2="450" y2="140" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- Claude Processing -->
<rect x="355" y="140" width="190" height="45" rx="8" fill="#2c2c54" stroke="#666" stroke-width="1" stroke-dasharray="5,3"/>
<text x="450" y="167" text-anchor="middle" fill="#aaa" font-size="12" font-family="sans-serif">Claude Code 执行工具</text>
<line x1="450" y1="185" x2="450" y2="215" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- PostToolUse -->
<rect x="380" y="215" width="140" height="55" rx="8" fill="#1a8a5c" opacity="0.9"/>
<text x="450" y="238" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">PostToolUse</text>
<text x="450" y="258" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">观测数据压缩</text>
<line x1="450" y1="270" x2="450" y2="300" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- Loop indicator -->
<rect x="555" y="155" width="120" height="40" rx="6" fill="#2c2c54" stroke="#888" stroke-width="1" stroke-dasharray="4,3"/>
<text x="615" y="178" text-anchor="middle" fill="#888" font-size="10" font-family="sans-serif">多次工具调用循环</text>
<path d="M545,175 Q560,175 555,175" stroke="#888" stroke-width="1" fill="none"/>
<!-- Stop -->
<rect x="380" y="300" width="140" height="55" rx="8" fill="#e67e22" opacity="0.9"/>
<text x="450" y="323" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">Stop</text>
<text x="450" y="343" text-anchor="middle" fill="#ffe0b2" font-size="10" font-family="sans-serif">会话摘要生成</text>
<line x1="450" y1="355" x2="450" y2="385" stroke="#e0e0e0" stroke-width="1.5" marker-end="url(#arrowW)"/>
<!-- SessionEnd -->
<rect x="380" y="385" width="140" height="55" rx="8" fill="#7f8c8d" opacity="0.9"/>
<text x="450" y="408" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SessionEnd</text>
<text x="450" y="428" text-anchor="middle" fill="#d5dbdb" font-size="10" font-family="sans-serif">清理资源</text>
<!-- Right side: Data stores -->
<rect x="630" y="140" width="200" height="90" rx="8" fill="#145a3e" opacity="0.8" stroke="#2ecc71" stroke-width="1"/>
<text x="730" y="165" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SQLite 存储</text>
<text x="730" y="185" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">observations</text>
<text x="730" y="200" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">session_summaries</text>
<text x="730" y="215" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">pending_messages</text>
<!-- ChromaDB -->
<rect x="630" y="250" width="200" height="70" rx="8" fill="#1a237e" opacity="0.8" stroke="#5c6bc0" stroke-width="1"/>
<text x="730" y="275" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">ChromaDB 向量库</text>
<text x="730" y="295" text-anchor="middle" fill="#9fa8da" font-size="10" font-family="sans-serif">narrative · facts · summary</text>
<text x="730" y="310" text-anchor="middle" fill="#9fa8da" font-size="10" font-family="sans-serif">语义搜索</text>
<!-- SDK Agent -->
<rect x="630" y="340" width="200" height="70" rx="8" fill="#533483" opacity="0.8" stroke="#9b59b6" stroke-width="1"/>
<text x="730" y="365" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SDK Agent 子进程</text>
<text x="730" y="385" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">Claude / Gemini / OpenRouter</text>
<text x="730" y="400" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">observation压缩 + summary生成</text>
<!-- Arrows from hooks to stores -->
<path d="M520,242 Q580,242 630,185" stroke="#2ecc71" stroke-width="1.5" fill="none" marker-end="url(#arrowW)"/>
<path d="M520,327 Q580,300 630,285" stroke="#5c6bc0" stroke-width="1.5" fill="none" marker-end="url(#arrowW)"/>
<path d="M520,327 Q590,360 630,375" stroke="#9b59b6" stroke-width="1.5" fill="none" marker-end="url(#arrowW)"/>
<!-- Context inject arrow -->
<path d="M100,110 Q100,480 380,480" stroke="#e94560" stroke-width="1.5" fill="none" stroke-dasharray="6,3" marker-end="url(#arrowW)"/>
<text x="60" y="480" fill="#e94560" font-size="10" font-family="sans-serif">上下文注入</text>
<text x="60" y="495" fill="#e94560" font-size="10" font-family="sans-serif">（新会话自动加载）</text>
</svg>

### 3.3 Hook执行管道

每个Hook的执行都遵循统一的管道模式：

```
stdin(JSON) → readJsonFromStdin → normalizeInput → execute(handler) → formatOutput → stdout(JSON)
```

**关键设计决策**：Hook的错误处理采用优雅降级策略——

- **传输层错误**（Worker不可用、网络中断等）：退出码0，Claude Code正常继续
- **客户端Bug**（数据格式错误、逻辑异常等）：退出码2，Claude Code显示错误

这意味着claude-mem永远不会因为自身问题阻塞Claude Code的正常使用。

---

## 四、核心引擎：记忆压缩与检索

### 4.1 Observation压缩管线

这是claude-mem最核心的机制。每次Claude Code执行工具调用后，PostToolUse Hook将工具调用的输入输出发送给Worker，Worker启动SDK Agent子进程将原始数据压缩为结构化的observation。

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 400" style="background:#1a1a2e;border-radius:12px">
<defs>
<marker id="arrowO" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 Z" fill="#ffc107"/></marker>
</defs>
<text x="430" y="32" text-anchor="middle" fill="#e0e0e0" font-size="18" font-weight="bold" font-family="sans-serif">Observation 压缩管线</text>
<!-- Step 1: Raw Tool Output -->
<rect x="30" y="60" width="160" height="80" rx="8" fill="#2c2c54" stroke="#888" stroke-width="1.5"/>
<text x="110" y="85" text-anchor="middle" fill="#e0e0e0" font-size="12" font-weight="bold" font-family="sans-serif">原始工具输出</text>
<text x="110" y="105" text-anchor="middle" fill="#aaa" font-size="10" font-family="sans-serif">1000-10000 tokens</text>
<text x="110" y="125" text-anchor="middle" fill="#888" font-size="10" font-family="sans-serif">文件内容/搜索结果/…</text>
<path d="M190,100 L230,100" stroke="#ffc107" stroke-width="2" marker-end="url(#arrowO)"/>
<!-- Step 2: PostToolUse Hook -->
<rect x="230" y="60" width="160" height="80" rx="8" fill="#1a8a5c" stroke="#2ecc71" stroke-width="1.5"/>
<text x="310" y="85" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">PostToolUse Hook</text>
<text x="310" y="105" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">隐私检查</text>
<text x="310" y="125" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">发送到Worker API</text>
<path d="M390,100 L430,100" stroke="#ffc107" stroke-width="2" marker-end="url(#arrowO)"/>
<!-- Step 3: SDK Agent -->
<rect x="430" y="50" width="180" height="100" rx="8" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="520" y="75" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SDK Agent</text>
<text x="520" y="95" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">Claude子进程</text>
<text x="520" y="115" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">XML结构化输出</text>
<text x="520" y="135" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">(all tools disallowed)</text>
<path d="M610,100 L650,100" stroke="#ffc107" stroke-width="2" marker-end="url(#arrowO)"/>
<!-- Step 4: Compressed Observation -->
<rect x="650" y="60" width="170" height="80" rx="8" fill="#e94560" stroke="#ff6b81" stroke-width="1.5"/>
<text x="735" y="85" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">压缩后Observation</text>
<text x="735" y="105" text-anchor="middle" fill="#ffcdd2" font-size="10" font-family="sans-serif">~500 tokens</text>
<text x="735" y="125" text-anchor="middle" fill="#ffcdd2" font-size="10" font-family="sans-serif">type+title+narrative+facts</text>
<!-- Dedup box -->
<rect x="30" y="175" width="800" height="90" rx="8" fill="#2c2c54" stroke="#666" stroke-width="1"/>
<text x="430" y="200" text-anchor="middle" fill="#e0e0e0" font-size="13" font-weight="bold" font-family="sans-serif">去重与持久化</text>
<rect x="60" y="215" width="220" height="35" rx="6" fill="#145a3e" stroke="#2ecc71" stroke-width="1"/>
<text x="170" y="237" text-anchor="middle" fill="#a5d6a7" font-size="11" font-family="sans-serif">SHA256内容哈希去重 (30s窗口)</text>
<rect x="310" y="215" width="220" height="35" rx="6" fill="#145a3e" stroke="#2ecc71" stroke-width="1"/>
<text x="420" y="237" text-anchor="middle" fill="#a5d6a7" font-size="11" font-family="sans-serif">SQLite WAL写入</text>
<rect x="560" y="215" width="240" height="35" rx="6" fill="#1a237e" stroke="#5c6bc0" stroke-width="1"/>
<text x="680" y="237" text-anchor="middle" fill="#9fa8da" font-size="11" font-family="sans-serif">ChromaDB向量同步 (批量100)</text>
<!-- XML Output Format -->
<rect x="30" y="290" width="800" height="95" rx="8" fill="#1a1a2e" stroke="#555" stroke-width="1"/>
<text x="430" y="315" text-anchor="middle" fill="#ffc107" font-size="13" font-weight="bold" font-family="sans-serif">SDK Agent XML 输出格式</text>
<text x="60" y="340" fill="#a5d6a7" font-size="11" font-family="monospace">&lt;observation type="decision|finding|action|pattern|error|insight"&gt;</text>
<text x="80" y="358" fill="#90caf9" font-size="11" font-family="monospace">&lt;title&gt;简洁标题&lt;/title&gt;</text>
<text x="80" y="374" fill="#90caf9" font-size="11" font-family="monospace">&lt;narrative&gt;压缩叙述（核心信息保留）&lt;/narrative&gt;</text>
<text x="60" y="374" fill="#a5d6a7" font-size="11" font-family="monospace">  </text>
</svg>

压缩的核心是SDK Agent。它通过`@anthropic-ai/claude-agent-sdk`启动一个Claude子进程，**禁用所有工具**（observer-only模式），将工具调用结果作为输入，让Claude将其压缩为结构化的XML输出。典型压缩比：1000-10000 tokens → ~500 tokens，压缩率80-95%。

### 4.2 CLAIM-CONFIRM消息队列

SDK Agent的调用是异步的，claude-mem使用了一个精心设计的持久化消息队列来管理：

```
enqueue(pending) → claimNextMessage(processing) → confirmProcessed(delete) / markFailed(retry<3)
```

**自愈机制**：如果消息处于`processing`状态超过60秒（比如Worker崩溃），系统会自动将其重置为`pending`状态，确保消息不会丢失。

**重试限制**：失败消息最多重试3次，超过后标记为永久失败，防止无限循环。

### 4.3 混合搜索策略

claude-mem实现了三种搜索策略，通过`SearchOrchestrator`统一调度：

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 860 340" style="background:#1a1a2e;border-radius:12px">
<defs>
<marker id="arrowS" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 Z" fill="#ffc107"/></marker>
</defs>
<text x="430" y="32" text-anchor="middle" fill="#e0e0e0" font-size="18" font-weight="bold" font-family="sans-serif">混合搜索策略架构</text>
<!-- Query Input -->
<rect x="330" y="50" width="200" height="45" rx="8" fill="#e94560" opacity="0.9"/>
<text x="430" y="78" text-anchor="middle" fill="#fff" font-size="13" font-weight="bold" font-family="sans-serif">SearchOrchestrator</text>
<!-- Three strategies -->
<line x1="350" y1="95" x2="150" y2="140" stroke="#ffc107" stroke-width="1.5" marker-end="url(#arrowS)"/>
<line x1="430" y1="95" x2="430" y2="140" stroke="#ffc107" stroke-width="1.5" marker-end="url(#arrowS)"/>
<line x1="510" y1="95" x2="710" y2="140" stroke="#ffc107" stroke-width="1.5" marker-end="url(#arrowS)"/>
<!-- SQLite Strategy -->
<rect x="40" y="140" width="220" height="100" rx="8" fill="#145a3e" stroke="#2ecc71" stroke-width="1.5"/>
<text x="150" y="165" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">SQLiteSearchStrategy</text>
<text x="150" y="185" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">纯过滤查询（无query文本）</text>
<text x="150" y="205" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">按type/concept/file过滤</text>
<text x="150" y="225" text-anchor="middle" fill="#a5d6a7" font-size="10" font-family="sans-serif">Chroma不可用时的fallback</text>
<!-- Chroma Strategy -->
<rect x="310" y="140" width="240" height="100" rx="8" fill="#1a237e" stroke="#5c6bc0" stroke-width="1.5"/>
<text x="430" y="165" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">ChromaSearchStrategy</text>
<text x="430" y="185" text-anchor="middle" fill="#9fa8da" font-size="10" font-family="sans-serif">向量语义搜索</text>
<text x="430" y="205" text-anchor="middle" fill="#9fa8da" font-size="10" font-family="sans-serif">query → Chroma → 90天过滤</text>
<text x="430" y="225" text-anchor="middle" fill="#9fa8da" font-size="10" font-family="sans-serif">按doc_type分类 → SQLite回填</text>
<!-- Hybrid Strategy -->
<rect x="600" y="140" width="220" height="100" rx="8" fill="#533483" stroke="#9b59b6" stroke-width="1.5"/>
<text x="710" y="165" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">HybridSearchStrategy</text>
<text x="710" y="185" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">SQLite过滤 → Chroma排序</text>
<text x="710" y="205" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">取交集 → 从SQLite回填</text>
<text x="710" y="225" text-anchor="middle" fill="#ce93d8" font-size="10" font-family="sans-serif">findByConcept/File/Type</text>
<!-- Result merge -->
<line x1="150" y1="240" x2="350" y2="280" stroke="#aaa" stroke-width="1" marker-end="url(#arrowS)"/>
<line x1="430" y1="240" x2="430" y2="280" stroke="#aaa" stroke-width="1" marker-end="url(#arrowS)"/>
<line x1="710" y1="240" x2="510" y2="280" stroke="#aaa" stroke-width="1" marker-end="url(#arrowS)"/>
<rect x="300" y="280" width="260" height="45" rx="8" fill="#e67e22" opacity="0.9"/>
<text x="430" y="308" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold" font-family="sans-serif">合并结果 → ContextBuilder</text>
</svg>

**HybridSearchStrategy**是最常用的策略，它先用SQLite的元数据过滤缩小范围，再用ChromaDB的语义排序精选相关结果，兼顾了精确性和语义理解。

---

## 五、上下文注入：让Claude"记住"

### 5.1 ContextBuilder编排

当新会话开始时，SessionStart Hook触发context handler，调用Worker的`/api/context/inject`接口。ContextBuilder负责编排整个上下文注入流程：

```
loadConfig → queryObservations/Summaries → buildTimeline → renderHeader/Timeline/Summary/Footer
```

注入的上下文包含四个渲染器输出的内容：

| 渲染器 | 功能 | 内容 |
|--------|------|------|
| **HeaderRenderer** | 会话元信息 | 项目名、会话数、token预算 |
| **TimelineRenderer** | 时间线视图 | 彩色终端时间线，按时间排列的observation |
| **SummaryRenderer** | 摘要视图 | 历史会话的结构化摘要 |
| **FooterRenderer** | 节省统计 | token节省量、压缩比 |

### 5.2 渐进式披露

claude-mem采用渐进式披露（Progressive Disclosure）策略，不在一次注入中塞入所有历史信息，而是：

1. **会话开始时**：注入最近会话的摘要 + 高相关度observation
2. **用户发消息时**：根据当前prompt语义检索相关历史
3. **工具调用前**：根据即将读取的文件注入文件级上下文

### 5.3 Token经济学

TokenCalculator跟踪token使用情况：

- **发现Token**（discoveryTokens）：如果不使用claude-mem，Claude需要读取的原始数据量
- **阅读Token**（readTokens）：claude-mem压缩后注入的上下文量
- **节省量** = discoveryTokens - readTokens

典型场景下，claude-mem能将上下文注入量压缩到原始数据的10-20%。

---

## 六、关键设计模式

### 6.1 断路器（Circuit Breaker）

SDK Agent的子进程可能因为各种原因失败（API限流、网络中断、模型错误等）。为防止无限重试，SessionManager实现了断路器：

```typescript
// 连续重启限制
if (consecutiveRestarts >= 3) {
  // 停止重试，等待手动干预
  return;
}
```

当连续重启达到3次时，断路器触发，停止自动恢复。此时Claude Code仍可正常使用（因为优雅降级），但不会产生新的observation。

### 6.2 优雅降级（Graceful Degradation）

这是claude-mem最重要的设计原则之一：**Worker不可用时，永远不阻塞Claude Code**。

在`hook-command.ts`中：

```typescript
function isWorkerUnavailableError(error: any): boolean {
  // 传输层错误：ECONNREFUSED, timeout, etc.
  return /ECONNREFUSED|ETIMEDOUT|ECONNRESET|fetch failed/i.test(error.message);
}

// 传输层错误 → exit 0（Claude Code继续）
// 客户端Bug → exit 2（Claude Code显示错误）
```

### 6.3 内容哈希去重

同一会话中可能产生重复的observation（例如Claude重复执行相同操作）。claude-mem使用SHA256内容哈希去重：

```
hash = SHA256(memory_session_id + title + narrative)[:16]
```

在30秒的去重窗口内，相同哈希的observation不会被重复存储。

### 6.4 隔离环境（Isolated Environment）

SDK Agent启动子进程时，使用`buildIsolatedEnv()`构建隔离环境：

- **API Key保护**：从项目`.env`中剥离`ANTHROPIC_API_KEY`，防止密钥泄露
- **独立配置**：子进程使用`~/.claude-mem/.env`中的凭证
- **工具禁用**：所有工具设为disallowed，子进程仅作为观察者

---

## 七、存储架构详解

### 7.1 SQLite Schema

claude-mem的SQLite数据库采用WAL模式，包含以下核心表：

```
┌─────────────────────────────────────────────────────┐
│                  SQLite 数据库                       │
├─────────────────────────────────────────────────────┤
│ sdk_sessions                                        │
│   ├── id (PK)                                       │
│   ├── memory_session_id                             │
│   ├── project_path                                  │
│   ├── status (active/completed/failed)              │
│   └── created_at / updated_at                       │
├─────────────────────────────────────────────────────┤
│ observations                                        │
│   ├── id (PK)                                       │
│   ├── memory_session_id (FK)                        │
│   ├── type (decision/finding/action/pattern/error)  │
│   ├── title                                         │
│   ├── narrative (压缩叙述)                           │
│   ├── facts (JSON数组)                               │
│   ├── content_hash (SHA256[:16], 去重)              │
│   ├── file_paths (JSON数组)                          │
│   ├── concepts (JSON数组)                            │
│   └── created_at                                    │
├─────────────────────────────────────────────────────┤
│ session_summaries                                   │
│   ├── id (PK)                                       │
│   ├── memory_session_id (FK)                        │
│   ├── summary_text                                  │
│   ├── key_decisions (JSON)                          │
│   └── created_at                                    │
├─────────────────────────────────────────────────────┤
│ pending_messages                                    │
│   ├── id (PK)                                       │
│   ├── status (pending/processing/failed)            │
│   ├── payload (JSON)                                │
│   ├── retry_count (max 3)                           │
│   ├── claimed_at                                    │
│   └── created_at                                    │
└─────────────────────────────────────────────────────┘
```

### 7.2 ChromaDB文档结构

每个observation在ChromaDB中被拆分为多个文档，以支持细粒度语义检索：

```
obs_{id}_narrative    → observation的叙述文本
obs_{id}_fact_0       → 第1个fact
obs_{id}_fact_1       → 第2个fact
...
sum_{id}_summary      → 会话摘要文本
```

这种拆分策略使得检索时可以精确匹配到observation中的特定fact，而不是只能匹配整段叙述。

### 7.3 Schema迁移

claude-mem的数据库schema经历了6次重大演进：

| 迁移 | 变更 |
|------|------|
| v1→v2 | 从memories/overviews迁移到observations/session_summaries |
| v3 | 添加sdk_sessions表，支持SDK Agent |
| v4 | 重构session模型 |
| v5 | 添加content_hash去重字段 |
| v6 | 添加FTS5全文索引（正在废弃中） |

迁移采用内联方式（在SessionStore中），每次启动时自动检测并执行。

---

## 八、多Provider支持与模式系统

### 8.1 Agent Provider

SDK Agent支持三种LLM Provider：

| Provider | 模型 | 说明 |
|----------|------|------|
| **Claude** | claude-sonnet-4-6 | 默认，通过Agent SDK调用 |
| **Gemini** | gemini-2.5-flash | 通过Google AI SDK调用 |
| **OpenRouter** | 可配置 | 支持任意OpenRouter模型 |

三种Provider共享`conversationHistory`，允许在会话中途切换Provider而不丢失上下文。

### 8.2 Mode系统

ModeManager是一个单例，管理着claude-mem的工作模式：

```json
// plugin/modes/code.json
{
  "name": "code",
  "observation_types": ["decision", "finding", "action", "pattern", "error", "insight"],
  "concepts": ["architecture", "bug", "feature", "refactor", "test", "deploy", "config"],
  "prompts": {
    "init": "...",
    "observation": "...",
    "summary": "..."
  }
}
```

模式之间支持继承：

```json
// plugin/modes/code--zh.json
{
  "name": "code--zh",
  "inherits": "code",
  "language": "zh-CN",
  "overrides": {
    "prompts": { "observation": "请用中文输出observation..." }
  }
}
```

### 8.3 知识图谱（Knowledge Agent）

claude-mem还包含一个实验性的知识图谱构建器：

- **CorpusStore**：管理知识语料
- **CorpusBuilder**：从observation中提取实体和关系
- **KnowledgeAgent**：使用LLM构建知识图谱

这个功能目前还在早期阶段，但展示了claude-mem从"记忆检索"向"知识推理"演进的方向。

---

## 九、MCP服务器：另一种集成方式

除了Hook机制，claude-mem还提供了MCP（Model Context Protocol）服务器，允许其他AI工具访问claude-mem的记忆：

```
MCP Client (Claude Desktop, etc.)
    ↓ stdio
MCP Server (Node.js)
    ↓ HTTP
Worker Service (Bun)
```

MCP服务器提供以下工具：

| 工具 | 功能 |
|------|------|
| `search` | 语义搜索observations |
| `timeline` | 获取会话时间线 |
| `get_observations` | 按ID获取observation详情 |
| `smart_outline` | 智能文件大纲（tree-sitter解析） |

值得注意的是，MCP服务器运行在Node.js下（而非Bun），因为MCP SDK的stdio传输在Node.js中更稳定。所有业务逻辑仍由Worker HTTP API处理，MCP服务器只是一个薄代理层。

---

## 十、性能优化与工程实践

### 10.1 SQLite优化

claude-mem对SQLite做了大量优化：

```typescript
// Database.ts 中的关键配置
db.exec('PRAGMA journal_mode = WAL');      // WAL模式，读写不互斥
db.exec('PRAGMA mmap_size = 268435456');   // 256MB内存映射
db.exec('PRAGMA cache_size = -10000');      // 10K页缓存
db.exec('PRAGMA synchronous = NORMAL');     // 平衡安全与性能
```

### 10.2 Worker启动优化

Worker的启动使用了多种策略确保快速响应：

- **PID文件检测**：先检查PID文件判断Worker是否已运行
- **健康检查**：HTTP探活，避免重复启动
- **端口占用检测**：防止端口冲突
- **Windows冷却期**：启动失败后2分钟冷却，防止弹窗循环

### 10.3 Session过期回收

SessionManager实现了两级过期回收：

- **生成器空闲回收**：5分钟无活动 → SIGKILL
- **会话空闲回收**：15分钟无活动 → 标记完成

这确保了资源不会因为遗忘的会话而泄漏。

### 10.4 智能文件读取

file-context handler实现了智能文件读取策略：

- **文件门控**：小于1.5KB的文件跳过（Claude自己读就够了）
- **mtime缓存**：基于文件修改时间的缓存失效
- **截断读取**：无限制读取截断为1行，节省token

---

## 十一、与同类项目对比

| 特性 | claude-mem | MemPalace | Cline Memory |
|------|-----------|-----------|--------------|
| **存储策略** | 压缩observation + 向量检索 | 原始对话 + 向量检索 | 摘要存储 |
| **压缩比** | 80-95% | 0%（存储原文） | 60-80% |
| **搜索方式** | 混合搜索（SQLite+ChromaDB） | 纯向量搜索 | 关键词搜索 |
| **集成方式** | Hook + MCP | MCP | 内置 |
| **多Provider** | ✅ Claude/Gemini/OpenRouter | ❌ | ❌ |
| **模式系统** | ✅ 可扩展 | ❌ | ❌ |
| **隐私控制** | ✅ 逐observation | ✅ 逐对话 | ❌ |
| **优雅降级** | ✅ Worker不可用不阻塞 | N/A | N/A |
| **开源自托管** | ✅ AGPL-3.0 | ✅ MIT | ✅ |

claude-mem的独特优势在于：**压缩而非原文存储 + 混合搜索 + 优雅降级**。这意味着它在token效率上远胜原文存储方案，在搜索精度上优于纯向量方案，在可靠性上远超任何可能阻塞宿主AI的方案。

---

## 十二、总结与展望

### 核心设计哲学

claude-mem的设计可以归结为三个核心哲学：

1. **绝不伤害宿主**：优雅降级是第一优先级，claude-mem的任何故障都不能阻塞Claude Code
2. **压缩而非堆砌**：通过LLM压缩将原始数据提炼为精炼observation，而非简单地存储所有内容
3. **渐进式披露**：根据上下文动态注入最相关的记忆，避免一次性token爆炸

### 架构亮点

- **Hook管道**：7个Hook覆盖Claude Code完整生命周期，实现了无侵入式集成
- **CLAIM-CONFIRM队列**：带自愈的持久化消息队列，确保observation不丢失
- **混合搜索**：SQLite元数据过滤 + ChromaDB语义排序，精度与效率兼顾
- **断路器**：连续失败自动熔断，防止无限重试
- **Mode系统**：可扩展的工作模式，支持不同场景的observation类型和提示词

### 局限与挑战

- **API成本**：每次observation压缩都消耗一次API调用，高频率使用时成本不低
- **延迟**：SDK Agent子进程启动和压缩需要时间，可能影响响应速度
- **FTS5废弃**：全文搜索功能正在被ChromaDB替代，过渡期可能不稳定
- **Windows兼容**：部分功能（如进程管理）在Windows上需要特殊处理

### 未来方向

从源码中可以看到几个明确的演进方向：

1. **知识图谱**：Knowledge Agent正在从"记忆检索"向"知识推理"演进
2. **多Provider优化**：Gemini和OpenRouter的支持正在完善，未来可能支持更多Provider
3. **RAGTIME**：邮件调查批量处理器展示了claude-mem在非编码场景的潜力
4. **Web Viewer**：正在开发的Web界面将提供记忆的可视化浏览和管理

claude-mem不仅仅是一个"让Claude记住东西"的工具，它是一套完整的**AI记忆基础设施**——从事件拦截、数据压缩、持久化存储、语义检索到上下文注入，每个环节都有精心设计。对于任何想要构建AI持久记忆系统的开发者，claude-mem的源码都是一份值得深入研究的参考。

---

> **参考资料**
> - 项目仓库：https://github.com/thedotmack/claude-mem
> - 架构文档：docs/architecture-overview.md
> - Hook规范：Claude Code官方文档
> - MCP协议：https://modelcontextprotocol.io/
