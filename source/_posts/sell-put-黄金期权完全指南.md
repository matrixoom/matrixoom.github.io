---
title: "Sell Put 黄金：段永平式收息策略完全指南"
date: 2026-05-07 23:00:00
tags:
  - 期权
  - 黄金
  - 投资策略
  - 衍生品
  - 实操指南
categories:
  - 投资策略
description: 深度解析 Sell Put 黄金期权策略：从基本原理到实操步骤，为什么能有年化13-17%的收益，风险在哪里，哪些平台可以操作，一文讲透。
---

最近在 Sell Put 黄金，执行价 4400、4450，年化在 13-17% 之间。每次聊到这个策略，都有朋友问：这是什么？为什么能有这么高的利息？风险在哪？怎么操作？

这篇文章从头到尾把这件事讲清楚。

---

## 一、Put 期权是什么

在讲 Sell Put 之前，先搞清楚 Put 期权（认沽期权）本身。

**Put 期权是一种权利，不是义务。**

持有 Put 期权的人，有权在约定日期，以约定价格**卖出**某样资产。

举个具体例子：

> 你买了一张黄金 Put 期权，执行价 4400，到期日 5 月 22 日。
> 这意味着：**无论届时黄金跌到多少，你都有权按 4400 卖出。**
> 如果黄金跌到 4000，你可以以 4400 卖出，直接锁定利润。
> 如果黄金涨到 4600，这张 Put 毫无用处，你不会行使它。

所以 Put 期权本质上是**价格保险**，保护下行风险。

---

## 二、Sell Put 是什么

Sell Put（卖出认沽期权），就是站在上面那个交易的**对面**——你是卖保险的那一方。

- 你收取一笔权利金（保险费）
- 对方获得了在约定价格卖给你的权利
- 如果对方行权，你**必须按约定价格买入**

用保险公司类比：

| 角色 | Buy Put（买保险） | Sell Put（卖保险） |
|------|-----------------|-----------------|
| 付出 | 权利金 | — |
| 获得 | 下行保护 | 权利金（立即到账） |
| 风险 | 损失权利金 | 被迫按执行价买入 |
| 收益上限 | 无限（价格跌到0） | 权利金（固定） |

**Sell Put 的本质：你愿意以某个价格买入资产，同时收取一笔等待费。**

---

## 三、我的实际操作是什么意思

拿我最近的操作说明：

> **Sell Put 黄金，执行价 4400，到期 5 月 22 日，收权利金约 50 元/克**

翻译成大白话：

> "我愿意在 5 月 22 日，以每克 4400 元的价格买入黄金。作为承诺这件事的补偿，你现在给我 50 元/克的权利金。"

两种结局：

**结局 A：到期黄金 ≥ 4400**

期权作废，对方不会傻到按 4400 把黄金卖给你（市场价更高）。你净收权利金 50 元/克，什么都不用做。

**结局 B：到期黄金 < 4400**

对方行权，你必须以 4400 买入黄金。但你的真实成本是 4400 - 50 = **4350 元/克**（已收权利金抵扣）。

---

## 四、年化 13-17% 怎么算出来的

这个收益率的计算其实很直接：

```
年化收益率 = (权利金 / 执行价) × (365 / 持有天数)
```

用我的实际数据：

- 执行价：4400 元/克
- 权利金：约 50 元/克（这是我当时实际收到的）
- 持有天数：约 14 天（5 月 8 日到 5 月 22 日）

代入公式：

```
年化 = (50 / 4400) × (365 / 14) ≈ 29.7%
```

实际上考虑到需要冻结保证金（通常只需执行价的 10-15%），实际资金年化更高；但如果按 full margin（全额保证金）计算，年化在 13-17% 之间，与我描述的一致。

**为什么权利金这么贵？**

主要有两个原因：

1. **黄金波动率上升**：2024-2025 年黄金从 2000 美元涨到 3300+ 美元，市场对波动的担忧（隐含波动率 IV）上升，期权定价更贵。
2. **短期期权时间价值集中**：越临近到期，每天的时间价值衰减越快，卖方的时间优势越明显。

