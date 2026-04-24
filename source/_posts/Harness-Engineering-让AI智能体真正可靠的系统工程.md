---
title: Harness Engineering：让AI智能体真正可靠的系统工程
date: 2026-04-24 14:00:00
tags:
  - AI
  - Agent
  - Harness Engineering
  - LLM
  - 软件工程
categories:
  - AI工程
---

> 2025年证明了Agent能工作，2026年要解决的是怎么让Agent稳定工作。答案不是更强的模型，而是更好的Harness。

## 一、背景：第三次范式跃迁

过去三年，AI工程经历了三次范式变迁，每一次都在回答一个更难的问题：

<svg viewBox="0 0 800 260" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<defs><marker id="ah1" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><polygon points="0 0,8 3,0 6" fill="#e94560"/></marker></defs>
<text x="400" y="30" text-anchor="middle" fill="#eee" font-size="16" font-weight="bold">AI工程的三次范式跃迁</text>
<!-- Prompt Engineering -->
<rect x="30" y="55" width="220" height="80" rx="8" fill="#16213e" stroke="#0f3460" stroke-width="1.5"/>
<text x="140" y="80" text-anchor="middle" fill="#e94560" font-size="14" font-weight="bold">Prompt Engineering</text>
<text x="140" y="100" text-anchor="middle" fill="#aaa" font-size="11">2022-2024</text>
<text x="140" y="122" text-anchor="middle" fill="#ccc" font-size="12">怎么和模型说话？</text>
<!-- Arrow 1 -->
<line x1="255" y1="95" x2="285" y2="95" stroke="#e94560" stroke-width="2" marker-end="url(#ah1)"/>
<!-- Context Engineering -->
<rect x="290" y="55" width="220" height="80" rx="8" fill="#16213e" stroke="#0f3460" stroke-width="1.5"/>
<text x="400" y="80" text-anchor="middle" fill="#f5a623" font-size="14" font-weight="bold">Context Engineering</text>
<text x="400" y="100" text-anchor="middle" fill="#aaa" font-size="11">2025</text>
<text x="400" y="122" text-anchor="middle" fill="#ccc" font-size="12">给模型看什么？</text>
<!-- Arrow 2 -->
<line x1="515" y1="95" x2="545" y2="95" stroke="#e94560" stroke-width="2" marker-end="url(#ah1)"/>
<!-- Harness Engineering -->
<rect x="550" y="55" width="220" height="80" rx="8" fill="#16213e" stroke="#e94560" stroke-width="2"/>
<text x="660" y="80" text-anchor="middle" fill="#4ecca3" font-size="14" font-weight="bold">Harness Engineering</text>
<text x="660" y="100" text-anchor="middle" fill="#aaa" font-size="11">2026-</text>
<text x="660" y="122" text-anchor="middle" fill="#ccc" font-size="12">怎么让Agent稳定工作？</text>
<!-- Bottom insight -->
<rect x="100" y="170" width="600" height="70" rx="8" fill="#0f3460" opacity="0.6"/>
<text x="400" y="198" text-anchor="middle" fill="#4ecca3" font-size="13" font-weight="bold">核心洞察</text>
<text x="400" y="222" text-anchor="middle" fill="#ddd" font-size="12">简单任务→提示词最重要 | 依赖知识的任务→上下文最关键 | 长链路真实场景→Harness决定成败</text>
</svg>

三者不是并列关系，而是嵌套关系：Prompt Engineering ⊂ Context Engineering ⊂ Harness Engineering。每次跃迁不是取代前一个，而是在更深的层面上解决更根本的问题。

这个概念的正式提出者是 Mitchell Hashimoto（HashiCorp 联合创始人），2026年2月在一次演讲中首次系统阐述了"驾驭工程"的理念。核心出处来自一个激进的实验：OpenAI的Codex团队用3名工程师、5个月时间、生成了100万行代码，零手写——靠的不是更强的模型，而是精心设计的运行环境。

## 二、核心概念：Agent = Model + Harness

一句话定义：**Harness是模型之外的一切**。

系统提示词、工具调用、文件系统、沙箱环境、编排逻辑、钩子中间件、反馈回路、约束机制——这些都是Harness。模型本身只是能力的来源，只有通过Harness把状态、工具、反馈和约束串起来，它才真正变成一个Agent。

