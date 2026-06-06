---
title: OpenCLI：把任意网站变成命令行，AI Agent 的万能连接器
date: 2026-06-06 15:30:00
tags: [AI Agent, 开源, 浏览器自动化, CLI, MCP, 架构分析]
categories: AI
toc: true
---

> 23.6k Stars 的开源项目，用一行命令让 AI 操作任何已登录网站。本文从架构、原理、部署到使用，系统讲解 OpenCLI 的完整技术栈。

---

## 一、这是什么？

你在终端里用过 `curl api.bilibili.com` 的时候，有没有想过——要是每个网站都自带一个 CLI 就好了？

这就是 **OpenCLI** 做的事。它把任意网站包装成命令行工具，让你（或你的 AI Agent）用一行命令就能操作网站，而且用的是你 Chrome 浏览器里已经登录的账号——不需要 API Key，不需要二次登录，不需要维护 Cookie。

```
# 抓 B 站热门
opencli bilibili hot --limit 10 -f json

# 搜知乎
opencli zhihu search "如何评价特朗普" -f md

# 下载小红书笔记
opencli xiaohongshu download 640a1b2c000000001203abcd
```

截止 2026 年 6 月，这个项目在 GitHub 积累了 **23,600+ Stars**、1,343 次提交、107 个 Release、100+ 内建网站适配器，覆盖 B 站、知乎、小红书、Twitter/X、Reddit、LinkedIn、Amazon、HackerNews 等主流平台。

---

## 二、解决的痛点

AI Agent 操作网页长期以来存在三个结构性问题：

| 问题 | Browser Use 方案 | OpenCLI 方案 |
|---|---|---|
| Token 消耗 | 每次数万（整个 DOM 树） | 每次数百（结构化 JSON） |
| 反爬风险 | 高（headless 浏览器指纹明显） | 低（复用你的 Chrome 登录态 + 反侦测） |
| 输出确定性 | 不稳定的 HTML 片段 | JSON/YAML/CSV/Markdown，schema 稳定 |

一个典型的内容研究任务（搜 5 个平台，提取 20 篇内容），Token 消耗从 15,000-20,000 降到 3,000-5,000，**成本降低七成以上**。

---

## 三、系统架构

OpenCLI 由四个核心组件组成：

```
┌─────────────────┐     HTTP      ┌───────────────┐     WS      ┌─────────────────┐
│   CLI 入口       │ ───────────── │  Daemon 守护   │ ─────────→ │  Chrome Extension│
│  (opencli 命令)  │  POST /cmd    │  (Port 19825)  │  CDP cmd   │  (Manifest V3)   │
└────────┬────────┘               └──────┬────────┘            └────────┬────────┘
         │                               │                              │
         ▼                               │                     ┌────────▼────────┐
┌─────────────────┐                       │                     │   已登录的 Chrome  │
│  适配器 (clis/) │                       │                     │  (复用登录态)     │
│  100+ JS 文件   │                       │                     └────────┬────────┘
└────────┬────────┘                       │                              │
         │                               │                              │
         └───────────────────────────────┴──────────────────────────────┘
                                   执行流程：
                adapter → pipeline → CLI → daemon → extension → Chrome → 网站
```

### 3.1 CLI 入口

一个 Node.js CLI 程序（`npm install -g @jackwener/opencli`），负责：
- 解析命令参数（`opencli <site> <command> [args]`）
- 发现和加载适配器（优先读预编译的 `cli-manifest.json`，fallback 到文件系统扫描）
- 执行 pipeline，通过 daemon 跟浏览器交互
- 格式化输出（支持 table/json/yaml/csv/md 五种格式）

关键优化：**快速路径**。`--version`、`completion` 等高频轻量命令不触发适配器发现，直接返回，启动几乎零延迟。

### 3.2 Daemon 守护进程

一个持久化运行的 HTTP + WebSocket 服务器，监听 `localhost:19825`：

```text
CLI → HTTP POST /command → Daemon → WebSocket → Extension
Extension → WebSocket result → Daemon → HTTP response → CLI
```

设计亮点：

- **auto-spawn**：首次浏览器命令时自动启动，之后持续存活
- **profile 隔离**：支持多 Chrome profile，每个 profile 维护独立的 WS 连接
- **安全五层**：Origin 检查 + 自定义 Header（X-OpenCLI）+ CORS 拒绝 + 1MB body 限制 + WS verifyClient
- **pending 队列**：命令 → Extension 的请求-响应配对通过 Map 管理，超时自动清理

### 3.3 Chrome Extension

Manifest V3 扩展，通过 `debugger` 权限获取 CDP 连接：

```json
{
  "permissions": ["debugger", "tabs", "cookies", "activeTab", "downloads"],
  "host_permissions": ["<all_urls>"],
  "background": { "service_worker": "dist/background.js" }
}
```

