---
title: "OpenClaw 深度解读：一个懂你的自托管 AI 助手网关"
date: 2026-06-06 17:00:00
tags:
  - AI Agent
  - OpenClaw
  - 源码分析
  - TypeScript
  - 网关架构
categories: 技术学习
---

## 这是什么？

OpenClaw 是一个**自托管的个人 AI 助手网关**，TypeScript 编写，MIT 开源，由 Peter Steinberger（奥地利 iOS 开发者，PSPDFKit 创始人）和社区共同维护。

一句话定位：**把你的各种聊天软件（微信 / QQ / Telegram / Discord / WhatsApp / 飞书等 22+ 平台）接入同一个 AI 大脑，跑在你自己的设备上。**

它的前身经历过三次改名：Warelay → Clawdbot → Moltbot → OpenClaw。作者最初的动机很朴素：想做个能真正干活的 AI 助手，而不仅仅是个聊天机器人。从去年到现在已经迭代了近两年，社区活跃度很高。

---

## 为什么值得研究它

AI Agent 赛道现在不缺产品，但大多数 Agent 是「single session, single channel」：你开一个终端或 IDE 跟它对话，关了就没了。

OpenClaw 走的是另一条路：**Gateway 常驻 + 多通道接入 + 多 Agent 路由**。它不是一个编程工具，也不是一个聊天插件，而是一个**控制平面**——消息从微信进来，路由到你的主力 Agent，Agent 调工具干活，结果通过微信发回去。全程没有切换成本，你不需要掏出另一个 App。

技术栈也选得有意思：TypeScript。按照官方的解释，选 TS 不是因为性能，而是因为「OpenClaw 本质是一个编排系统——处理 prompts、工具、协议、集成——TypeScript 在这个领域迭代最快，也最容易 hack」。

---

## 核心架构

OpenClaw 的架构可以用一句话概括：**一个 Gateway 管全局，WebSocket 协议做通信，多通道做消息总线，Agent Runtime 做执行引擎。**