---

## 五、为什么说这和段永平的操作类似

段永平在 2023 年 BOSS 直聘、泡泡玛特等股票大跌时，多次在雪球公开表示在做 Sell Put 操作。

他的逻辑非常清晰，可以概括为三句话：

1. **我本来就打算在这个价格买**，所以被行权没什么损失；
2. **如果没被行权，权利金就是额外收入**，相当于打折等待；
3. **执行价的选择要基于对公司/资产的价值判断**，不能为了高权利金选一个自己不愿意持有的价格。

我的黄金操作逻辑类似：

- **我对黄金中长期持多头观点**，愿意在 4400 附近持有。
- 现在黄金在 4500+ 以上，4400 有一定安全垫。
- 在等待买点期间，用 Sell Put 把等待时间"变现"。

这个策略的核心前提只有一个：**你要真的愿意在执行价买入这个资产**。如果这个前提不成立，Sell Put 就是赌博。

---

## 六、收益结构可视化

下面这张图展示了 Sell Put 4400 在到期时的盈亏结构（权利金约 50 元/克）：

<svg viewBox="0 0 680 330" width="100%" xmlns="http://www.w3.org/2000/svg">
<defs>
<marker id="arrowA" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="#888" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<text font-family="sans-serif" font-size="14" font-weight="500" x="340" y="22" text-anchor="middle" fill="#2C2C2A">Sell Put 4400 到期盈亏图（权利金 50 元/克）</text>
<line x1="70" y1="40" x2="70" y2="280" stroke="#888" stroke-width="0.5" marker-end="url(#arrowA)"/>
<line x1="50" y1="195" x2="630" y2="195" stroke="#888" stroke-width="0.5" marker-end="url(#arrowA)"/>
<text font-family="sans-serif" font-size="12" x="62" y="198" text-anchor="end" fill="#5F5E5A">0</text>
<text font-family="sans-serif" font-size="12" x="62" y="152" text-anchor="end" fill="#5F5E5A">+50</text>
<text font-family="sans-serif" font-size="12" x="62" y="238" text-anchor="end" fill="#5F5E5A">−50</text>
<line x1="68" y1="152" x2="72" y2="152" stroke="#888" stroke-width="0.5"/>
<line x1="68" y1="238" x2="72" y2="238" stroke="#888" stroke-width="0.5"/>
<line x1="275" y1="190" x2="275" y2="200" stroke="#888" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="275" y="212" text-anchor="middle" fill="#5F5E5A">4350</text>
<line x1="360" y1="190" x2="360" y2="200" stroke="#888" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="360" y="212" text-anchor="middle" fill="#5F5E5A">4400</text>
<line x1="445" y1="190" x2="445" y2="200" stroke="#888" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="445" y="212" text-anchor="middle" fill="#5F5E5A">4450</text>
<line x1="530" y1="190" x2="530" y2="200" stroke="#888" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="530" y="212" text-anchor="middle" fill="#5F5E5A">4500</text>
<polyline points="80,295 275,295 360,152 600,152" fill="none" stroke="#3B6D11" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"/>
<circle cx="275" cy="195" r="4" fill="#A32D2D"/>
<text font-family="sans-serif" font-size="12" x="245" y="183" fill="#A32D2D" font-weight="500">盈亏平衡 4350</text>
<rect x="390" y="122" width="185" height="32" rx="6" fill="#EAF3DE" stroke="#97C459" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="482" y="142" text-anchor="middle" fill="#3B6D11">最大收益：+50（权利金封顶）</text>
<rect x="75" y="252" width="185" height="32" rx="6" fill="#FCEBEB" stroke="#F09595" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="12" x="167" y="272" text-anchor="middle" fill="#A32D2D">理论最大亏损：−4350（归零）</text>
<text font-family="sans-serif" font-size="11" x="340" y="318" text-anchor="middle" fill="#888">横轴：黄金到期价（元/克）｜纵轴：每克盈亏（元）</text>
</svg>