关键设计：
- 后台 service worker 保持 WS 连接到 daemon
- 收到 daemon 命令后通过 CDP 操作页面
- **tab lease 机制**：每个命令租用一个浏览器 tab，执行完释放，避免 tab 泄漏
- 支持 foreground/background 窗口模式

### 3.4 适配器系统

适配器是 OpenCLI 的灵魂。每个网站一个目录，每个命令一个 JS 文件：

```javascript
// clis/bilibili/hot.js
import { cli } from '@jackwener/opencli/registry';
cli({
    site: 'bilibili',
    name: 'hot',
    access: 'read',
    description: 'B站热门视频',
    domain: 'www.bilibili.com',
    args: [
        { name: 'limit', type: 'int', default: 20 }
    ],
    columns: ['rank', 'title', 'author', 'play'],
    pipeline: [
        { navigate: 'https://www.bilibili.com' },
        { evaluate: `(async () => {
            const res = await fetch('https://api.bilibili.com/x/web-interface/popular?...');
            const data = await res.json();
            return (data?.data?.list || []).map(item => ({...}));
        })()` },
        { map: { rank: '${{ index + 1 }}', title: '${{ item.title }}' } },
        { limit: '${{ args.limit }}' },
    ],
});
```

P​ipeline 是声明的步骤序列，引擎按顺序执行：

| Step | 功能 |
|---|---|
| `navigate` | 打开 URL（复用登录态） |
| `evaluate` | 在页面中执行 JS（拦截 API 调用/提取 DOM） |
| `fetch` | 发起 HTTP 请求（带 cookie） |
| `intercept` | 拦截网络请求的响应 |
| `transform` | 数据变换/格式化 |
| `map` | 字段映射 + 模板变量 `${{ }}` |
| `limit` | 限制输出数量 |

这种声明式设计意味着：**新增一个网站支持，大多数情况只需写一个 JS 文件，不需要碰核心代码**。

### 3.5 五种认证策略

| 策略 | 说明 | 适用场景 |
|---|---|---|
| `PUBLIC` | 直接请求公开 API | HackerNews、V2EX |
| `LOCAL` | 无浏览器，纯本地操作 | 文件处理 |
| `COOKIE` | 复用 Chrome 登录态 | 知乎、Bilibili |
| `INTERCEPT` | CDP 拦截页面 API 请求 | Twitter/X、Reddit |
| `UI` | 模拟点击/输入操作 | 需要交互的网站 |

---

## 四、三大核心机制

### 4.1 反侦测（Stealth）

OpenCLI 的 stealth 模块是有 350 行 JS 的静默补丁，在页面脚本执行前注入：

```
navigator.webdriver → false
window.chrome → 伪造 runtime/loadTimes/csi
navigator.plugins → 五个假 PDF 插件
window.outerWidth/outerHeight → 1920×1080
Error stack trace → 移除 CDP 相关帧
```

关键细节：**guard flag 藏在 EventTarget.prototype 的非枚举属性里**——不会被普通指纹检测代码扫描到，同时避免了跨 CDP 评估的双重注入。

### 4.2 自修复协议（Self-Repair）

这是 OpenCLI 最「智能」的设计。当 AI Agent 运行 `opencli <site> <command>` 失败时：

```text
1. 自动重新执行：opencli ... --trace retain-on-failure
2. 读取 trace 产物的 summary.md，获取 adapterSourcePath
3. 分析错误原因（failed network / 变化的 DOM / 变的 API）
4. 直接编辑适配器文件修复
5. 重新运行原命令
6. 最多重复 3 轮，失败则报告
7. 成功后自动建议向 jackwener/OpenCLI 提交 upstream issue
```

这套协议的哲学是：**命令本身就是 spec**。不需要文档、不需要测试用例。跑一遍看输出来判断修没修好。

修复范围有严格约束：只改 `clis/<site>/*.js`，不能动 `src/`、`extension/`、`tests/`。

### 4.3 分层加载（Token 效率）

这是相比 MCP Server 的核心优势：

| 方式 | 初始 Token 占用 | 每新增工具 |
|---|---|---|
| MCP 全量加载 | 5,000-15,000+ | +100-300 |
| Skill 全量加载 | 3,000-10,000+ | +200-500 |
| OpenCLI 分层加载 | **约 50**（一行指令） | 按需 200-500 |

工作流：
1. Agent 执行 `opencli list` 拿到命令清单（~300 Token）
2. 确认需要的网站后，执行 `opencli <site> --help` 看参数
3. 执行具体命令，context window 只保留当前任务相关数据

---

## 五、部署安装

### 系统要求

- Node.js >= 20
- Chrome/Chromium 浏览器
- 目标网站需先在 Chrome 中登录

### 步骤 1：安装 CLI

```bash
node --version  # 确保 >= 20
npm install -g @jackwener/opencli
```

### 步骤 2：安装浏览器扩展

从 GitHub Releases 下载 `opencli-extension.zip`，解压后在 `chrome://extensions` 开启开发者模式，加载解压后的文件夹。

### 步骤 3：诊断验证