<svg viewBox="0 0 680 580" width="100%" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="OpenClaw 系统架构图">
  <defs>
    <style>
      text { font-family: sans-serif; }
      .title { font-size: 16px; font-weight: bold; }
      .label { font-size: 13px; font-weight: bold; }
      .body { font-size: 12px; }
      .caption { font-size: 11px; fill: #666; }
    </style>
  </defs>

  <rect width="680" height="580" fill="#FAFAFA" rx="8"/>

  <!-- Title -->
  <text x="340" y="28" text-anchor="middle" class="title" fill="#222">OpenClaw 系统架构</text>

  <!-- ===== 输入层: Channels ===== -->
  <rect x="20" y="45" width="640" height="95" fill="#EFF6FF" rx="6" stroke="#3B82F6" stroke-width="1.5"/>
  <text x="340" y="65" text-anchor="middle" class="label" fill="#1E40AF">输入层：22+ 消息通道</text>
  <text x="35" y="85" class="caption" fill="#3B82F6">WhatsApp</text>
  <text x="110" y="85" class="caption" fill="#3B82F6">Telegram</text>
  <text x="185" y="85" class="caption" fill="#3B82F6">WeChat / QQ</text>
  <text x="270" y="85" class="caption" fill="#3B82F6">Discord / Slack</text>
  <text x="360" y="85" class="caption" fill="#3B82F6">飞书</text>
  <text x="410" y="85" class="caption" fill="#3B82F6">Google Chat</text>
  <text x="500" y="85" class="caption" fill="#3B82F6">Signal / iMessage</text>
  <text x="35" y="105" class="caption" fill="#3B82F6">Microsoft Teams / Matrix / LINE / Mattermost / Nextcloud Talk / Nostr / IRC / Zalo / Twitch</text>
  <text x="35" y="125" class="caption" fill="#64748B">每个通道由对应 adapter 驱动（grammY for Telegram / Baileys for WhatsApp / 微信 SDK 等）</text>

  <!-- Arrow down -->
  <line x1="340" y1="142" x2="340" y2="160" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- ===== Gateway 核心 ===== -->
  <rect x="20" y="162" width="640" height="100" fill="#FEF3C7" rx="6" stroke="#D97706" stroke-width="1.5"/>
  <text x="340" y="182" text-anchor="middle" class="label" fill="#92400E">Gateway（端口 18789，WebSocket 协议）</text>
  <rect x="35" y="195" width="175" height="55" fill="#FFFBEB" rx="4" stroke="#D97706" stroke-width="0.5"/>
  <text x="122" y="215" text-anchor="middle" class="body" fill="#92400E">消息路由</text>
  <text x="122" y="232" text-anchor="middle" class="caption">DM配对 → Session分发</text>
  <text x="122" y="246" text-anchor="middle" class="caption">多Agent路由策略</text>

  <rect x="230" y="195" width="175" height="55" fill="#FFFBEB" rx="4" stroke="#D97706" stroke-width="0.5"/>
  <text x="317" y="215" text-anchor="middle" class="body" fill="#92400E">命令队列</text>
  <text x="317" y="232" text-anchor="middle" class="caption">per-session 串行化</text>
  <text x="317" y="246" text-anchor="middle" class="caption">steer/followup/collect/interrupt</text>

  <rect x="425" y="195" width="220" height="55" fill="#FFFBEB" rx="4" stroke="#D97706" stroke-width="0.5"/>
  <text x="535" y="215" text-anchor="middle" class="body" fill="#92400E">生命周期管理</text>
  <text x="535" y="232" text-anchor="middle" class="caption">health/heartbeat/presence</text>
  <text x="535" y="246" text-anchor="middle" class="caption">cron调度 + Webhook触发</text>

  <!-- Arrow down -->
  <line x1="122" y1="264" x2="122" y2="290" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>
  <line x1="535" y1="264" x2="535" y2="290" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- ===== Agent Runtime ===== -->
  <rect x="20" y="292" width="640" height="115" fill="#F5F3FF" rx="6" stroke="#7C3AED" stroke-width="1.5"/>
  <text x="340" y="312" text-anchor="middle" class="label" fill="#5B21B6">Agent Runtime 执行层</text>

  <rect x="35" y="325" width="300" height="70" fill="#EDE9FE" rx="4" stroke="#7C3AED" stroke-width="0.5"/>
  <text x="185" y="343" text-anchor="middle" class="body" fill="#5B21B6">OpenClaw 原生 Runtime</text>
  <text x="185" y="360" text-anchor="middle" class="caption">while-loop: context → model → tool → persist</text>
  <text x="185" y="375" text-anchor="middle" class="caption">生命周期事件: lifecycle / assistant / tool</text>
  <text x="185" y="390" text-anchor="middle" class="caption">自动压缩 + 超时保护（默认48h）</text>

  <rect x="355" y="325" width="290" height="70" fill="#EDE9FE" rx="4" stroke="#7C3AED" stroke-width="0.5"/>
  <text x="500" y="343" text-anchor="middle" class="body" fill="#5B21B6">外部 Harness 适配</text>
  <text x="365" y="360" class="caption">Codex（OpenAI 订阅）</text>
  <text x="365" y="375" class="caption">Copilot（GitHub Copilot CLI）</text>
  <text x="365" y="390" class="caption">Claude CLI / Gemini CLI / OpenCode / Cursor（ACP协议）</text>

  <!-- Arrow down -->
  <line x1="185" y1="409" x2="185" y2="430" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>
  <line x1="500" y1="409" x2="500" y2="430" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- ===== Tools & Plugins ===== -->
  <rect x="20" y="432" width="640" height="65" fill="#F0FDF4" rx="6" stroke="#16A34A" stroke-width="1.5"/>
  <text x="340" y="452" text-anchor="middle" class="label" fill="#15803D">插件 & 工具层</text>
  <text x="35" y="472" class="caption" fill="#15803D">MCP（Server+Client）</text>
  <text x="175" y="472" class="caption" fill="#15803D">Skills（agentskills.io）</text>
  <text x="310" y="472" class="caption" fill="#15803D">ClawHub 插件市场</text>
  <text x="425" y="472" class="caption" fill="#15803D">Canvas（A2UI）</text>
  <text x="530" y="472" class="caption" fill="#15803D">Cron / Webhooks</text>
  <text x="35" y="490" class="caption" fill="#64748B">代码插件（运行时hook）+ Bundle插件（Skills/MCP打包）→ npm分发</text>

  <!-- Arrow down -->
  <line x1="340" y1="499" x2="340" y2="515" stroke="#94A3B8" stroke-width="2" marker-end="url(#arrow)"/>

  <!-- ===== Companion Apps ===== -->
  <rect x="20" y="517" width="640" height="50" fill="#EFF6FF" rx="6" stroke="#3B82F6" stroke-width="1.5"/>
  <text x="340" y="537" text-anchor="middle" class="label" fill="#1E40AF">配套应用（可选）</text>
  <text x="35" y="557" class="caption" fill="#3B82F6">Windows Hub</text>
  <text x="140" y="557" class="caption" fill="#3B82F6">macOS 菜单栏</text>
  <text x="240" y="557" class="caption" fill="#3B82F6">iOS Node</text>
  <text x="315" y="557" class="caption" fill="#3B82F6">Android Node</text>
  <text x="410" y="557" class="caption" fill="#3B82F6">WebChat</text>
  <text x="480" y="557" class="caption" fill="#3B82F6">Voice Wake / Talk</text>

  <!-- Defs: arrow marker -->
  <defs>
    <marker id="arrow" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto">
      <path d="M0,0 L8,4 L0,8 Z" fill="#94A3B8"/>
    </marker>
  </defs>
</svg>

### Gateway：一切的中心

Gateway（默认端口 `18789`）是 OpenClaw 的「唯一大脑」。它不是跑在一个线程里的 Web 服务，而是常驻的操作系统级守护进程——macOS 用 launchd，Linux 用 systemd，Windows 用系统服务。

Gateway 的核心职责是**长期维护所有消息通道的连接状态**：WhatsApp 用的是 Baileys 驱动，Telegram 用 grammY，Discord/Slack/微信各自有适配器。所有通道的收发都经过 Gateway 统一处理。

和其他同类工具的关键差异：**Gateway 是 WebSocket 协议，不是 REST API**。这意味着：

- 客户端和服务端之间是长连接，可以实时推送事件
- 控制面板（CLI / macOS App / Web UI）通过同一个 WS 连接管理全局
- 手机节点（iOS / Android）也通过 WS 注册为 `node` 角色，暴露 `camera.*`、`screen.record`、`location.get` 等设备能力

所有 WS 连接都必须先完成**设备配对**：客户端发送 `connect` 帧，Gateway 验证身份后返回 `hello-ok`。配对机制保证了**即使是本地回环，也要明确授权**——只有操作者手动审批过的设备才能连。

### Agent Runtime：执行引擎的核心

OpenClaw 的 Agent Loop 是典型的 **while-loop 模型**：

```
接收消息 → 解析模型/参数 → 构建上下文 → 调用LLM → 执行工具 → 流式输出 → 持久化
```

但细节之处有很多工程化成果：

**队列机制**：每个 session 有独立的串行队列，防止多个消息同时触发同一个 session 时发生工具调用冲突或历史记录错乱。通道层可以选四种队列模式——`steer`（操控）、`followup`（跟进）、`collect`（收集）、`interrupt`（打断）。

**运行时无关设计**：这是 OpenClaw 最独特的架构决策。Agent Loop 本身不绑定任何模型运行时，而是通过「Harness 适配层」支持多种后端：

| Runtime | 说明 |
|---------|------|
| `openclaw` | 原生运行时，直接调用各提供商的 API |
| `codex` | 通过 OpenAI Codex app-server 执行 Agent turn |
| `copilot` | 通过 GitHub Copilot CLI（`@github/copilot-sdk`）执行 |
| `claude-cli` | 通过 Claude CLI 进程执行（外部后端模式） |
| `acp` | 通过 ACP/acpx 协议接入 Claude Code / Gemini CLI / Cursor 等外部 harness |

这个设计的意义在于：**你可以在 OpenClaw 的同一个对话会话里，让不同模型的不同运行时交替服务**。比如日常闲聊用 `openclaw + GPT-5.5`，复杂编码任务用 `codex + GPT-5.5`，本地容灾用 `claude-cli`。

### Provider / Model / Runtime 三层解耦

很多人容易把 Provider、Model、Runtime 混为一谈。OpenClaw 的架构把它们拆成了三层：

| 层 | 举例 | 说明 |
|----|------|------|
| Provider | `openai`、`anthropic`、`github-copilot` | 认证方式、模型发现、API 端点 |
| Model | `gpt-5.5`、`claude-opus-4-8` | 具体模型的选择 |
| Runtime | `openclaw`、`codex`、`claude-cli` | 谁来执行 Agent turn |

配置示例：

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-8",
      models: {
        "anthropic/claude-opus-4-8": {
          agentRuntime: { id: "claude-cli" }  // 用 Claude CLI 执行
        },
      },
    },
  },
}
```

---

## 关键机制深度解析

### 多 Agent 路由：不只是「一个 AI」

OpenClaw 支持配置多个 Agent 实例，每个 Agent 可以有独立的 workspace、模型、skills、会话历史。消息进来时，Gateway 根据通道/账号/发送者路由到对应的 Agent：

```json5
{
  agents: {
    list: [
      { id: "me", name: "我的助理", model: "openai/gpt-5.5" },
      {
        id: "work",
        name: "工作助理",
        model: "anthropic/claude-opus-4-8",
        workspace: "~/.openclaw/workspace-work"
      },
    ],
  },
}
```

这意味着**一台机器上可以跑多个 AI 人格**，互不干扰。

### 安全模型：配对 + 沙箱

OpenClaw 连接的都是真实的消息平台，安全问题不是可选项。

**DM 配对机制**：默认所有 DM 消息（Telegram/WhatsApp/Signal/Discord/Slack 等）都需要配对才能处理。陌生人发消息，机器人只回复一个配对码，不执行任何指令。操作者用 `openclaw pairing approve <channel> <code>` 手动批准后，该发送者才被加入本地白名单。

**沙箱隔离**：非 `main` session 可以强制在 Docker / SSH / OpenShell 沙箱中运行：
- 沙箱默认允许：`bash`、`process`、`read`、`write`、`edit`、基本的 sessions 操作
- 沙箱默认禁止：`browser`、`canvas`、`nodes`、`cron`、`discord`、`gateway` 等全局性工具

这个设计解决了「在公司群聊里接入 AI 助手，但又不能让它随便操作宿主机」的痛点。

### 插件系统：代码插件 + Bundle 插件

OpenClaw 的插件机制有两种形式：

1. **Code Plugins（代码插件）**：运行时注册 hook，深度集成到 Agent 循环中。可以拦截工具调用（`before_tool_call` / `after_tool_call`）、注入上下文（`before_prompt_build`）、甚至直接代答（`before_agent_reply`）
2. **Bundle Plugins（Bundle 插件）**：打包 Skills、MCP 服务器、prompts、themes，通过 npm 分发

插件在 `package.json` 中声明资源：

```json
{
  "openclaw": {
    "extensions": ["extensions/index.ts"],
    "skills": ["skills/*.md"],
    "prompts": ["prompts/*.md"],
    "themes": ["themes/*.json"]
  }
}
```

插件市场在 [ClawHub](https://clawhub.ai)。核心代码刻意保持轻量，大多数功能走插件化，这也是 OpenClaw 代码库虽然大但核心逻辑清晰的原因。

### Canvas 与 A2UI

一个容易被忽略但很特别的功能：OpenClaw 内置了一个 **Canvas 渲染系统**，Agent 可以直接编写 HTML/CSS/JS 生成可视化界面，通过 Gateway 的 HTTP 路由 `/__openclaw__/canvas/` 对外展示。同时支持 A2UI（Agent-to-User Interface）协议。

这意味着不仅仅是文字对话——Agent 可以画图表、做交互式仪表盘、甚至生成小型 Web 应用。

---

## 部署方式

### 最简单的安装

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

`onboard` 向导会引导设置 Gateway、workspace、通道、skills。macOS/Linux 上会自动注册为系统服务（launchd/systemd），重启后自动拉起。

### Docker 部署

```bash
docker compose up -d
# docker-compose.yml 随仓库提供
```

### 从源码运行

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install
pnpm openclaw setup     # 首次运行
pnpm gateway:watch      # 开发模式，自动热重载
```