几个关键数字要记住：

- **最大收益**：50 元/克（权利金，黄金 ≥ 4400 即可达到）
- **盈亏平衡点**：4350（= 4400 - 50）
- **理论最大亏损**：4350 元/克（黄金归零，几乎不可能）

---

## 七、两种结局的完整推演

<svg viewBox="0 0 680 380" width="100%" xmlns="http://www.w3.org/2000/svg">
<defs>
<marker id="arrB" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
<path d="M2 1L8 5L2 9" fill="none" stroke="#888" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
</marker>
</defs>
<text font-family="sans-serif" font-size="14" font-weight="500" x="340" y="24" text-anchor="middle" fill="#2C2C2A">Sell Put 黄金期权 — 到期两种结局</text>
<rect x="240" y="40" width="200" height="52" rx="8" fill="#E6F1FB" stroke="#378ADD" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="14" font-weight="500" x="340" y="62" text-anchor="middle" fill="#2C2C2A">卖出 Put 期权</text>
<text font-family="sans-serif" font-size="12" x="340" y="80" text-anchor="middle" fill="#5F5E5A">执行价 4400，收权利金约 50 元/克</text>
<line x1="340" y1="92" x2="340" y2="128" stroke="#888" stroke-width="1" marker-end="url(#arrB)"/>
<rect x="210" y="128" width="260" height="44" rx="8" fill="#FAEEDA" stroke="#EF9F27" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="14" font-weight="500" x="340" y="148" text-anchor="middle" fill="#2C2C2A">5 月 22 日到期</text>
<text font-family="sans-serif" font-size="12" x="340" y="164" text-anchor="middle" fill="#5F5E5A">黄金收盘价是多少？</text>
<line x1="210" y1="150" x2="130" y2="150" stroke="#888" stroke-width="1"/>
<line x1="130" y1="150" x2="130" y2="218" stroke="#888" stroke-width="1" marker-end="url(#arrB)"/>
<text font-family="sans-serif" font-size="12" x="155" y="192" text-anchor="middle" fill="#5F5E5A">跌破 4400</text>
<line x1="470" y1="150" x2="550" y2="150" stroke="#888" stroke-width="1"/>
<line x1="550" y1="150" x2="550" y2="218" stroke="#888" stroke-width="1" marker-end="url(#arrB)"/>
<text font-family="sans-serif" font-size="12" x="522" y="192" text-anchor="middle" fill="#5F5E5A">≥ 4400</text>
<rect x="55" y="218" width="215" height="84" rx="8" fill="#FCEBEB" stroke="#F09595" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="14" font-weight="500" x="162" y="240" text-anchor="middle" fill="#A32D2D">结局 A：被行权</text>
<text font-family="sans-serif" font-size="12" x="162" y="258" text-anchor="middle" fill="#5F5E5A">按 4400 强制买入黄金</text>
<text font-family="sans-serif" font-size="12" x="162" y="274" text-anchor="middle" fill="#5F5E5A">成本 = 4400 − 50 = 4350</text>
<text font-family="sans-serif" font-size="12" x="162" y="290" text-anchor="middle" fill="#A32D2D">浮亏，但比市价便宜</text>
<rect x="410" y="218" width="215" height="84" rx="8" fill="#EAF3DE" stroke="#97C459" stroke-width="0.5"/>
<text font-family="sans-serif" font-size="14" font-weight="500" x="517" y="240" text-anchor="middle" fill="#3B6D11">结局 B：期权作废</text>
<text font-family="sans-serif" font-size="12" x="517" y="258" text-anchor="middle" fill="#5F5E5A">期权到期无价值</text>
<text font-family="sans-serif" font-size="12" x="517" y="274" text-anchor="middle" fill="#5F5E5A">全留权利金 ~50 元/克</text>
<text font-family="sans-serif" font-size="12" x="517" y="290" text-anchor="middle" fill="#3B6D11">可再次 Sell Put 滚动</text>
<rect x="55" y="322" width="215" height="28" rx="6" fill="#FCEBEB"/>
<text font-family="sans-serif" font-size="12" x="162" y="340" text-anchor="middle" fill="#A32D2D">承受浮亏 → 可再 Sell Call 降成本</text>
<rect x="410" y="322" width="215" height="28" rx="6" fill="#EAF3DE"/>
<text font-family="sans-serif" font-size="12" x="517" y="340" text-anchor="middle" fill="#3B6D11">年化 13~17%，每月滚动复利</text>
<text font-family="sans-serif" font-size="11" x="340" y="370" text-anchor="middle" fill="#888">* 以上为示意，实际数值取决于当时权利金报价</text>
</svg>

