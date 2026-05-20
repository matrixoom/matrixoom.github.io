---
title: "华为开源JiuwenSwarm：蜂群智能体凭什么能打败单Agent范式"
date: 2026-05-20 09:30:00
tags:
  - AI Agent
  - 多智能体
  - 华为
  - 开源
  - 蜂群智能
categories: 项目分析
---

> 2026年5月18日，华为2012实验室和华为云AgentArts团队联合支持的openJiuwen社区，发布并开源了JiuwenSwarm。两天之内GitHub Star破700、Fork过130，AI圈迅速沸腾。这篇文章拉了源码下来，从代码层面讲清楚它到底做了什么、为什么这么设计、值不值得你关注。

---

## 先说背景：为什么是"Swarm"

单智能体（Single Agent）的上限，行业现在已经看清楚了。给一个LLM配上工具、记忆、任务规划，它能干很多事，但碰到"需要多个专家协作才能搞定"的场景——比如写代码需要架构设计+实现+测试+代码审查——单Agent的瓶颈就出来了。

openJiuwen把AI Agent工程范式的演化分成了四个阶段：

| 阶段 | 名称 | 核心关注点 |
|------|------|-----------|
| 第一阶段 | Prompt Engineering | 调提示词，让模型理解意图 |
| 第二阶段 | Context Engineering | 组织上下文、记忆、工具、状态 |
| 第三阶段 | Harness Engineering | 单Agent工程化、轨迹管理、错误恢复 |
| **第四阶段** | **Coordination Engineering** | **多Agent协同工程化** |

JiuwenSwarm就是在第四阶段上押注的产物。它提出了一个名叫"蜂群智能"的比喻：不是一个全能AI，而是一群各有分工、协同配合的Agent像蜜蜂一样工作。

---

## 系统架构总览

<svg viewBox="0 0 680 500" width="100%" role="img" aria-label="JiuwenSwarm系统架构总览">
<title>JiuwenSwarm 系统架构总览</title>
<defs>
  <marker id="jsarr" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M2 1L8 5L2 9" fill="none" stroke="context-stroke" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker>