### 远程访问

推荐 Tailscale 或 VPN。不想折腾的话，一行 SSH 隧道也能搞定：

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

---

## 通道生态：22+ 平台覆盖

这是 OpenClaw 最实用的卖点。支持的通道列表：

| 类别 | 通道 |
|------|------|
| **国际 IM** | WhatsApp、Telegram、Discord、Slack、Signal、Google Chat、LINE |
| **国内 IM** | 微信、QQ、飞书 |
| **企业** | Microsoft Teams、Mattermost |
| **开放协议** | Matrix、IRC、Nostr |
| **其他** | iMessage、Nextcloud Talk、Synology Chat、Zalo、Twitch、WebChat |

对国内用户而言，**原生支持微信/QQ/飞书**是很多同类工具做不到的。大部分海外 Agent 工具根本不考虑这块。

---

## 亮点：真正让人眼前一亮的设计

### 1. 三层解耦的运行时架构

Provider / Model / Runtime 三层独立，允许在同一个 Agent 里混用不同运行时。这个设计不是拍脑袋来的，而是从实际需求演化出来的——OpenClaw 早期只支持自己的 Runtime，后来发现用户需要 Codex 的订阅体验、需要 Claude CLI 的编码能力，才逐步抽象出 Harness 层。

### 2. 持久化存储：只用 SQLite