<svg viewBox="0 0 800 300" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<text x="400" y="28" text-anchor="middle" fill="#eee" font-size="15" font-weight="bold">Agent = Model + Harness（模型是CPU，Harness是操作系统）</text>
<!-- Model box -->
<rect x="30" y="50" width="180" height="230" rx="10" fill="#16213e" stroke="#e94560" stroke-width="2"/>
<text x="120" y="80" text-anchor="middle" fill="#e94560" font-size="14" font-weight="bold">Model</text>
<text x="120" y="105" text-anchor="middle" fill="#aaa" font-size="11">能力来源</text>
<rect x="50" y="120" width="140" height="28" rx="4" fill="#1a1a2e"/>
<text x="120" y="139" text-anchor="middle" fill="#ccc" font-size="11">推理能力</text>
<rect x="50" y="155" width="140" height="28" rx="4" fill="#1a1a2e"/>
<text x="120" y="174" text-anchor="middle" fill="#ccc" font-size="11">语言理解</text>
<rect x="50" y="190" width="140" height="28" rx="4" fill="#1a1a2e"/>
<text x="120" y="209" text-anchor="middle" fill="#ccc" font-size="11">代码生成</text>
<rect x="50" y="225" width="140" height="28" rx="4" fill="#1a1a2e"/>
<text x="120" y="244" text-anchor="middle" fill="#ccc" font-size="11">工具调用</text>
<!-- Plus sign -->
<text x="240" y="175" text-anchor="middle" fill="#4ecca3" font-size="28" font-weight="bold">+</text>
<!-- Harness box -->
<rect x="270" y="50" width="500" height="230" rx="10" fill="#16213e" stroke="#4ecca3" stroke-width="2"/>
<text x="520" y="80" text-anchor="middle" fill="#4ecca3" font-size="14" font-weight="bold">Harness（驾驭层）</text>
<!-- Row 1 -->
<rect x="290" y="95" width="150" height="50" rx="6" fill="#0f3460"/>
<text x="365" y="116" text-anchor="middle" fill="#f5a623" font-size="12" font-weight="bold">上下文工程</text>
<text x="365" y="134" text-anchor="middle" fill="#aaa" font-size="10">新员工手册</text>
<rect x="450" y="95" width="150" height="50" rx="6" fill="#0f3460"/>
<text x="525" y="116" text-anchor="middle" fill="#4ecca3" font-size="12" font-weight="bold">架构约束</text>
<text x="525" y="134" text-anchor="middle" fill="#aaa" font-size="10">缰绳</text>
<rect x="610" y="95" width="145" height="50" rx="6" fill="#0f3460"/>
<text x="682" y="116" text-anchor="middle" fill="#e94560" font-size="12" font-weight="bold">熵管理</text>
<text x="682" y="134" text-anchor="middle" fill="#aaa" font-size="10">垃圾回收</text>
<!-- Row 2 -->
<rect x="290" y="155" width="150" height="50" rx="6" fill="#0f3460"/>
<text x="365" y="176" text-anchor="middle" fill="#8b5cf6" font-size="12" font-weight="bold">反馈循环</text>
<text x="365" y="194" text-anchor="middle" fill="#aaa" font-size="10">智能体审智能体</text>
<rect x="450" y="155" width="150" height="50" rx="6" fill="#0f3460"/>
<text x="525" y="176" text-anchor="middle" fill="#3b82f6" font-size="12" font-weight="bold">记忆系统</text>
<text x="525" y="194" text-anchor="middle" fill="#aaa" font-size="10">跨会话积累</text>
<rect x="610" y="155" width="145" height="50" rx="6" fill="#0f3460"/>
<text x="682" y="176" text-anchor="middle" fill="#ec4899" font-size="12" font-weight="bold">可观测性</text>
<text x="682" y="194" text-anchor="middle" fill="#aaa" font-size="10">行为溯源</text>
<!-- Row 3 -->
<rect x="290" y="215" width="305" height="50" rx="6" fill="#0f3460"/>
<text x="442" y="236" text-anchor="middle" fill="#f97316" font-size="12" font-weight="bold">执行编排</text>
<text x="442" y="254" text-anchor="middle" fill="#aaa" font-size="10">Plan → Battle → Execute</text>
<rect x="610" y="215" width="145" height="50" rx="6" fill="#0f3460"/>
<text x="682" y="236" text-anchor="middle" fill="#14b8a6" font-size="12" font-weight="bold">故障恢复</text>
<text x="682" y="254" text-anchor="middle" fill="#aaa" font-size="10">重试/回滚/降级</text>
</svg>