</defs>
<style>.jt{font-family:sans-serif;font-size:13px;fill:#1a1a1a}.jts{font-family:sans-serif;font-size:11px;fill:#555}.jth{font-family:sans-serif;font-size:14px;font-weight:600;fill:#1a1a1a}.jthw{font-family:sans-serif;font-size:13px;font-weight:600}</style>
<rect x="0" y="0" width="680" height="500" fill="#FAFAFA"/>
<text class="jth" x="340" y="28" text-anchor="middle" font-size="16px">JiuwenSwarm 系统架构总览</text>
<!-- 接入层 -->
<rect x="30" y="46" width="620" height="72" rx="10" fill="#E3F2FD" stroke="#1565C0" stroke-width="1.2"/>
<text class="jthw" x="340" y="66" text-anchor="middle" fill="#1565C0">接入层（Channels）</text>
<rect x="44" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="89" y="95" text-anchor="middle">Web前端</text>
<rect x="144" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="189" y="95" text-anchor="middle">小艺（华为）</text>
<rect x="244" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="289" y="95" text-anchor="middle">飞书/Discord</text>
<rect x="344" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="389" y="95" text-anchor="middle">WhatsApp</text>
<rect x="444" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="489" y="95" text-anchor="middle">TUI终端</text>
<rect x="544" y="74" width="90" height="32" rx="6" fill="#fff" stroke="#1565C0" stroke-width="0.8"/>
<text class="jt" x="589" y="95" text-anchor="middle">A2X协议</text>
<!-- 箭头 -->
<line x1="340" y1="118" x2="340" y2="138" stroke="#888" stroke-width="1.5" marker-end="url(#jsarr)"/>
<!-- 核心层 -->
<rect x="30" y="140" width="620" height="210" rx="10" fill="#FFF8E1" stroke="#F57F17" stroke-width="1.2"/>
<text class="jthw" x="340" y="162" text-anchor="middle" fill="#F57F17">核心层（Harness + Team）</text>
<!-- Harness模块 -->
<rect x="44" y="170" width="180" height="168" rx="8" fill="#FFECB3" stroke="#F57F17" stroke-width="0.8"/>
<text class="jthw" x="134" y="190" text-anchor="middle" font-size="12px" fill="#E65100">单Agent Harness</text>
<rect x="54" y="198" width="160" height="26" rx="5" fill="#fff" stroke="#F57F17" stroke-width="0.5"/>
<text class="jts" x="134" y="215" text-anchor="middle">PLAN / AGENT / CODE 模式</text>
<rect x="54" y="230" width="160" height="26" rx="5" fill="#fff" stroke="#F57F17" stroke-width="0.5"/>
<text class="jts" x="134" y="247" text-anchor="middle">Memory（BM25+向量双路检索）</text>
<rect x="54" y="262" width="160" height="26" rx="5" fill="#fff" stroke="#F57F17" stroke-width="0.5"/>
<text class="jts" x="134" y="279" text-anchor="middle">Rail插件系统（权限/打断/提示）</text>
<rect x="54" y="294" width="160" height="26" rx="5" fill="#fff" stroke="#F57F17" stroke-width="0.5"/>
<text class="jts" x="134" y="311" text-anchor="middle">AutoHarness（自动进化流水线）</text>
<!-- Team模块 -->
<rect x="244" y="170" width="190" height="168" rx="8" fill="#FFE0B2" stroke="#E65100" stroke-width="0.8"/>
<text class="jthw" x="339" y="190" text-anchor="middle" font-size="12px" fill="#BF360C">Team Swarm</text>
<rect x="254" y="198" width="170" height="26" rx="5" fill="#fff" stroke="#E65100" stroke-width="0.5"/>
<text class="jts" x="339" y="215" text-anchor="middle">TeamAgent（Leader+Teammate）</text>
<rect x="254" y="230" width="170" height="26" rx="5" fill="#fff" stroke="#E65100" stroke-width="0.5"/>
<text class="jts" x="339" y="247" text-anchor="middle">分布式运行时（ZMQ/PostgreSQL）</text>
<rect x="254" y="262" width="170" height="26" rx="5" fill="#fff" stroke="#E65100" stroke-width="0.5"/>
<text class="jts" x="339" y="279" text-anchor="middle">Skill同步（成员间共享技能目录）</text>
<rect x="254" y="294" width="170" height="26" rx="5" fill="#fff" stroke="#E65100" stroke-width="0.5"/>
<text class="jts" x="339" y="311" text-anchor="middle">SkillEvolutionRail（技能演进轨道）</text>
<!-- 安全模块 -->
<rect x="454" y="170" width="180" height="168" rx="8" fill="#FCE4EC" stroke="#AD1457" stroke-width="0.8"/>
<text class="jthw" x="544" y="190" text-anchor="middle" font-size="12px" fill="#880E4F">JiuwenBox（安全沙箱）</text>
<rect x="464" y="198" width="160" height="26" rx="5" fill="#fff" stroke="#AD1457" stroke-width="0.5"/>
<text class="jts" x="544" y="215" text-anchor="middle">InferencePrivacyProxy（推理代理）</text>
<rect x="464" y="230" width="160" height="26" rx="5" fill="#fff" stroke="#AD1457" stroke-width="0.5"/>
<text class="jts" x="544" y="247" text-anchor="middle">PolicyEngine（策略引擎）</text>
<rect x="464" y="262" width="160" height="26" rx="5" fill="#fff" stroke="#AD1457" stroke-width="0.5"/>
<text class="jts" x="544" y="279" text-anchor="middle">SandboxManager（隔离执行）</text>
<rect x="464" y="294" width="160" height="26" rx="5" fill="#fff" stroke="#AD1457" stroke-width="0.5"/>
<text class="jts" x="544" y="311" text-anchor="middle">AuditLogger（操作审计）</text>
<!-- 箭头 -->
<line x1="340" y1="350" x2="340" y2="370" stroke="#888" stroke-width="1.5" marker-end="url(#jsarr)"/>
<!-- 扩展层 -->
<rect x="30" y="372" width="620" height="72" rx="10" fill="#E8F5E9" stroke="#2E7D32" stroke-width="1.2"/>
<text class="jthw" x="340" y="393" text-anchor="middle" fill="#2E7D32">扩展层（Extensions + Skill Hub）</text>
<rect x="44" y="402" width="130" height="30" rx="6" fill="#fff" stroke="#2E7D32" stroke-width="0.8"/>
<text class="jts" x="109" y="422" text-anchor="middle">Extension插件系统</text>
<rect x="186" y="402" width="130" height="30" rx="6" fill="#fff" stroke="#2E7D32" stroke-width="0.8"/>
<text class="jts" x="251" y="422" text-anchor="middle">MCP服务器集成</text>
<rect x="328" y="402" width="130" height="30" rx="6" fill="#fff" stroke="#2E7D32" stroke-width="0.8"/>
<text class="jts" x="393" y="422" text-anchor="middle">Swarm Skills Hub</text>
<rect x="470" y="402" width="165" height="30" rx="6" fill="#fff" stroke="#2E7D32" stroke-width="0.8"/>
<text class="jts" x="552" y="422" text-anchor="middle">华为云MaaS / 小艺平台</text>
<!-- 底部说明 -->
<text class="jts" x="340" y="470" text-anchor="middle" fill="#888">Python 76.5% | TypeScript 21.2% | Apache-2.0 开源</text>
<text class="jts" x="340" y="488" text-anchor="middle" fill="#888">GitHub: openJiuwen-ai/jiuwenswarm | ⭐ 736  🍴 130（2026-05-20）</text>
</svg>

---

## 四大核心设计，逐一拆开看

### 一、Team Swarm——多Agent协调的工程实现

这是整个系统最核心的部分，也是"蜂群"名字的来源。

`team_manager.py` 里可以看到，TeamManager维护着一套完整的Agent团队生命周期：Leader负责任务规划和分配，Teammate负责各自职责范围内的执行，跨成员通信走ZMQ或内存总线（pyzmq transport）。

```python
# team_manager.py 核心摘录
class TeamManager:
    """Manage team instances across sessions."""
    def __init__(self):
        self._team_agents: dict[str, TeamAgent] = {}
        self._team_skill_rails: dict[str, Any] = {}         # 技能演进轨道
        self._team_member_skill_evolution_rails: dict[str, list[Any]] = {}
        self._team_skill_create_rails: dict[str, Any] = {}  # 团队技能创建轨道
        self._team_evolution_watchers: dict[str, asyncio.Task] = {}  # 自动扫描演进
```

几个值得注意的细节：

**技能在成员间同步**：`_sync_skills_dir()` 把一个Agent跑通的技能（一个包含`SKILL.md`的目录）复制到其他成员，确保团队成员共享最新实践：

```python
def _sync_skills_dir(source: Path, target: Path) -> None:
    for skill_dir in source.iterdir():
        if not skill_dir.is_dir() or not (skill_dir / "SKILL.md").is_file():
            continue
        dest = target / skill_dir.name
        if dest.exists():
            shutil.rmtree(dest)
        shutil.copytree(skill_dir, dest)
```

**分布式模式**：`distributed_runtime.py` 支持多机部署，通过PostgreSQL做持久化存储、ZMQ做消息传输，Leader和Teammate的角色在配置文件里通过`runtime.role`字段区分：

```python
def runtime_role(config_base: dict) -> str:
    role = str(runtime_cfg.get("role", "leader")).strip().lower()
    return role if role in ("leader", "teammate") else "leader"
```

**模型路由**：不同角色的Agent可以用不同的模型——Leader用强推理模型做规划，Teammate可以用小模型做执行，降低成本。

---

### 二、记忆系统——双路检索，从SQLite跑出向量数据库效果

单Agent的记忆系统藏在 `memory/manager.py` 里，设计相当扎实。它同时维护两套索引：

- **FTS（全文检索）**：基于SQLite FTS5，速度快，擅长精确关键词匹配
- **向量检索（Vector）**：把对话/文件内容embedding后存成二进制BLOB，支持语义相似度检索

```python
# 向量存储就是一个packed float列表，不依赖外部向量数据库
def vector_to_blob(embedding: List[float]) -> bytes:
    return struct.pack(f'{len(embedding)}f', *embedding)

def blob_to_vector(blob: bytes) -> List[float]:
    count = len(blob) // 4
    return list(struct.unpack(f'{count}f', blob))
```

这个设计的亮点在于：**零外部依赖**。不需要Chroma、Weaviate或者Pinecone，纯SQLite就实现了混合检索，对本地部署非常友好。

记忆管理器还引入了会话增量追踪（`SessionDeltaState`），只对有变化的会话片段重新索引，不会每次都全量重建，性能开销很小。

---

### 三、技能自演进——SkillEvolutionRail轨道机制

这是JiuwenSwarm最有意思的设计，也是区别于一般Agent框架的地方。

框架里有一个叫Rail（轨道）的插件机制，它不是普通的回调，而是可以在Agent执行流程的特定时机介入、修改行为的钩子。`SkillEvolutionRail`、`TeamSkillEvolutionRail`、`TeamSkillCreateRail` 这几个Rail共同构成了技能演进管道。

工作原理是这样的：

<svg viewBox="0 0 680 300" width="100%" role="img" aria-label="技能自演进流程">
<title>JiuwenSwarm 技能自演进（Skill Self-Evolution）流程</title>
<defs>
  <marker id="evarr" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path d="M2 1L8 5L2 9" fill="none" stroke="#555" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></marker>
</defs>
<style>.et{font-family:sans-serif;font-size:12px;fill:#1a1a1a}.ets{font-family:sans-serif;font-size:10px;fill:#555}.eth{font-family:sans-serif;font-size:13px;font-weight:600}</style>
<rect x="0" y="0" width="680" height="300" fill="#FAFAFA"/>
<text class="eth" x="340" y="24" text-anchor="middle">Skill Self-Evolution 流程</text>
<!-- 步骤框 -->
<rect x="20" y="44" width="100" height="56" rx="8" fill="#E3F2FD" stroke="#1565C0" stroke-width="1.2"/>
<text class="et" x="70" y="67" text-anchor="middle">用户发起</text>
<text class="et" x="70" y="84" text-anchor="middle">任务</text>
<line x1="120" y1="72" x2="148" y2="72" stroke="#555" stroke-width="1.5" marker-end="url(#evarr)"/>
<rect x="148" y="44" width="110" height="56" rx="8" fill="#FFF8E1" stroke="#F57F17" stroke-width="1.2"/>
<text class="et" x="203" y="63" text-anchor="middle">Team Agent</text>
<text class="et" x="203" y="80" text-anchor="middle">执行任务</text>
<text class="ets" x="203" y="95" text-anchor="middle">（记录执行轨迹）</text>
<line x1="258" y1="72" x2="286" y2="72" stroke="#555" stroke-width="1.5" marker-end="url(#evarr)"/>
<rect x="286" y="44" width="120" height="56" rx="8" fill="#E8F5E9" stroke="#2E7D32" stroke-width="1.2"/>
<text class="et" x="346" y="63" text-anchor="middle">EvolutionRail</text>
<text class="et" x="346" y="80" text-anchor="middle">分析轨迹模式</text>
<text class="ets" x="346" y="95" text-anchor="middle">（自动扫描watchers）</text>
<line x1="406" y1="72" x2="434" y2="72" stroke="#555" stroke-width="1.5" marker-end="url(#evarr)"/>
<rect x="434" y="44" width="110" height="56" rx="8" fill="#FCE4EC" stroke="#AD1457" stroke-width="1.2"/>
<text class="et" x="489" y="63" text-anchor="middle">生成/更新</text>
<text class="et" x="489" y="80" text-anchor="middle">SKILL.md</text>
<line x1="544" y1="72" x2="572" y2="72" stroke="#555" stroke-width="1.5" marker-end="url(#evarr)"/>
<rect x="572" y="44" width="88" height="56" rx="8" fill="#EDE7F6" stroke="#7B1FA2" stroke-width="1.2"/>
<text class="et" x="616" y="63" text-anchor="middle">跨成员</text>
<text class="et" x="616" y="80" text-anchor="middle">同步技能</text>
<!-- 两个演进层说明 -->
<rect x="20" y="130" width="310" height="120" rx="10" fill="#FFF3E0" stroke="#E65100" stroke-width="1"/>
<text class="eth" x="175" y="152" text-anchor="middle" font-size="12px" fill="#E65100">团队层演进</text>
<text class="ets" x="175" y="172" text-anchor="middle">• 自动新增/删除角色</text>
<text class="ets" x="175" y="190" text-anchor="middle">• 补充任务约束规则</text>
<text class="ets" x="175" y="208" text-anchor="middle">• 优化Leader的任务规划策略</text>
<text class="ets" x="175" y="226" text-anchor="middle">• 沉淀最优角色搭配 → Swarm Skill</text>
<rect x="350" y="130" width="310" height="120" rx="10" fill="#E8F5E9" stroke="#2E7D32" stroke-width="1"/>
<text class="eth" x="505" y="152" text-anchor="middle" font-size="12px" fill="#2E7D32">成员层演进</text>
<text class="ets" x="505" y="172" text-anchor="middle">• 记录工具报错和接口超时</text>
<text class="ets" x="505" y="190" text-anchor="middle">• 沉淀调用技巧到个人技能库</text>
<text class="ets" x="505" y="208" text-anchor="middle">• 避免重复踩坑</text>
<text class="ets" x="505" y="226" text-anchor="middle">• 成员技能上传Hub共享</text>
<!-- 注释 -->
<text class="ets" x="340" y="274" text-anchor="middle" fill="#888">AutoHarness中的 EXTENDED_EVOLVE_PIPELINE 驱动整个演进引擎</text>
<text class="ets" x="340" y="290" text-anchor="middle" fill="#888">源码：jiuwenswarm/agents/harness/common/auto_harness/service.py</text>
</svg>

从源码可以看到，AutoHarness里导入了一个叫 `EXTENDED_EVOLVE_PIPELINE` 的流水线：

```python
from openjiuwen.auto_harness.pipelines import EXTENDED_EVOLVE_PIPELINE
from openjiuwen.auto_harness.pipelines.extended_evolve_pipeline import ExtensionTaskPipeline
from openjiuwen.auto_harness.stages.activate import ExtendActivateStage
```

这个流水线负责把每次任务执行的轨迹消化成可复用的技能，写成`SKILL.md`格式，存到技能目录里。

---

### 四、JiuwenBox——不是功能，是安全护城河

源码里有一个叫 `jiuwenbox/` 的子项目，容易被忽略，但它才是企业落地的关键。

JiuwenBox本质是一个**本地化的推理隐私代理**，它在Agent调用LLM之前拦截请求，做三件事：

1. **推理隐私代理（InferencePrivacyProxy）**：把敏感数据（身份证号、手机号等）在发给LLM之前替换成占位符，返回后再还原。
2. **策略引擎（PolicyEngine）**：用配置文件定义哪些操作允许、哪些需要审批、哪些直接拒绝。
3. **沙箱隔离（SandboxManager）**：Agent执行代码在隔离环境里跑，防止越权。

```python
# jiuwenbox/src/jiuwenbox/proxy/inference_privacy_proxy.py
# 在推理前替换敏感字段，确保原始数据不离开本地环境
```

这个设计解决了企业落地最大的顾虑：**Agent用的数据不能裸奔出去**。

---

## 两种人机协作模式：HOTS vs HITS

这两个模式不只是文档里的概念，在代码里有对应的实现：

| 模式 | 名称 | 你的角色 | 控制粒度 |
|------|------|---------|---------|
| **HOTS** | Human on the Swarm | 上帝视角指挥官 | 全局调度、优先级调整、中途换人 |
| **HITS** | Human in the Swarm | 团队成员之一 | 与Agent同场协作、实时推演 |

HOTS模式适合"我想掌控全局"的场景，比如指挥一个软件开发团队，告诉Agent们大方向，自己审核关键节点。HITS模式适合"我想深度参与"的场景，比如真正和Agent共同做研究，而不是等结果。

---

## 评测数据值得认真看

发布时官方给出了两组评测数据：

**PinchBench（Agent综合能力基准）**——覆盖代码开发、创意写作、文档处理、会议管理等场景：

| 指标 | JiuwenSwarm | OpenClaw |
|------|------------|---------|
| 综合得分 | **94.2%（SOTA）** | 91.6% |
| Token消耗 | **降低34.8%** | 基准 |

**LOCOMO（长期记忆评测）**：
- 记忆准确率**85%**，使用8B参数模型就能实现，优于业界主流方案。

Token消耗降34.8%这个数字是最实在的——多Agent系统最大的成本问题就是上下文膨胀，这个指标直接说明了它在上下文管理上做了认真的工程优化。

---

## 怎么用：五分钟跑起来

```bash
# 安装
pip install jiuwenswarm

# 初始化（首次）
jiuwenswarm-init

# 启动，访问 http://localhost:5173
jiuwenswarm-start

# 可选：终端UI版本
pip install jiuwenswarm-tui
jiuwenswarm-tui
```

用之前记得备份三个目录（升级时会初始化，不备份会丢数据）：

```
.jiuwenswarm/workspace/agent/memory   # 所有对话记忆
.jiuwenswarm/workspace/agent/skills   # 自定义技能
.jiuwenswarm/config                   # 配置文件
```

---

## 应用场景：哪些地方真的适合用

<svg viewBox="0 0 680 340" width="100%" role="img" aria-label="JiuwenSwarm应用场景矩阵">
<title>JiuwenSwarm 应用场景矩阵</title>
<style>.st{font-family:sans-serif;font-size:12px;fill:#1a1a1a}.sts{font-family:sans-serif;font-size:11px;fill:#555}.sth{font-family:sans-serif;font-size:13px;font-weight:600}</style>
<rect x="0" y="0" width="680" height="340" fill="#FAFAFA"/>
<text class="sth" x="340" y="26" text-anchor="middle" font-size="15px">JiuwenSwarm 典型应用场景</text>
<!-- 6个场景卡片 -->
<rect x="20" y="44" width="196" height="110" rx="10" fill="#E3F2FD" stroke="#1565C0" stroke-width="1"/>
<text class="sth" x="118" y="68" text-anchor="middle" font-size="12px" fill="#1565C0">💻 复杂软件工程</text>
<text class="sts" x="118" y="88" text-anchor="middle">需求分析Agent</text>
<text class="sts" x="118" y="104" text-anchor="middle">+架构设计Agent</text>
<text class="sts" x="118" y="120" text-anchor="middle">+代码实现Agent</text>
<text class="sts" x="118" y="136" text-anchor="middle">+测试/代码审查Agent</text>
<rect x="242" y="44" width="196" height="110" rx="10" fill="#E8F5E9" stroke="#2E7D32" stroke-width="1"/>
<text class="sth" x="340" y="68" text-anchor="middle" font-size="12px" fill="#2E7D32">🏥 医疗多学科会诊</text>
<text class="sts" x="340" y="88" text-anchor="middle">分诊Agent</text>
<text class="sts" x="340" y="104" text-anchor="middle">+23专科医学专家Agent</text>
<text class="sts" x="340" y="120" text-anchor="middle">+动态创建专科成员</text>
<text class="sts" x="340" y="136" text-anchor="middle">+联合会诊求同存异</text>
<rect x="464" y="44" width="196" height="110" rx="10" fill="#FFF3E0" stroke="#E65100" stroke-width="1"/>
<text class="sth" x="562" y="68" text-anchor="middle" font-size="12px" fill="#E65100">🎬 内容批量生产</text>
<text class="sts" x="562" y="88" text-anchor="middle">脚本创作Agent</text>
<text class="sts" x="562" y="104" text-anchor="middle">+素材搜索Agent</text>
<text class="sts" x="562" y="120" text-anchor="middle">+标题文案Agent</text>
<text class="sts" x="562" y="136" text-anchor="middle">+跑完自动沉淀技能</text>
<rect x="20" y="174" width="196" height="110" rx="10" fill="#FCE4EC" stroke="#AD1457" stroke-width="1"/>
<text class="sth" x="118" y="198" text-anchor="middle" font-size="12px" fill="#AD1457">📚 多学科教育辅导</text>
<text class="sts" x="118" y="218" text-anchor="middle">语文/数学/理化各科Agent</text>
<text class="sts" x="118" y="234" text-anchor="middle">+学情评估Agent</text>
<text class="sts" x="118" y="250" text-anchor="middle">学生/家长身份切换</text>
<text class="sts" x="118" y="266" text-anchor="middle">（HITS沉浸模式）</text>
<rect x="242" y="174" width="196" height="110" rx="10" fill="#EDE7F6" stroke="#7B1FA2" stroke-width="1"/>
<text class="sth" x="340" y="198" text-anchor="middle" font-size="12px" fill="#7B1FA2">⚙️ 企业自动化流程</text>
<text class="sts" x="340" y="218" text-anchor="middle">数据收集Agent</text>
<text class="sts" x="340" y="234" text-anchor="middle">+分析决策Agent</text>
<text class="sts" x="340" y="250" text-anchor="middle">+推送执行Agent</text>
<text class="sts" x="340" y="266" text-anchor="middle">JiuwenBox保障数据安全</text>
<rect x="464" y="174" width="196" height="110" rx="10" fill="#E0F7FA" stroke="#006064" stroke-width="1"/>
<text class="sth" x="562" y="198" text-anchor="middle" font-size="12px" fill="#006064">🔬 AI辅助科研</text>
<text class="sts" x="562" y="218" text-anchor="middle">文献调研Agent</text>
<text class="sts" x="562" y="234" text-anchor="middle">+实验设计Agent</text>
<text class="sts" x="562" y="250" text-anchor="middle">+数据分析Agent</text>
<text class="sts" x="562" y="266" text-anchor="middle">人类研究员以HITS身份参与</text>
<!-- 说明 -->
<text class="sts" x="340" y="308" text-anchor="middle" fill="#888">共同特征：需要多个专家协同、存在明确角色分工、任务有一定持续性</text>
<text class="sts" x="340" y="326" text-anchor="middle" fill="#888">JiuwenBox保障数据主权，Skill Hub沉淀和共享协作经验</text>
</svg>

---

## 价值潜力：为什么值得认真看

**1. 华为生态绑定，但不绑架**

深度集成华为云MaaS和小艺开放平台，对已经在华为云上的用户是直接加分。但框架本身是Apache-2.0协议，也支持Anspire、DeepSeek、Gemini、Ollama等多家模型，不是封闭生态。

**2. Coordination Engineering可能是下半场核心命题**

现在大多数Agent框架还在解决"一个Agent能干多少事"的问题，JiuwenSwarm在押注的是"一群Agent如何高效协作"——这个方向对标的是人类组织的协作模式，天花板更高。

**3. Swarm Skills Hub是潜在护城河**

如果技能共享生态起来了，就像npm之于Node.js——Hub上沉淀的团队级技能会形成网络效应，这才是最难复制的部分。目前Hub刚上线，能不能做起来还要看社区运营。

**4. 评测数据里的Token降低34.8%是货真价实的差异化**

在多Agent场景里，每个Agent都消耗上下文，Token成本是放大的。能把总Token降低34.8%，背后必然是有效的上下文压缩和卸载机制——源码里也确实能看到ContextCompression相关的模块。

---

## 一点冷静的判断

JiuwenSwarm工程化程度扎实，这点毋庸置疑。但有几个地方值得观察：

- **Coordination Engineering范式的落地深度**：两天Star 736，在AI框架里不算爆炸性数据。同类的OpenHands、AutoGen在发布时的数字都在这个量级之上。真正的考验是半年后社区活跃度。
- **Swarm Skills Hub的冷启动**：技能市场起步最难的就是鸡蛋问题——没有足够多的用户就没有足够好的技能，没有足够好的技能就吸引不来用户。
- **本地记忆方案的可扩展性**：基于SQLite的双路检索在个人和小团队场景很好，但如果是多节点分布式部署，SQLite的天然单机限制会成为瓶颈。

这些不是否定，而是看清楚它的适用边界：**现阶段JiuwenSwarm最适合的场景是：有数据主权需求、使用华为生态、需要多Agent协作的中等规模团队。**

---

## 总结

JiuwenSwarm做对了三件事：提出了一个有理论体系的范式（Coordination Engineering）、写出了一套工程上可落地的实现（Team Swarm + SkillEvolution）、找到了一个有护城河潜力的商业支点（Skill Hub）。

华为的开源项目里，工程质量高的不少，能做起社区的很少。JiuwenSwarm目前还在早期，但它押注的方向——让多个AI专家像一支真正的团队一样协作——这条路大概率是对的。

---

**项目地址**：[openJiuwen-ai/jiuwenswarm](https://github.com/openJiuwen-ai/jiuwenswarm)

**Swarm Skills Hub**：[swarmskills.openjiuwen.com](https://swarmskills.openjiuwen.com/)

*本文基于源码分析和公开资料，不构成任何投资或技术选型建议。*