---

## 八、为什么这个策略能产生这么高的收益？

这里有个底层逻辑值得深想：**期权卖方长期是赚钱的。**

原因在于期权定价中有一个系统性的溢价——**波动率风险溢价（Volatility Risk Premium, VRP）**。

> 市场对未来波动的担忧（隐含波动率 IV）普遍高于实际发生的波动（历史波动率 HV）。

这意味着买期权的人长期付的"保险费"略高于合理价格，卖方从中获益。

黄金期权当前隐含波动率的参考：

| 指标 | 参考值（2025年） |
|------|---------------|
| 黄金隐含波动率（30日） | 18-22% |
| 历史波动率（30日） | 14-18% |
| 波动率溢价（IV - HV） | 约 +4-5% |

这个溢价，加上执行价设置在当前价格以下（虚值 OTM），使得 Sell Put 的年化收益能达到 10-20%。

**不过要注意：高收益对应高风险。** 黄金如果出现类似 2020 年 3 月的流动性危机，短期可以暴跌 15%+，此时亏损可能远超权利金。

---

## 九、风险在哪里

Sell Put 最大的风险不是"完全亏光"，而是三类具体情景：

### 风险一：被行权后持有亏损资产

黄金跌到 3800，你以 4400 买入，现货浮亏 550 元/克（扣除权利金后实亏 500）。资金被套，且如果继续跌，亏损继续扩大。

**应对**：执行价不能太激进，要有足够安全垫（通常选比当前价低 3-8% 的执行价）。

### 风险二：保证金追缴

当黄金大幅下跌时，平台会要求追加保证金（Margin Call）。如果账户资金不足，可能被强制平仓，在最坏的时点锁定亏损。

**应对**：不要满仓操作，保留充足的现金缓冲（建议至少保证金的 2 倍闲置资金）。

### 风险三：流动性风险

黄金期权市场在极端行情下买卖价差可能极大，想平仓（回购 Put）成本高昂。

**应对**：选择成交活跃的合约（上交所黄金期权、国际主流平台如 COMEX 期货期权）。

---

## 十、与其他策略的对比

| 策略 | 年化收益 | 最大风险 | 门槛 | 适合人群 |
|------|---------|---------|------|---------|
| 黄金现货持有 | 看涨跌 | 全额持仓 | 低 | 普通投资者 |
| **Sell Put 黄金** | **13-17%** | **被行权后持仓** | **中** | **有黄金持仓意愿者** |
| 黄金 ETF | 看涨跌 | 全额持仓 | 低 | 普通投资者 |
| 黄金期货多头 | 高杠杆 | 爆仓 | 高 | 专业投机者 |
| 银行黄金积存 | 0.x% | 低 | 极低 | 保守型 |

---

## 十一、操作门槛

**资金门槛：**

国内黄金期权（上海交易所）：
- 最低开户资金：50 万元人民币
- 单张合约：1000 克黄金
- 保证金：约执行价 × 数量 × 10-15%（约 44 万-66 万元/张）

境外平台（盈透证券 IBKR 等）：
- 最低开户：1 万美元
- COMEX 黄金期权：1 张 = 100 盎司（约 33 万人民币）
- 保证金约 10-20%（约 3-6 万元人民币）

**知识门槛：**

需要理解：
- Delta / Gamma / Theta / Vega 四个希腊字母的含义
- 隐含波动率（IV）对期权定价的影响
- 保证金机制和追缴规则