为什么瓶颈不在模型而在Harness？一个实验可以说明：Can.ac团队用同一个模型，只换了文件编辑接口的调用方式，编码基准分数从6.7%直接跳到68.3%。模型没变，变的是外围的系统。LangChain的Terminal Bench 2.0也验证了这一点——优化Agent运行环境后，从全球第30名升到第5名，得分从52.8%涨到66.5%。模型没换，Harness换了。

核心哲学八个字：**人类掌舵，智能体执行。** Harness不优化模型本身，而是优化模型运行的环境。Agent的每一次失败，都是环境设计不完善的信号。

## 三、六层架构：从信息到约束

Harness的工程化落地，需要理解其分层架构：

<svg viewBox="0 0 800 520" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<text x="400" y="28" text-anchor="middle" fill="#eee" font-size="15" font-weight="bold">Harness 六层架构</text>
<!-- L6 -->
<rect x="50" y="45" width="700" height="65" rx="8" fill="#2d1b3d" stroke="#e94560" stroke-width="2"/>
<text x="80" y="72" fill="#e94560" font-size="14" font-weight="bold">L6 约束·校验·恢复</text>
<text x="80" y="95" fill="#aaa" font-size="11">红线规则 · 应急预案 · 出错重试/回滚 · 什么事绝对不能做，出了事怎么补救</text>
<text x="720" y="82" text-anchor="end" fill="#e94560" font-size="20">🛡️</text>
<!-- L5 -->
<rect x="50" y="118" width="700" height="65" rx="8" fill="#1b2d3d" stroke="#f5a623" stroke-width="1.5"/>
<text x="80" y="145" fill="#f5a623" font-size="14" font-weight="bold">L5 评估·观测</text>
<text x="80" y="168" fill="#aaa" font-size="11">独立验证机制 · 可观测性 · 质检流程 · 怎么检验做对了没有</text>
<text x="720" y="152" text-anchor="end" fill="#f5a623" font-size="20">🔍</text>
<!-- L4 -->
<rect x="50" y="191" width="700" height="65" rx="8" fill="#1b2d2d" stroke="#4ecca3" stroke-width="1.5"/>
<text x="80" y="218" fill="#4ecca3" font-size="14" font-weight="bold">L4 记忆·状态</text>
<text x="80" y="241" fill="#aaa" font-size="11">任务状态管理 · 中间产物 · 长期记忆 · 项目管理系统和笔记本</text>
<text x="720" y="225" text-anchor="end" fill="#4ecca3" font-size="20">🧠</text>
<!-- L3 -->
<rect x="50" y="264" width="700" height="65" rx="8" fill="#1b1b3d" stroke="#8b5cf6" stroke-width="1.5"/>
<text x="80" y="291" fill="#8b5cf6" font-size="14" font-weight="bold">L3 执行编排</text>
<text x="80" y="314" fill="#aaa" font-size="11">多步骤任务串联 · Plan-Execute · 标准操作流程</text>
<text x="720" y="298" text-anchor="end" fill="#8b5cf6" font-size="20">⚙️</text>
<!-- L2 -->
<rect x="50" y="337" width="700" height="65" rx="8" fill="#1b2d1b" stroke="#3b82f6" stroke-width="1.5"/>
<text x="80" y="364" fill="#3b82f6" font-size="14" font-weight="bold">L2 工具系统</text>
<text x="80" y="387" fill="#aaa" font-size="11">工具选拔 · 调用时机 · 结果提炼 · 办公工具</text>
<text x="720" y="371" text-anchor="end" fill="#3b82f6" font-size="20">🔧</text>
<!-- L1 -->
<rect x="50" y="410" width="700" height="65" rx="8" fill="#2d2d1b" stroke="#f97316" stroke-width="1.5"/>
<text x="80" y="437" fill="#f97316" font-size="14" font-weight="bold">L1 信息边界</text>
<text x="80" y="460" fill="#aaa" font-size="11">角色定义 · 目标裁剪 · 任务状态组织 · 岗位说明书（该关注什么）</text>
<text x="720" y="444" text-anchor="end" fill="#f97316" font-size="20">📋</text>
<!-- Priority hint -->
<rect x="200" y="490" width="400" height="25" rx="4" fill="#0f3460" opacity="0.7"/>
<text x="400" y="507" text-anchor="middle" fill="#4ecca3" font-size="11">💡 实操建议：先搭 L1（信息边界）+ L6（约束恢复），投入产出比最高</text>
</svg>