```bash
opencli doctor
```

### 步骤 4：集成到 AI Agent

```bash
# Claude Code / Codex 等
npx skills add jackwener/opencli
```

### 常用命令速查

```bash
# 列出所有可用网站
opencli list

# 查看某网站支持的命令
opencli bilibili --help

# 执行命令（默认表格输出）
opencli hackernews top --limit 5

# JSON 输出，可以 pipe 给 jq 或 LLM
opencli bilibili hot -f json | jq '.[0].title'

# 搜索
opencli zhihu search "量化交易"

# 下载
opencli xiaohongshu download <note_id>
opencli twitter download elonmusk --limit 20
```

### 高级功能

```bash
# 多 Chrome Profile 管理
opencli profile list
opencli profile rename <id> work

# 创建自定义适配器
opencli plugin create

# 注册本地工具到 CLI Hub
opencli external register gh  # 注册 GitHub CLI

# 浏览器探索模式（自动分析 API 结构）
opencli browser init <site>/<command>
```

---

## 六、技术实力 vs 短板

### 做对了什么

1. **架构解耦**：CLI / Daemon / Extension 三层完全独立，任何一层出问题不影响其他
2. **适配器声明式**：写适配器就是写 pipeline，不需要懂 CDP、不需要懂 Node.js 架构
3. **反侦测用心**：不是简单的 `navigator.webdriver = false`，而是系统性的 10+ 层补丁
4. **自修复协议**：AI Agent 时代的正确设计——不是写死逻辑，而是给 Agent 工具让它自己修
5. **Token 效率**：分层加载设计天然适合 Agent 场景
6. **安全默认严格**：daemon 五层防护，扩展不在 Chrome Web Store 时给出明确警告

### 短板和风险

1. **Node.js 依赖性**：`npm install -g` 是唯一安装路径，没有独立 binary
2. **适配器覆盖率**：100+ 网站看似多，但相对整个互联网是沧海一粟。大部分中小网站需要自己写适配器
3. **CAPTCHA 无解**：遇到验证码只能报告，没有任何绕过手段
4. **风控严格的平台**：银行、政府网站的风险不可忽视
5. **Extension 权限范围大**：`debugger` + `<all_urls>` + `cookies`，在敏感 Chrome profile 上安装需要谨慎

---

## 七、与同类项目的对比

| | OpenCLI | Browser Use | Puppeteer | MCP Server |
|---|---|---|---|---|
| 定位 | 网站→CLI + Agent 工具 | 纯浏览器操作 | 浏览器自动化 | 通用工具服务 |
| 登录态 | 复用 Chrome 已登录 | 需单独处理 | 需单独处理 | 不涉及 |
| Token 消耗 | 极低 | 极高 | — | 中等 |
| 结构化输出 | 5 种格式 | 需额外处理 | 需额外处理 | JSON |
| 反侦测 | 10+ 层 stealth | 基础 | 无 | 不涉及 |
| 自修复 | 有 | 无 | 无 | 无 |
| 安装 | `npm i -g` | `pip install` | `npm i` | 各种 |

**本质上，OpenCLI 是介于"裸浏览器自动化"和"API 集成"之间的一个最优折衷**——既有浏览器的灵活性（任何网站都能操作），又有 API 的确定性（结构化输出）。

---

## 八、我的评价

OpenCLI 解决了一个真实且普遍的问题：**AI Agent 需要操作网页，但目前的方案要么 Token 成本太高（Browser Use），要么需要每个网站单独接入 API**。

它的核心设计哲学是"复用已有的"——复用 Chrome 登录态、复用 CDP 协议、复用 JavaScript 生态。这让它不需要重新发明轮子，但需要处理大量的边缘情况（反侦测、网站结构变化、认证策略差异）。

自修复协议是最让我惊艳的部分。它不是一个写死的"如果 X 就 Y"的规则引擎，而是一个给 AI Agent 的开放工具——trace 产物、错误信封、适配器路径、重试预算，全部交出去让 Agent 自己决策。这是一种"为 AI 设计"的思路，而不是"把 AI 塞进现有框架"。

当然，它也不是万能钥匙。遇到需要两步验证的网站、有严格风控的平台、需要复杂交互的 UI 流程，OpenCLI 一样无能为力。但它在自己覆盖的范围内（内容搜索、数据提取、重复性浏览任务），效率碾压 Browser Use 方案。

**适合的场景**：AI Agent 的内容研究、自动化数据收集、重复性网页操作（每天抓取热搜、自动下载文章等）。

**不适合的场景**：需要 CAPTCHA 绕过的任务、银行/支付等安全要求高的操作、需要复杂交互的工作流。

---

*参考链接*
- GitHub: [jackwener/OpenCLI](https://github.com/jackwener/opencli)
- AutoCLI（Rust 重写版）: [nashsu/opencli-rs](https://github.com/nashsu/opencli-rs)
- 官方文档: [opencli.cn](https://opencli.cn)