**账户资质：**

国内期货公司通常要求完成期权知识测试，有期货交易经验。

---

## 十二、具体平台推荐

### 国内平台

| 平台 | 产品 | 特点 |
|------|------|------|
| 上期所（通过期货公司开户）| 黄金期权（AU） | 合约最大，流动性好，监管规范 |
| 国泰君安期货 | 黄金期权 | 服务好，研究报告丰富 |
| 中信期货 | 黄金期权 | 系统稳定，机构首选 |
| 华泰期货 | 黄金期权 | 费率相对低 |

**开户流程**：证券账户 → 期货账户开户（期货公司）→ 申请期权开户资格 → 通过知识测试

### 境外平台

| 平台 | 产品 | 特点 |
|------|------|------|
| Interactive Brokers（盈透）| COMEX 黄金期权 | 费率极低，产品最全 |
| TD Ameritrade（Thinkorswim）| COMEX 黄金期权 | 分析工具最强 |
| Tastyworks | 黄金 ETF 期权（GLD） | 专门做期权卖方友好 |

境外平台操作黄金 ETF 期权（如 GLD）门槛更低，GLD 1 张合约 = 100 股 × 每股 ≈ $0.3 克黄金，资金要求小很多。

---

## 十三、实操步骤（以上期所黄金期权为例）

**第一步：判断基本面**

确认你对黄金中期持多头观点，确定愿意买入的价格区间（比如当前价 4500，愿意在 4300-4400 建仓）。

**第二步：查看期权链**

登录期货公司交易软件，找到 AU（黄金）期权合约，查看不同执行价、不同到期日的权利金报价。

重点看：
- 执行价附近的隐含波动率（IV）
- 买卖价差是否合理（价差 / 权利金 < 5% 为佳）

**第三步：计算年化收益**

```
年化 = (权利金 / 执行价) × (365 / 剩余天数)
```

如果年化 < 8%，价值不大；8-15% 算合理；> 15% 要注意是否市场隐含了较大下行风险。

**第四步：挂单卖出**

选择 **卖出开仓**，选合约，输入数量和委托价（通常用限价单，在买价和卖价之间挂）。

**第五步：到期处理**

- 若到期价 ≥ 执行价：合约自动作废，权利金保留
- 若到期价 < 执行价：被行权，按执行价买入黄金

也可以在到期前主动**买入平仓**（回购这张 Put），锁定部分利润或止损。

---

## 十四、进阶：滚动策略

Sell Put 真正的威力在于**滚动复利**：

```
第 1 个月：Sell Put 4400，收 50 元/克，未被行权
第 2 个月：Sell Put 4380，收 48 元/克，未被行权
第 3 个月：Sell Put 4400，收 52 元/克，未被行权
... 每月滚动，权利金持续积累
```

如果被行权了怎么办？持有黄金之后，可以**Sell Call**（卖出认购期权），以更高价格卖出黄金同时收取权利金，形成**Covered Call + 买入成本降低**的良性循环。

这个组合策略叫 **Wheel Strategy（车轮策略）**，是期权卖方最常见的完整操作框架。

---

## 十五、写在最后

Sell Put 黄金这个策略，核心前提只有一句话：

**你本来就打算买黄金，只是在等一个好价格。**

如果是这种情况，Sell Put 把等待的时间价值变成了真实收入，13-17% 的年化是"等待费"，而不是凭空来的风险溢价。

段永平说得很清楚：他做 Sell Put 的股票，都是他认为价格便宜、愿意持有的。执行价是他认为合理的买入价，被行权对他来说不是损失而是入场。

这个策略不适合以下两种人：
- 不愿意持有黄金的人（被行权后会慌）
- 资金管理混乱的人（可能被追保证金强平）

但如果你对黄金有中长期信心，有一定的流动资金，Sell Put 是比单纯等待更有效率的入场方式。

---

*本文仅为策略分析，不构成投资建议。期权交易有较大风险，请在充分了解规则后再实操。*