类比一个新手员工的工作环境就很好理解：

- L1 = 岗位说明书（该关注什么）
- L2 = 办公工具（用什么干活）
- L3 = 标准操作流程（按什么步骤做事）
- L4 = 项目管理系统和笔记本（怎么记住做过的事）
- L5 = 质检流程（怎么检验做对了没有）
- L6 = 红线规则和应急预案（什么事绝对不能做、出了事怎么补救）

实操建议：不要一开始就搭齐六层。**从L1和L6入手，投入产出比最高**。L1决定Agent知道该干什么，L6决定搞砸了能不能拉回来。中间层次随项目复杂度逐步补齐。

## 四、上下文管理：40%红线与上下文重置

Dex Horthy在观察了大量Agent运行后，发现了一个关键阈值：168K token的上下文窗口，用到约40%时，Agent输出质量明显下降。

<svg viewBox="0 0 800 200" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<text x="400" y="25" text-anchor="middle" fill="#eee" font-size="14" font-weight="bold">上下文利用率 vs Agent 输出质量</text>
<!-- Smart Zone -->
<rect x="50" y="45" width="280" height="110" rx="6" fill="#0a3d2a" stroke="#4ecca3" stroke-width="1.5"/>
<text x="190" y="70" text-anchor="middle" fill="#4ecca3" font-size="16" font-weight="bold">Smart Zone (0-40%)</text>
<text x="190" y="95" text-anchor="middle" fill="#aaa" font-size="11">推理聚焦 · 工具调用准确 · 代码质量高</text>
<text x="190" y="115" text-anchor="middle" fill="#4ecca3" font-size="12">✅ 这是Agent的最佳工作区间</text>
<!-- Dumb Zone -->
<rect x="340" y="45" width="410" height="110" rx="6" fill="#3d0a0a" stroke="#e94560" stroke-width="1.5"/>
<text x="545" y="70" text-anchor="middle" fill="#e94560" font-size="16" font-weight="bold">Dumb Zone (>40%)</text>
<text x="545" y="95" text-anchor="middle" fill="#aaa" font-size="11">幻觉增多 · 兜圈子 · 格式混乱 · 低质量代码</text>
<text x="545" y="115" text-anchor="middle" fill="#e94560" font-size="12">❌ 上下文膨胀导致性能断崖</text>
<!-- 40% threshold line -->
<line x1="330" y1="40" x2="330" y2="160" stroke="#f5a623" stroke-width="2" stroke-dasharray="6,3"/>
<text x="330" y="175" text-anchor="middle" fill="#f5a623" font-size="12" font-weight="bold">40% 阈值</text>
</svg>

Anthropic还发现一个有趣的现象：Sonnet 4.5在上下文快填满时会变得犹豫，倾向于提前收工——哪怕任务还没做完。这叫"上下文焦虑"。

怎么解决？Anthropic最终的做法不是压缩，而是**重置**：

1. 上下文接近饱和时，先把当前状态、已完成工作、待办事项结构化提取
2. 启动一个全新的"干净"Agent
3. 把结构化交接文档交给新Agent，从干净状态继续工作

类比程序碰到内存泄漏的解法——不是手动释放每个内存块，而是直接重启进程，从检查点恢复状态。

## 五、实操：Plan-Battle-Execute 执行策略

复杂任务的执行策略，一线团队普遍推荐Plan→Battle→Execute模式，而非让Agent自由探索（ReAct）：