VISION.md 里有个硬性规定：「Storage default: SQLite only. Do not add JSON/JSONL/TXT/sidecar files for OpenClaw-owned runtime state」。全局状态和插件 KV 数据放 `state/openclaw.sqlite`，Agent 专属状态放 `agents/<agentId>/agent/openclaw-agent.sqlite`。

没有 JSON 文件满天飞、没有 .env 散落各处、没有「if SQLite fails use JSON」的兼容地狱。对比一下大多数 Node.js 项目的配置管理现状，这是一股清流。

### 3. 设备配对 + 沙箱双保险

不在安全上「相信用户会配置好」，而是默认严格（DM 必须配对）、开放需要显式 opt-in（`dmPolicy="open"` + `allowFrom: ["*"]`）。沙箱的 deny-by-default 策略也很务实：关键工具（browser、gateway、cron）默认不在沙箱里可用。

### 4. doctor 命令的兼容性哲学

OpenClaw 有一个独特的规定：「Runtime reads canonical config only. No silent compat for old/malformed config keys」。但这不是粗暴地丢弃旧配置——每次 config schema 变更，必须同时提供 `openclaw doctor --fix` 的自动迁移脚本。

这是一个对长期维护非常友好的工程决策。用户升级不用手动改配置，`openclaw doctor` 自动检测、备份、迁移。