<svg viewBox="0 0 800 360" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<defs><marker id="ah2" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><polygon points="0 0,8 3,0 6" fill="#4ecca3"/></marker></defs>
<text x="400" y="28" text-anchor="middle" fill="#eee" font-size="15" font-weight="bold">Plan → Battle → Execute 五步流程</text>
<!-- Step 1 -->
<rect x="50" y="50" width="140" height="55" rx="8" fill="#16213e" stroke="#f97316" stroke-width="1.5"/>
<text x="120" y="73" text-anchor="middle" fill="#f97316" font-size="12" font-weight="bold">① 动态规划</text>
<text x="120" y="93" text-anchor="middle" fill="#aaa" font-size="10">制定计划·分解步骤</text>
<!-- Arrow -->
<line x1="195" y1="77" x2="220" y2="77" stroke="#4ecca3" stroke-width="1.5" marker-end="url(#ah2)"/>
<!-- Step 2 -->
<rect x="225" y="50" width="140" height="55" rx="8" fill="#16213e" stroke="#e94560" stroke-width="1.5"/>
<text x="295" y="73" text-anchor="middle" fill="#e94560" font-size="12" font-weight="bold">② 辩论验证</text>
<text x="295" y="93" text-anchor="middle" fill="#aaa" font-size="10">攻击计划·找漏洞</text>
<!-- Arrow -->
<line x1="370" y1="77" x2="395" y2="77" stroke="#4ecca3" stroke-width="1.5" marker-end="url(#ah2)"/>
<!-- Step 3 -->
<rect x="400" y="50" width="140" height="55" rx="8" fill="#16213e" stroke="#8b5cf6" stroke-width="1.5"/>
<text x="470" y="73" text-anchor="middle" fill="#8b5cf6" font-size="12" font-weight="bold">③ 迭代打磨</text>
<text x="470" y="93" text-anchor="middle" fill="#aaa" font-size="10">N轮辩论·质量达标</text>
<!-- Arrow -->
<line x1="545" y1="77" x2="570" y2="77" stroke="#4ecca3" stroke-width="1.5" marker-end="url(#ah2)"/>
<!-- Step 4 -->
<rect x="575" y="50" width="100" height="55" rx="8" fill="#16213e" stroke="#f5a623" stroke-width="1.5"/>
<text x="625" y="73" text-anchor="middle" fill="#f5a623" font-size="12" font-weight="bold">④ 人确认</text>
<text x="625" y="93" text-anchor="middle" fill="#aaa" font-size="10">关键控制节点</text>
<!-- Arrow -->
<line x1="680" y1="77" x2="705" y2="77" stroke="#4ecca3" stroke-width="1.5" marker-end="url(#ah2)"/>
<!-- Step 5 -->
<rect x="710" y="50" width="70" height="55" rx="8" fill="#0a3d2a" stroke="#4ecca3" stroke-width="2"/>
<text x="745" y="73" text-anchor="middle" fill="#4ecca3" font-size="12" font-weight="bold">⑤ 执行</text>
<text x="745" y="93" text-anchor="middle" fill="#aaa" font-size="10">确定性执行</text>
<!-- Comparison -->
<rect x="50" y="140" width="340" height="90" rx="8" fill="#16213e" stroke="#e94560" stroke-width="1"/>
<text x="220" y="165" text-anchor="middle" fill="#e94560" font-size="13" font-weight="bold">❌ ReAct（自由探索）</text>
<text x="220" y="188" text-anchor="middle" fill="#aaa" font-size="11">灵活但不可控，token成本高</text>
<text x="220" y="210" text-anchor="middle" fill="#aaa" font-size="11">路径不可复现，结果难保证</text>
<rect x="410" y="140" width="340" height="90" rx="8" fill="#0a3d2a" stroke="#4ecca3" stroke-width="2"/>
<text x="580" y="165" text-anchor="middle" fill="#4ecca3" font-size="13" font-weight="bold">✅ Plan-Battle-Execute</text>
<text x="580" y="188" text-anchor="middle" fill="#aaa" font-size="11">规划质量高，执行可控，风险可预见</text>
<text x="580" y="210" text-anchor="middle" fill="#aaa" font-size="11">从"过程确定性"到"结果确定性"</text>
<!-- Key insight -->
<rect x="100" y="260" width="600" height="55" rx="8" fill="#0f3460" opacity="0.6"/>
<text x="400" y="285" text-anchor="middle" fill="#4ecca3" font-size="12" font-weight="bold">关键转变</text>
<text x="400" y="303" text-anchor="middle" fill="#ddd" font-size="11">人的角色从"写每一行代码"变成"画终点线 + 验收结果"——告诉AI终点在哪，让它自己找路</text>
</svg>

LangChain的推理预算实验也很有启发：纯高推理模式得分53.9%（大量超时），中等推理63.6%，而"高-中-高"三明治策略（规划高推理→实现中等→验证高推理）得分66.5%。给Agent更多思考时间不等于更好，关键是把推理预算花在刀刃上。

## 六、一线团队实战：数据与踩坑

### OpenAI（Codex团队）

3名工程师5个月生成100万行代码、约1500个PR、零手写代码。核心做法：

- **AGENTS.md采用渐进式披露**：约100行的目录文件，指向88个子系统深层文档。踩过的坑：巨型单文件导致上下文拥挤、过度指导、即时腐烂。
- **架构约束编码为Linter**：分层依赖方向`Types → Config → Repo → Service → Runtime → UI`，违反即CI阻止合并。Linter报错不只说违反了什么，还告诉Agent怎么改。
- **熵治理**：曾每周花20%时间手动清理"AI slop"，后来改为后台Agent定期扫描偏差代码、开出重构PR。

### Stripe（Minions系统）

每周1300+无人值守PR合并。核心做法：

- **500个工具不全暴露**：确定性编排器筛选约15个相关工具给Agent。踩过的坑：工具太多导致"token瘫痪"——Agent浪费时间选择工具而非做事。
- **Blueprint模式**：确定性门禁"夹住"概率性LLM工作，Agent最多尝试2轮CI，然后要么成功要么放弃。

### Anthropic

- **双体架构**：初始化Agent生成200+功能列表（JSON格式），编码Agent每次只做一个功能。用JSON而非Markdown追踪状态，因为Markdown更容易被Agent意外修改。
- **浏览器自动化验证**：显式提示Agent使用Puppeteer MCP做端到端测试。踩过的坑：Agent倾向用curl或单元测试声称完成，不做端到端验证。

### 六大高频踩坑

| 踩坑 | 表现 | 应对 |
|------|------|------|
| First Plausible Solution Bias | 对第一个看起来合理的方案产生偏见，不验证就停 | Build-Verify Loop，对照原始需求验证 |
| AI Slop放大 | 忠实复制代码库中的坏模式 | 自定义Linter + 后台清理Agent |
| Doom Loop | 同一个坏方案反复微调10+次 | LoopDetectionMiddleware，超N次注入"重新审视方案" |
| 伪完成 | 用curl/单元测试声称功能完成 | 强制端到端测试 |
| Token瘫痪 | 工具太多，浪费时间选工具 | 确定性编排器筛选相关工具 |
| 巨型文档失效 | 单一AGENTS.md文件撑爆上下文 | 渐进式披露，目录+子文档 |

## 七、知识工程：真正的护城河

<svg viewBox="0 0 800 280" xmlns="http://www.w3.org/2000/svg" style="width:100%;background:#1a1a2e;border-radius:8px;">
<defs><marker id="ah3" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><polygon points="0 0,8 3,0 6" fill="#e94560"/></marker></defs>
<text x="400" y="28" text-anchor="middle" fill="#eee" font-size="15" font-weight="bold">知识漏斗：90%的信息在传递中损耗</text>
<!-- Funnel layers -->
<polygon points="150,55 650,55 580,100 220,100" fill="#4ecca3" opacity="0.8"/>
<text x="400" y="83" text-anchor="middle" fill="#1a1a2e" font-size="13" font-weight="bold">脑子里想的（100%）</text>
<polygon points="220,105 580,105 510,150 290,150" fill="#f5a623" opacity="0.8"/>
<text x="400" y="133" text-anchor="middle" fill="#1a1a2e" font-size="13" font-weight="bold">说出来的（~40%）</text>
<polygon points="290,155 510,155 440,200 360,200" fill="#3b82f6" opacity="0.8"/>
<text x="400" y="183" text-anchor="middle" fill="#1a1a2e" font-size="13" font-weight="bold">写下来的（~15%）</text>
<polygon points="360,205 440,205 420,250 380,250" fill="#e94560" opacity="0.8"/>
<text x="400" y="233" text-anchor="middle" fill="#fff" font-size="12" font-weight="bold">AI可用的（~5%）</text>
<!-- Arrow -->
<line x1="460" y1="230" x2="520" y2="230" stroke="#e94560" stroke-width="2" marker-end="url(#ah3)"/>
<text x="660" y="225" text-anchor="middle" fill="#e94560" font-size="12" font-weight="bold">损耗95%</text>
<text x="660" y="245" text-anchor="middle" fill="#aaa" font-size="10">隐性知识 → AI永远学不会</text>
</svg>