---

## 批评：说几个我认为的问题

### 1. 复杂度有明显门槛

安装简单（`openclaw onboard` 向导做得很用心），但深入使用后配置量不小。通道配置、Auth Profile、Runtime 选择、Sandbox 规则、Skills 管理——每个都有独立的配置逻辑。对于「我就想接个微信 AI 助手」的用户来说，学习曲线偏陡。

### 2. 文档结构有待改进

文档覆盖率高但组织方式偏「字典式」——你知道要找什么知道去哪，但不知道要找什么的时候容易迷失。架构文档分散在 `docs/concepts/`、`docs/gateway/`、`AGENTS.md`、`VISION.md` 等多个位置，缺少一篇「5 分钟看懂 OpenClaw」的总览。

### 3. 对非 Apple 平台的体验不一致

macOS 和 iOS 是「一等公民」（Voice Wake、Talk Mode、Canvas 全有），Windows Hub 功能偏少，Android Node 功能也弱于 iOS。这对 Windows 用户来说体验打了折扣。

### 4. 插件生态还在早期

ClawHub 插件市场上的质量参差不齐，很多插件的文档只有几行 README。虽然框架提供了很好的插件 API，但高质量、有维护的第三方插件数量还不够。

---

## 总结

OpenClaw 不是用来写代码的 Agent，也不是用来闲聊的套壳工具。它是一个**自托管的 AI 消息网关**，核心价值在于：

1. **通道聚合**：微信/QQ/飞书/Telegram/WhatsApp 统一接入一个 AI
2. **持久化 + 多 Agent**：你的 AI 助手永远在线，不会「失忆」
3. **运行时无关**：GPT / Claude / Codex / Copilot 随意切换
4. **安全可控**：配对 + 沙箱，数据全在你自己的设备上

适合什么人？自己部署一台常驻服务器 + 有多通道消息需求 + 对隐私有要求的技术用户。不适合只想开箱即用的普通用户。

我个人最欣赏的是它做架构决策时的「克制」——只做 Gateway，不往上堆产品层；强制 SQLite，不往文件系统上堆 JSON；坚持 config migration，不偷偷做兼容。这些决策加起来，就是一个长期可维护的项目该有的样子。

---

*本文基于 OpenClaw main 分支源码及官方文档分析，截至 2026 年 6 月。项目仓库：[github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)*