模型智力达到临界点之后，谁在垂直领域的知识沉淀更深，谁就能让AI产出更好的结果。这种隐性知识——比如代码里处理了A/B/C三种账号类型但实际有五种——通用模型永远学不会，因为它是公司内部的私域定义，从没出现在任何公开资料里。

务实的知识存储策略：Markdown + JSON。纯文本摩擦最低、组织最灵活——不是最优解，而是"没有办法的办法"中最好的那个。核心原则是**Local First**：不在快速变化的线上产品中沉淀核心上下文，本地存储的纯文本文件才是最稳定、最可控、最容易迁移的知识载体。

## 八、最佳实践行动清单

从零搭建Harness，按优先级排列：

### P0：立即做

| 行动 | 原因 |
|------|------|
| 创建AGENTS.md并持续维护 | Agent每次启动自动加载，犯错就更新，形成反馈循环 |
| 构建自定义Linter+修复指令 | 错误消息里直接告诉Agent怎么改，纠错的同时在"教" |
| 把团队知识放进仓库 | 写在Slack/Wiki里的知识对Agent等于不存在 |

⚠️ 常见误区：把AGENTS.md当"超级System Prompt"写，所有规则塞一个文件，撑爆上下文。正确做法：AGENTS.md只当目录用（约100行），详细规则放子文档按需加载。

### P1：P0做完后考虑

| 行动 | 原因 |
|------|------|
| 分层管理上下文 | 渐进式披露，不塞一个文件 |
| 建立进度文件和功能列表 | JSON格式追踪状态，Agent不太会乱改结构化数据 |
| 给Agent端到端验证能力 | 浏览器自动化让Agent能像用户一样验证 |
| 控制上下文利用率 | 尽量不超过40%，增量执行 |

### P2：有余力再考虑

| 行动 | 原因 |
|------|------|
| Agent专业化分工 | 每个Agent携带更少无关信息，留在Smart Zone |
| 定期垃圾回收 | 确保清理速度跟得上生成速度 |
| 可观测性集成 | 把"性能优化"从玄学变成可度量 |

## 九、未来趋势：从Harness到Environment

Harness Engineering本身也在演化。下一个概念已经在浮现——**Environment Engineering（环境工程）**。

如果Harness是给一匹马装上缰绳和马鞍，Environment Engineering就是修路、建交通信号灯、设立路牌。不再只是约束单个Agent，而是让整个运行环境本身就具备确定性——类似车路协同（V2X），不只是让车更聪明，而是让路也更聪明。

目前的几个趋势方向：

1. **多Agent编排标准化**：从单Agent的Harness到多Agent间的协议和协作机制
2. **可观测性一等公民**：从"事后调试"到"设计阶段就嵌入可观测性"
3. **知识工程自动化**：从手动写AGENTS.md到Agent自动沉淀和更新知识
4. **Harness自身由AI管理**：用AI监控和优化Harness参数，形成元循环

## 十、总结

一句话：**模型决定了系统的上限，Harness决定了系统的底线。** 与其纠结选哪个模型，不如先把Harness搭好。

三次范式跃迁的核心逻辑：从"怎么表达"到"给什么信息"再到"怎么让系统可靠运行"。每一次跃迁都在更深的层面上解决问题，而Harness Engineering是当前最深的那一层。

对于工程师而言，角色正在发生根本性转变：从代码作者变成系统设计师——设计约束、构建反馈回路、维护知识基础设施、管理工作流编排。Architect和Conductor正在成为AI时代最关键的工程师角色。

核心不是你用什么模型，而是你为模型搭建了什么样的运行环境。

---

**参考资料**

- Mitchell Hashimoto, "Agents in the Terminal", 2026.02
- Ryan Lopopolo (OpenAI Codex), "Building Agents That Work", 2026.03
- Justin Young (Anthropic), "Context Resets and Agent Architecture", 2026.03
- Vivek Trivedi (LangChain), "The Anatomy of an Agent Harness", 2026.03
- Dex Horthy, "The 40% Context Threshold", 2026.02
- Can.ac, "Edit Format Impact on Agent Performance", 2026.01
- Birgitta Böckeler, "Ambient Affordances for AI Agents", 2026.03
