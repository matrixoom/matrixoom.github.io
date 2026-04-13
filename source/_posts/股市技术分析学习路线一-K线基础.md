---
title: 股市技术分析学习路线（一）：K线基础——从一根蜡烛读懂多空博弈
cover: /images/cover001.png
sticky: 0
comments: true
categories:
  - 金融投资
tags:
  - 技术分析
  - K线
  - 股票
  - 学习路线
date: 2026-04-11 12:00:00
---

# 股市技术分析学习路线（一）：K线基础——从一根蜡烛读懂多空博弈

> 这是一切的根基。K线是市场的"字母表"，后面的形态、指标都建立在对K线语言的理解之上。这个阶段务必扎实，宁可多花一周也不要囫囵吞枣。

---

## 本阶段学习目标

- 看到任何一根K线，能在3秒内判断多空力量对比
- 看到K线组合，能判断它是反转信号还是持续信号
- 能在任何股票图表上准确画出趋势线
- 能标出关键支撑位和阻力位
- 理解不同时间周期K线的含义和关系

---

## 一、K线的诞生与本质

### 1.1 起源

K线（蜡烛图）起源于18世纪日本大阪的稻米期货市场，由米商**本间宗久**发明。他利用K线记录米价波动，积累了巨额财富。20世纪80年代末，史蒂夫·尼森将其引入西方，从此成为全球金融市场最通用的图表形式。

### 1.2 K线的本质：多空博弈的可视化

每一根K线记录的是一个时间周期内（1分钟/5分钟/日/周/月）**买方和卖方战斗的结果**：

- **实体大小** = 胜方优势大小
- **影线长度** = 败方反击力度
- **颜色** = 谁赢了这场战斗

---

## 二、单根K线的结构（逐层拆解）

### 2.1 四个价格

每根K线由4个价格确定：**开盘价、收盘价、最高价、最低价**

<svg viewBox="0 0 400 320" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <!-- 背景 -->
  <rect width="400" height="320" fill="#1a1f2e" rx="8"/>
  <!-- 阳线 -->
  <line x1="120" y1="40" x2="120" y2="280" stroke="#888" stroke-width="1.5"/>
  <rect x="90" y="90" width="60" height="80" fill="#e74c3c" rx="2"/>
  <line x1="120" y1="90" x2="120" y2="40" stroke="#888" stroke-width="1.5"/>
  <line x1="120" y1="170" x2="120" y2="280" stroke="#888" stroke-width="1.5"/>
  <!-- 标注 -->
  <text x="170" y="48" fill="#e0e0e0" font-size="13" font-family="sans-serif">← 最高价</text>
  <line x1="160" y1="44" x2="155" y2="44" stroke="#e0e0e0" stroke-width="1"/>
  <text x="170" y="98" fill="#e0e0e0" font-size="13" font-family="sans-serif">← 收盘价</text>
  <line x1="160" y1="94" x2="150" y2="94" stroke="#e0e0e0" stroke-width="1"/>
  <text x="170" y="162" fill="#e0e0e0" font-size="13" font-family="sans-serif">← 开盘价</text>
  <line x1="160" y1="158" x2="150" y2="158" stroke="#e0e0e0" stroke-width="1"/>
  <text x="170" y="285" fill="#e0e0e0" font-size="13" font-family="sans-serif">← 最低价</text>
  <line x1="160" y1="281" x2="155" y2="281" stroke="#e0e0e0" stroke-width="1"/>
  <!-- 区间标注 -->
  <text x="50" y="68" fill="#aaa" font-size="11" font-family="sans-serif">上影线</text>
  <line x1="72" y1="64" x2="86" y2="64" stroke="#aaa" stroke-width="1"/>
  <text x="38" y="138" fill="#e74c3c" font-size="11" font-family="sans-serif">实体</text>
  <text x="38" y="153" fill="#e74c3c" font-size="11" font-family="sans-serif">(阳线)</text>
  <text x="50" y="230" fill="#aaa" font-size="11" font-family="sans-serif">下影线</text>
  <line x1="72" y1="226" x2="86" y2="226" stroke="#aaa" stroke-width="1"/>
  <!-- 标题 -->
  <text x="120" y="310" fill="#e0e0e0" font-size="13" font-family="sans-serif" text-anchor="middle">阳线（收盘>开盘）</text>
</svg>

### 2.2 阳线与阴线

**中国A股惯例**：🔴 红色=阳线（涨）| 🟢 绿色=阴线（跌）

<svg viewBox="0 0 600 320" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="600" height="320" fill="#1a1f2e" rx="8"/>
  <!-- 阳线 -->
  <line x1="140" y1="50" x2="140" y2="80" stroke="#aaa" stroke-width="1.5"/>
  <rect x="110" y="80" width="60" height="120" fill="#e74c3c" rx="2"/>
  <line x1="140" y1="200" x2="140" y2="270" stroke="#aaa" stroke-width="1.5"/>
  <text x="140" y="300" fill="#e74c3c" font-size="14" font-family="sans-serif" text-anchor="middle">阳线（涨）🔴</text>
  <!-- 阴线 -->
  <line x1="440" y1="50" x2="440" y2="110" stroke="#aaa" stroke-width="1.5"/>
  <rect x="410" y="110" width="60" height="100" fill="#2ecc71" rx="2"/>
  <line x1="440" y1="210" x2="440" y2="270" stroke="#aaa" stroke-width="1.5"/>
  <text x="440" y="300" fill="#2ecc71" font-size="14" font-family="sans-serif" text-anchor="middle">阴线（跌）🟢</text>
  <!-- 阳线标注 -->
  <text x="185" y="90" fill="#ccc" font-size="11" font-family="sans-serif">收盘</text>
  <text x="185" y="200" fill="#ccc" font-size="11" font-family="sans-serif">开盘</text>
  <text x="80" y="165" fill="#e74c3c" font-size="11" font-family="sans-serif">实体</text>
  <text x="80" y="68" fill="#aaa" font-size="11" font-family="sans-serif">上影线</text>
  <text x="80" y="248" fill="#aaa" font-size="11" font-family="sans-serif">下影线</text>
  <!-- 阴线标注 -->
  <text x="485" y="120" fill="#ccc" font-size="11" font-family="sans-serif">开盘</text>
  <text x="485" y="210" fill="#ccc" font-size="11" font-family="sans-serif">收盘</text>
  <text x="370" y="170" fill="#2ecc71" font-size="11" font-family="sans-serif">实体</text>
  <text x="370" y="80" fill="#aaa" font-size="11" font-family="sans-serif">上影线</text>
  <text x="370" y="250" fill="#aaa" font-size="11" font-family="sans-serif">下影线</text>
</svg>

**关键理解**：
- **实体越大** → 多空某一方的优势越明显
- **上影线长** → 价格上攻过但被打回来了，上方有压力
- **下影线长** → 价格下跌过但被拉回来了，下方有支撑
- **没有影线** → 一方完全碾压，没有给对手任何反击机会

### 2.3 K线的"语言"——从形状读多空

| K线形态 | 形状特征 | 多空解读 | 信号含义 |
|---------|----------|----------|----------|
| **大阳线** | 长实体，几乎无影线 | 多方碾压，空方无还手之力 | 强势买入信号 |
| **大阴线** | 长实体，几乎无影线 | 空方碾压，多方无还手之力 | 强势卖出信号 |
| **长上影阳线** | 阳线+长上影线 | 多方进攻后遭空方反击 | 上涨受阻⚠️ |
| **长下影阳线** | 阳线+长下影线 | 空方打压后被多方收复 | 下方支撑强✅ |
| **长上影阴线** | 阴线+长上影线 | 多方冲高后被空方击溃 | 转弱信号🔴 |
| **长下影阴线** | 阴线+长下影线 | 空方打压但底部有承接 | 可能见底🔄 |
| **十字星** | 极小/无实体，上下影线均有 | 多空势均力敌 | 犹豫/变盘⚠️ |
| **T字线** | 无上影线，有下影线 | 开盘=收盘=最高价，下方有支撑 | 底部出现=看涨 |
| **倒T字线** | 无下影线，有上影线 | 开盘=收盘=最低价，上方有压力 | 顶部出现=看跌 |
| **一字线** | 开盘=收盘=最高=最低 | 涨停/跌停，无任何交易空间 | 极端信号 |

---

## 三、K线形态详解——反转形态（重点）

> 反转形态是最有实战价值的K线组合，学好了能帮你抓住趋势的拐点。

### 3.1 锤子线（Hammer）⭐⭐⭐

**出现在下跌趋势末端，是底部反转信号**

<svg viewBox="0 0 300 300" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="300" height="300" fill="#1a1f2e" rx="8"/>
  <!-- 下跌趋势线 -->
  <polyline points="30,40 80,70 130,110 180,150 220,200 240,250" fill="none" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <!-- 锤子线 -->
  <rect x="230" y="110" width="30" height="20" fill="#e74c3c" rx="1"/>
  <line x1="245" y1="110" x2="245" y2="100" stroke="#aaa" stroke-width="1.5"/>
  <line x1="245" y1="130" x2="245" y2="230" stroke="#aaa" stroke-width="1.5"/>
  <!-- 标注 -->
  <text x="270" y="122" fill="#e74c3c" font-size="11" font-family="sans-serif">实体</text>
  <text x="270" y="180" fill="#aaa" font-size="11" font-family="sans-serif">下影线</text>
  <text x="270" y="195" fill="#aaa" font-size="11" font-family="sans-serif">(&gt;实体2倍)</text>
  <!-- 箭头 -->
  <path d="M245,240 L240,230 L250,230 Z" fill="#e74c3c"/>
  <text x="255" y="252" fill="#e74c3c" font-size="12" font-family="sans-serif">反转信号</text>
</svg>

**判断标准**：
1. 出现在明确的下跌趋势中
2. 实体很小，位于K线上端
3. 下影线长度 ≥ 实体的2倍
4. 上影线极短或没有
5. 实体颜色不重要，但阳线锤子信号更强

**实战要点**：
- 下影线越长，反转信号越强（说明空方砸下去被多方拉回来的力度大）
- 需要后续K线确认——下一根如果是阳线，确认信号有效
- 成交量如果在锤子线当天放大，信号更可靠

### 3.2 倒锤子（Inverted Hammer）⭐⭐

**出现在下跌趋势末端，信号较弱需确认**

<svg viewBox="0 0 300 300" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="300" height="300" fill="#1a1f2e" rx="8"/>
  <polyline points="30,40 80,70 130,110 180,150 220,200 240,250" fill="none" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <!-- 倒锤子 -->
  <rect x="230" y="200" width="30" height="20" fill="#e74c3c" rx="1"/>
  <line x1="245" y1="200" x2="245" y2="100" stroke="#aaa" stroke-width="1.5"/>
  <line x1="245" y1="220" x2="245" y2="235" stroke="#aaa" stroke-width="1.5"/>
  <text x="270" y="210" fill="#e74c3c" font-size="11" font-family="sans-serif">实体</text>
  <text x="270" y="155" fill="#aaa" font-size="11" font-family="sans-serif">上影线</text>
  <text x="270" y="170" fill="#aaa" font-size="11" font-family="sans-serif">(&gt;实体2倍)</text>
</svg>

与锤子线的区别：长影线在上方而不是下方。信号较弱，因为上影线说明多方曾试图反攻但被打回，还需要后续阳线确认。

### 3.3 悬挂人（Hanging Man）⭐⭐⭐

**形状与锤子线完全一样，但出现在上涨趋势末端**

<svg viewBox="0 0 500 280" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="500" height="280" fill="#1a1f2e" rx="8"/>
  <!-- 上涨趋势+锤子线 -->
  <polyline points="20,230 60,200 100,175 140,140 180,120 210,95" fill="none" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <rect x="200" y="95" width="22" height="15" fill="#e74c3c" rx="1"/>
  <line x1="211" y1="95" x2="211" y2="85" stroke="#aaa" stroke-width="1.5"/>
  <line x1="211" y1="110" x2="211" y2="180" stroke="#aaa" stroke-width="1.5"/>
  <text x="240" y="145" fill="#2ecc71" font-size="12" font-family="sans-serif">锤子线</text>
  <text x="240" y="162" fill="#2ecc71" font-size="11" font-family="sans-serif">(底部反转)</text>
  <!-- 下跌趋势+悬挂人 -->
  <polyline points="280,50 320,80 360,110 400,145 430,175 450,210" fill="none" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <rect x="440" y="210" width="22" height="15" fill="#2ecc71" rx="1"/>
  <line x1="451" y1="210" x2="451" y2="200" stroke="#aaa" stroke-width="1.5"/>
  <line x1="451" y1="225" x2="451" y2="260" stroke="#aaa" stroke-width="1.5"/>
  <text x="230" y="35" fill="#e0e0e0" font-size="11" font-family="sans-serif">下跌末端</text>
  <text x="480" y="240" fill="#e74c3c" font-size="12" font-family="sans-serif">悬挂人</text>
  <text x="480" y="257" fill="#e74c3c" font-size="11" font-family="sans-serif">(顶部反转)</text>
  <text x="390" y="35" fill="#e0e0e0" font-size="11" font-family="sans-serif">上涨末端</text>
</svg>

**核心教训**：K线形态不能只看形状，**位置决定含义！**

### 3.4 吞没形态（Engulfing）⭐⭐⭐⭐

**最强的单日反转信号之一**

<svg viewBox="0 0 600 280" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="600" height="280" fill="#1a1f2e" rx="8"/>
  <!-- 看涨吞没 -->
  <text x="130" y="30" fill="#e0e0e0" font-size="14" font-family="sans-serif" text-anchor="middle">看涨吞没</text>
  <!-- 小阴线 -->
  <rect x="100" y="100" width="24" height="80" fill="#2ecc71" rx="1"/>
  <line x1="112" y1="100" x2="112" y2="85" stroke="#aaa" stroke-width="1"/>
  <line x1="112" y1="180" x2="112" y2="200" stroke="#aaa" stroke-width="1"/>
  <!-- 大阳线吞没 -->
  <rect x="140" y="70" width="30" height="130" fill="#e74c3c" rx="1"/>
  <line x1="155" y1="70" x2="155" y2="50" stroke="#aaa" stroke-width="1"/>
  <line x1="155" y1="200" x2="155" y2="220" stroke="#aaa" stroke-width="1"/>
  <text x="130" y="260" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">阴线</text>
  <text x="155" y="260" fill="#e74c3c" font-size="11" font-family="sans-serif" text-anchor="middle">阳线</text>
  <!-- 看跌吞没 -->
  <text x="430" y="30" fill="#e0e0e0" font-size="14" font-family="sans-serif" text-anchor="middle">看跌吞没</text>
  <!-- 小阳线 -->
  <rect x="400" y="100" width="24" height="80" fill="#e74c3c" rx="1"/>
  <line x1="412" y1="100" x2="412" y2="85" stroke="#aaa" stroke-width="1"/>
  <line x1="412" y1="180" x2="412" y2="200" stroke="#aaa" stroke-width="1"/>
  <!-- 大阴线吞没 -->
  <rect x="440" y="60" width="30" height="140" fill="#2ecc71" rx="1"/>
  <line x1="455" y1="60" x2="455" y2="40" stroke="#aaa" stroke-width="1"/>
  <line x1="455" y1="200" x2="455" y2="230" stroke="#aaa" stroke-width="1"/>
  <text x="430" y="260" fill="#e74c3c" font-size="11" font-family="sans-serif" text-anchor="middle">阳线</text>
  <text x="455" y="260" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">阴线</text>
</svg>

**判断标准**：
1. 必须出现在明确的趋势中
2. 第二根K线的实体**完全包含**第一根K线的实体
3. 两根K线颜色相反
4. 第二根K线成交量显著放大则更可靠

**为什么强？** 看涨吞没意味着空方先发力（阴线），但多方随后以更大力度反击（阳线吞没），完全收复失地并创出新高——这是多空力量根本性逆转的信号。

### 3.5 早晨之星（Morning Star）⭐⭐⭐⭐

**最经典的底部反转组合，由三根K线构成**

<svg viewBox="0 0 500 300" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="500" height="300" fill="#1a1f2e" rx="8"/>
  <!-- 大阴线 -->
  <rect x="80" y="60" width="40" height="140" fill="#2ecc71" rx="1"/>
  <line x1="100" y1="60" x2="100" y2="45" stroke="#aaa" stroke-width="1.5"/>
  <line x1="100" y1="200" x2="100" y2="220" stroke="#aaa" stroke-width="1.5"/>
  <text x="100" y="240" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">大阴线</text>
  <!-- 十字星 -->
  <line x1="190" y1="170" x2="190" y2="220" stroke="#aaa" stroke-width="1.5"/>
  <rect x="183" y="188" width="14" height="8" fill="#f1c40f" rx="1"/>
  <line x1="190" y1="180" x2="190" y2="170" stroke="#aaa" stroke-width="1.5"/>
  <text x="190" y="240" fill="#f1c40f" font-size="11" font-family="sans-serif" text-anchor="middle">十字星</text>
  <!-- 大阳线 -->
  <rect x="270" y="90" width="40" height="120" fill="#e74c3c" rx="1"/>
  <line x1="290" y1="90" x2="290" y2="70" stroke="#aaa" stroke-width="1.5"/>
  <line x1="290" y1="210" x2="290" y2="225" stroke="#aaa" stroke-width="1.5"/>
  <text x="290" y="240" fill="#e74c3c" font-size="11" font-family="sans-serif" text-anchor="middle">大阳线</text>
  <!-- 步骤标注 -->
  <text x="100" y="272" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">①空方主导</text>
  <text x="190" y="272" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">②犹豫平衡</text>
  <text x="290" y="272" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">③多方接管</text>
  <!-- 星号 -->
  <text x="190" y="162" fill="#f1c40f" font-size="18" font-family="sans-serif" text-anchor="middle">★</text>
</svg>

**判断标准**：
1. 第一根：大阴线（下跌趋势的延续）
2. 第二根：小实体K线（十字星最佳），与第一根实体之间有向下缺口更佳
3. 第三根：大阳线，实体深入第一根阴线实体的50%以上

**实战要点**：
- 中间的十字星代表"黎明前的黑暗"，多空短暂平衡
- 第三根阳线实体越大，信号越可靠
- 如果第三根阳线成交量显著放大，确认度极高

### 3.6 黄昏之星（Evening Star）⭐⭐⭐⭐

早晨之星的镜像，顶部反转信号。黄昏之星在A股中出现的频率和可靠性都高于早晨之星（因为A股上涨慢、下跌快的特点）。

### 3.7 刺透形态（Piercing Line）⭐⭐⭐

**看涨吞没的"弱化版"**——阳线收盘只超过了阴线实体的50%，没有完全吞没。信号稍弱但同样有效。收盘超过前阴实体的50%是底线，如果连50%都没过，那不算刺透形态。

### 3.8 乌云盖顶（Dark Cloud Cover）⭐⭐⭐

刺透形态的镜像，顶部信号——阴线收盘低于阳线实体中点。

---

## 四、K线形态详解——持续形态

### 4.1 三白兵（Three White Soldiers）⭐⭐⭐

<svg viewBox="0 0 300 300" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="300" height="300" fill="#1a1f2e" rx="8"/>
  <!-- 第一根阳线 -->
  <rect x="70" y="210" width="35" height="50" fill="#e74c3c" rx="1"/>
  <line x1="87" y1="210" x2="87" y2="200" stroke="#aaa" stroke-width="1"/>
  <line x1="87" y1="260" x2="87" y2="270" stroke="#aaa" stroke-width="1"/>
  <!-- 第二根阳线 -->
  <rect x="130" y="150" width="35" height="50" fill="#e74c3c" rx="1"/>
  <line x1="147" y1="150" x2="147" y2="140" stroke="#aaa" stroke-width="1"/>
  <line x1="147" y1="200" x2="147" y2="215" stroke="#aaa" stroke-width="1"/>
  <!-- 第三根阳线 -->
  <rect x="190" y="90" width="35" height="50" fill="#e74c3c" rx="1"/>
  <line x1="207" y1="90" x2="207" y2="80" stroke="#aaa" stroke-width="1"/>
  <line x1="207" y1="140" x2="207" y2="155" stroke="#aaa" stroke-width="1"/>
  <!-- 标注 -->
  <text x="87" y="288" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">①</text>
  <text x="147" y="288" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">②</text>
  <text x="207" y="288" fill="#aaa" font-size="10" font-family="sans-serif" text-anchor="middle">③</text>
</svg>

**判断标准**：连续三根阳线，每根开盘在前一根实体范围内，每根收盘创新高，影线较短。

**⚠️ 注意**：如果第三根阳线实体缩小、上影线加长 → "三白兵停滞"，上涨可能结束。

### 4.2 上升三法（Rising Three Methods）⭐⭐⭐

<svg viewBox="0 0 500 250" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="500" height="250" fill="#1a1f2e" rx="8"/>
  <!-- 大阳线① -->
  <rect x="50" y="80" width="40" height="100" fill="#e74c3c" rx="1"/>
  <line x1="70" y1="80" x2="70" y2="65" stroke="#aaa" stroke-width="1"/>
  <line x1="70" y1="180" x2="70" y2="195" stroke="#aaa" stroke-width="1"/>
  <!-- 小阴线② -->
  <rect x="120" y="100" width="25" height="30" fill="#2ecc71" rx="1"/>
  <line x1="132" y1="100" x2="132" y2="92" stroke="#aaa" stroke-width="1"/>
  <line x1="132" y1="130" x2="132" y2="138" stroke="#aaa" stroke-width="1"/>
  <!-- 小阴线③ -->
  <rect x="170" y="115" width="25" height="30" fill="#2ecc71" rx="1"/>
  <line x1="182" y1="115" x2="182" y2="107" stroke="#aaa" stroke-width="1"/>
  <line x1="182" y1="145" x2="182" y2="153" stroke="#aaa" stroke-width="1"/>
  <!-- 小阴线④ -->
  <rect x="220" y="125" width="25" height="30" fill="#2ecc71" rx="1"/>
  <line x1="232" y1="125" x2="232" y2="117" stroke="#aaa" stroke-width="1"/>
  <line x1="232" y1="155" x2="232" y2="163" stroke="#aaa" stroke-width="1"/>
  <!-- 大阳线⑤ -->
  <rect x="280" y="60" width="40" height="110" fill="#e74c3c" rx="1"/>
  <line x1="300" y1="60" x2="300" y2="45" stroke="#aaa" stroke-width="1"/>
  <line x1="300" y1="170" x2="300" y2="190" stroke="#aaa" stroke-width="1"/>
  <!-- 标注 -->
  <text x="70" y="228" fill="#e74c3c" font-size="11" font-family="sans-serif" text-anchor="middle">①大阳</text>
  <text x="132" y="228" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">②</text>
  <text x="182" y="228" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">③</text>
  <text x="232" y="228" fill="#2ecc71" font-size="11" font-family="sans-serif" text-anchor="middle">④</text>
  <text x="300" y="228" fill="#e74c3c" font-size="11" font-family="sans-serif" text-anchor="middle">⑤大阳</text>
  <!-- 虚线框 -->
  <rect x="105" y="88" width="155" height="82" fill="none" stroke="#f39c12" stroke-width="1" stroke-dasharray="4,3" rx="3"/>
  <text x="180" y="82" fill="#f39c12" font-size="10" font-family="sans-serif" text-anchor="middle">回调在大阳线范围内</text>
</svg>

**含义**：多方大举进攻(①)→空方小幅反击(②③④)但始终在大阳线范围内→多方再次进攻(⑤)

**关键判断**：中间的小阴线不能跌破第一根大阳线的低点，否则形态失败。

### 4.3 下降三法

上升三法的镜像，下跌中继信号，中间小阳线反弹幅度有限。

---

## 五、K线形态详解——注意形态

### 5.1 十字星（Doji）⭐⭐⭐

**开盘价≈收盘价，多空打成平手**

<svg viewBox="0 0 600 180" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="600" height="180" fill="#1a1f2e" rx="8"/>
  <!-- 标准十字星 -->
  <line x1="75" y1="30" x2="75" y2="130" stroke="#f1c40f" stroke-width="2"/>
  <line x1="55" y1="80" x2="95" y2="80" stroke="#f1c40f" stroke-width="2"/>
  <text x="75" y="160" fill="#f1c40f" font-size="12" font-family="sans-serif" text-anchor="middle">标准十字星</text>
  <!-- 长腿十字星 -->
  <line x1="200" y1="20" x2="200" y2="140" stroke="#f1c40f" stroke-width="2"/>
  <line x1="180" y1="80" x2="220" y2="80" stroke="#f1c40f" stroke-width="2"/>
  <text x="200" y="160" fill="#f1c40f" font-size="12" font-family="sans-serif" text-anchor="middle">长腿十字星</text>
  <!-- 蜻蜓十字星 -->
  <line x1="325" y1="50" x2="325" y2="140" stroke="#f1c40f" stroke-width="2"/>
  <circle cx="325" cy="50" r="3" fill="#f1c40f"/>
  <text x="325" y="160" fill="#f1c40f" font-size="12" font-family="sans-serif" text-anchor="middle">蜻蜓十字星</text>
  <!-- 墓碑十字星 -->
  <line x1="450" y1="20" x2="450" y2="110" stroke="#f1c40f" stroke-width="2"/>
  <circle cx="450" cy="110" r="3" fill="#f1c40f"/>
  <text x="450" y="160" fill="#f1c40f" font-size="12" font-family="sans-serif" text-anchor="middle">墓碑十字星</text>
</svg>

| 十字星类型 | 特征 | 出现在底部 | 出现在顶部 |
|-----------|------|-----------|-----------|
| 标准十字星 | 上下影线等长 | 犹豫，等待方向 | 变盘预警 |
| 长腿十字星 | 上下影线很长 | 波动剧烈但无结果 | 市场剧烈震荡 |
| 蜻蜓十字星 | 只有下影线 | 底部反转信号 | — |
| 墓碑十字星 | 只有上影线 | — | 顶部反转信号 |

**十字星的实战价值**：
- 在趋势末端出现十字星 → 变盘预警（不是操作信号！）
- 十字星 + 后续确认K线 → 才能形成操作依据
- 在震荡区间中间的十字星 → 无意义

### 5.2 纺锤线（Spinning Top）⭐⭐

实体很小，但上下影线都有。与十字星的区别：十字星几乎没有实体，纺锤线有小实体但很小。

**含义**：多空双方都没有取得明显优势，市场犹豫。如果出现在趋势末端，可能预示反转。

### 5.3 流星线（Shooting Star）⭐⭐⭐

**出现在上涨趋势末端，长上影线+小实体在底部**

<svg viewBox="0 0 300 280" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="300" height="280" fill="#1a1f2e" rx="8"/>
  <!-- 上涨趋势 -->
  <polyline points="20,250 70,210 120,170 170,130 210,90" fill="none" stroke="#555" stroke-width="1" stroke-dasharray="4,3"/>
  <!-- 流星线 -->
  <rect x="200" y="180" width="24" height="16" fill="#2ecc71" rx="1"/>
  <line x1="212" y1="180" x2="212" y2="60" stroke="#aaa" stroke-width="1.5"/>
  <line x1="212" y1="196" x2="212" y2="210" stroke="#aaa" stroke-width="1.5"/>
  <text x="240" y="130" fill="#aaa" font-size="11" font-family="sans-serif">长上影线</text>
  <text x="240" y="188" fill="#2ecc71" font-size="11" font-family="sans-serif">小实体</text>
</svg>

**解读**：多方曾经大幅上攻，但被空方强力打回，收盘回到开盘价附近。上攻失败的信号。

---

## 六、趋势线——最重要的画线工具

### 6.1 什么是趋势

**道氏理论定义**：

<svg viewBox="0 0 600 250" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="600" height="250" fill="#1a1f2e" rx="8"/>
  <!-- 上升趋势 -->
  <text x="140" y="25" fill="#e0e0e0" font-size="13" font-family="sans-serif" text-anchor="middle">上升趋势：更高的高点+更高的低点</text>
  <polyline points="40,200 80,170 120,190 160,140 200,160 240,100 280,130 320,60" fill="none" stroke="#e74c3c" stroke-width="2"/>
  <!-- 趋势线 -->
  <line x1="80" y1="170" x2="320" y2="120" stroke="#3498db" stroke-width="1.5" stroke-dasharray="6,3"/>
  <!-- 标注 -->
  <text x="42" y="195" fill="#aaa" font-size="10">L1</text>
  <text x="122" y="203" fill="#aaa" font-size="10">L2</text>
  <text x="202" y="172" fill="#aaa" font-size="10">L3</text>
  <text x="75" y="166" fill="#3498db" font-size="10">H1</text>
  <text x="155" y="136" fill="#3498db" font-size="10">H2</text>
  <text x="235" y="96" fill="#3498db" font-size="10">H3</text>
  <!-- 下降趋势 -->
  <text x="460" y="25" fill="#e0e0e0" font-size="13" font-family="sans-serif" text-anchor="middle">下降趋势：更低的高点+更低的低点</text>
  <polyline points="350,60 390,90 430,75 470,130 510,110 550,170 580,155" fill="none" stroke="#2ecc71" stroke-width="2"/>
  <line x1="390" y1="90" x2="580" y2="160" stroke="#3498db" stroke-width="1.5" stroke-dasharray="6,3"/>
  <text x="385" y="86" fill="#3498db" font-size="10">H1</text>
  <text x="465" y="126" fill="#3498db" font-size="10">H2</text>
  <text x="545" y="166" fill="#3498db" font-size="10">H3</text>
</svg>

### 6.2 趋势线的画法

**上升趋势线**：连接两个或以上的**低点**
**下降趋势线**：连接两个或以上的**高点**

**画线规则**：
1. 至少需要两个点才能画线
2. 第三个点触线反弹 → 趋势线得到确认
3. 触及次数越多，趋势线越有效
4. 趋势线的时间跨度越长，越重要

### 6.3 趋势线的突破

**有效突破的判断标准**：

| 条件 | 说明 |
|------|------|
| **收盘价突破** | 仅盘中突破不算，收盘价站上/下趋势线才有效 |
| **突破幅度** | 超过趋势线3%以上更可靠 |
| **成交量** | 向上突破需要放量确认，向下突破不一定需要 |
| **时间** | 突破后至少3个交易日没有回到趋势线另一侧 |

**假突破**：价格短暂突破趋势线后迅速回归，通常是主力诱多/诱空。

### 6.4 趋势通道

<svg viewBox="0 0 500 250" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="500" height="250" fill="#1a1f2e" rx="8"/>
  <!-- 上升通道 -->
  <line x1="40" y1="200" x2="460" y2="80" stroke="#3498db" stroke-width="1.5" stroke-dasharray="6,3"/>
  <line x1="40" y1="160" x2="460" y2="40" stroke="#e74c3c" stroke-width="1.5" stroke-dasharray="6,3"/>
  <!-- 价格线 -->
  <polyline points="40,180 80,170 120,145 160,140 200,110 240,115 280,85 320,90 360,60 400,55 440,30" fill="none" stroke="#f1c40f" stroke-width="2"/>
  <!-- 标注 -->
  <text x="462" y="75" fill="#3498db" font-size="11" font-family="sans-serif">下轨(支撑)</text>
  <text x="462" y="38" fill="#e74c3c" font-size="11" font-family="sans-serif">上轨(阻力)</text>
  <text x="240" y="230" fill="#e0e0e0" font-size="13" font-family="sans-serif" text-anchor="middle">上升通道：价格在两条平行线之间运行</text>
</svg>

**通道的实战用法**：
- 价格触及上轨 → 考虑减仓/止盈
- 价格触及下轨 → 考虑买入
- 价格突破上轨 → 加速上涨，但注意可能过度延伸
- 价格跌破下轨 → 趋势可能改变

---

## 七、支撑与阻力——价格的记忆

### 7.1 核心概念

<svg viewBox="0 0 500 220" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="500" height="220" fill="#1a1f2e" rx="8"/>
  <!-- 阻力线 -->
  <line x1="30" y1="50" x2="470" y2="50" stroke="#e74c3c" stroke-width="2" stroke-dasharray="8,4"/>
  <text x="475" y="54" fill="#e74c3c" font-size="11" font-family="sans-serif">阻力位(R)</text>
  <!-- 支撑线 -->
  <line x1="30" y1="180" x2="470" y2="180" stroke="#2ecc71" stroke-width="2" stroke-dasharray="8,4"/>
  <text x="475" y="184" fill="#2ecc71" font-size="11" font-family="sans-serif">支撑位(S)</text>
  <!-- 价格走势 -->
  <polyline points="50,160 100,80 140,55 180,110 220,55 260,80 300,120 340,55 380,100 420,55" fill="none" stroke="#f1c40f" stroke-width="2"/>
  <!-- 触点标注 -->
  <circle cx="140" cy="55" r="4" fill="#e74c3c" opacity="0.7"/>
  <circle cx="220" cy="55" r="4" fill="#e74c3c" opacity="0.7"/>
  <circle cx="340" cy="55" r="4" fill="#e74c3c" opacity="0.7"/>
  <circle cx="420" cy="55" r="4" fill="#e74c3c" opacity="0.7"/>
  <text x="150" y="20" fill="#e74c3c" font-size="10">多次触及=强阻力</text>
</svg>

### 7.2 支撑/阻力的形成原因

| 原因 | 解释 |
|------|------|
| **心理价位** | 整数关口（3000点、100元等）人们天然关注 |
| **历史成交密集区** | 大量买卖曾经发生的位置，套牢盘或获利盘集中 |
| **前期高点/低点** | 曾经受阻/止跌的位置，市场有"记忆" |
| **缺口** | 跳空缺口处没有成交，回来时会有补缺口行为 |
| **移动平均线** | MA20/MA60等天然形成动态支撑/阻力 |
| **黄金分割位** | 0.382/0.5/0.618回撤位 |

### 7.3 角色互换（最重要的概念之一）

<svg viewBox="0 0 600 200" xmlns="http://www.w3.org/2000/svg" style="display:block;margin:20px auto;max-width:100%;">
  <rect width="600" height="200" fill="#1a1f2e" rx="8"/>
  <!-- 阻力变支撑 -->
  <text x="140" y="20" fill="#e0e0e0" font-size="12" font-family="sans-serif" text-anchor="middle">阻力位突破后 → 变成支撑位</text>
  <line x1="30" y1="70" x2="250" y2="70" stroke="#e74c3c" stroke-width="2" stroke-dasharray="5,3"/>
  <text x="255" y="74" fill="#e74c3c" font-size="10">原阻力</text>
  <polyline points="30,90 60,72 90,90 120,72 150,55 180,30 210,70 240,70" fill="none" stroke="#f1c40f" stroke-width="2"/>
  <text x="150" y="175" fill="#2ecc71" font-size="10" text-anchor="middle">突破后回踩不破 → 阻力变支撑 ✅</text>
  <!-- 支撑变阻力 -->
  <text x="440" y="20" fill="#e0e0e0" font-size="12" font-family="sans-serif" text-anchor="middle">支撑位跌破后 → 变成阻力位</text>
  <line x1="330" y1="120" x2="570" y2="120" stroke="#2ecc71" stroke-width="2" stroke-dasharray="5,3"/>
  <text x="575" y="124" fill="#2ecc71" font-size="10">原支撑</text>
  <polyline points="330,100 360,120 390,100 420,120 450,140 480,170 510,120 540,120" fill="none" stroke="#f1c40f" stroke-width="2"/>
  <text x="440" y="192" fill="#e74c3c" font-size="10" text-anchor="middle">跌破后反弹不过 → 支撑变阻力 ❌</text>
</svg>

**为什么？** 因为在支撑位买入的人，当支撑被跌破后变成套牢盘。等价格再涨回来时，这些人会急于回本卖出，形成新的卖压。

### 7.4 支撑/阻力的强度判断

| 强度因素 | 强支撑/阻力 | 弱支撑/阻力 |
|----------|------------|------------|
| 触及次数 | 3次以上 | 1-2次 |
| 时间跨度 | 数月甚至数年 | 几天 |
| 成交量 | 触及时成交量巨大 | 触及时成交量一般 |
| 周期级别 | 周线/月线级别 | 5分钟/15分钟级别 |
| 整数关口 | 大整数（3000/10000） | 非整数 |

---

## 八、时间周期——不同级别的K线

### 8.1 常用时间周期

| 周期 | 适用人群 | 特点 |
|------|----------|------|
| 1分钟/5分钟 | 日内交易者 | 噪音大，信号频繁但假信号多 |
| 15分钟/30分钟 | 短线交易者 | 日内波段，需盯盘 |
| 60分钟 | 短中线 | 较好的短线操作周期 |
| **日线** | **中线交易者** | **最常用，信号可靠性最好** |
| 周线 | 中长线 | 过滤日线噪音，大趋势判断 |
| 月线 | 长线投资者 | 超大级别方向，年度级别支撑/阻力 |

### 8.2 多周期共振（进阶概念）

**核心原则**：多个时间周期同时发出相同方向信号 → 可靠性大幅提升

- 月线：上升趋势 ✅ → 周线：上升趋势 ✅ → 日线：买入信号 ✅ = **三周期共振，强买入信号！**
- 月线：上升趋势 ✅ → 周线：下降趋势 ❌ → 日线：买入信号 = **只有日线信号，周线不支持，谨慎操作**

**实战建议**：
1. 先看大周期定方向（周线/月线判断大趋势）
2. 再看小周期找入场点（日线/60分钟找买卖点）
3. 大周期信号比小周期信号重要得多

### 8.3 周期之间的转换

- **1根周线 = 5根日线**（一周5个交易日）
- **1根月线 ≈ 20根日线**（一月约20个交易日）
- **1根日线 = 4根60分钟线**

这意味着：周线上的一根K线形态，比日线上同等形态重要得多。

---

## 九、K线实战练习方法

### 9.1 每日必做

1. **翻看30只股票的日K线**（可以用同花顺的涨幅榜和跌幅榜各看15只）
2. **标注K线形态**：在图上标出你认识的形态（锤子线、吞没、十字星等）
3. **验证**：标注后往后看，这个形态是否真的带来了反转/持续？

### 9.2 盲测训练法

1. 打开任意股票的日K线图
2. 用纸片/工具遮住右半部分
3. 逐根"揭开"K线，在揭开下一根之前判断：
   - 当前趋势是什么？
   - 最新K线是什么形态？
   - 下一根K线你预期会怎样？
4. 揭开验证，记录判断对了几次

**目标**：连续判断20次，正确率超过60%

### 9.3 常见错误与纠正

| 错误 | 纠正 |
|------|------|
| 只看K线形状不看位置 | 先判断趋势，再看K线是反转还是持续 |
| 孤立看一根K线 | K线要结合前后几根一起看，组合才有意义 |
| 忽略成交量 | K线形态 + 量能配合才可靠 |
| 在小周期上过度交易 | 初学者应该只看日线级别 |
| 看到信号就立即操作 | 信号出现后等待确认再操作 |
| 觉得每个K线都有信号 | 大部分K线是无意义的普通K线，只有关键位置才有信号 |

---

## 十、阶段一自我测试

完成以下测试后，就可以进入阶段二：

### 基础题（必须全对）
1. 一根大阳线+长上影线，说明什么？
2. 锤子线和悬挂人形状一样，为什么信号相反？
3. 早晨之星由哪三根K线构成？
4. 上升趋势线连接的是高点还是低点？
5. 什么是支撑/阻力的角色互换？

### 进阶题（至少对4/5）
1. 看涨吞没和刺透形态的区别是什么？哪个信号更强？
2. 十字星出现在下跌趋势末端和横盘中间，含义有什么不同？
3. 为什么有效突破需要收盘价确认，盘中突破不算？
4. 有人说"下影线越长支撑越强"，这句话有什么问题？
5. 三白兵的"停滞"形态是什么样的？意味着什么？

### 实战题
打开任意A股日K线图，完成以下操作：
1. 画出当前的趋势线
2. 标出最近的支撑位和阻力位
3. 识别最近5个交易日内出现的K线形态
4. 判断当前应该持有、买入还是卖出（说出理由）

---

**答案要点**：

基础题：
1. 多方曾大幅上攻但被空方打回，上涨受阻，可能是顶部信号
2. 位置不同——锤子线在下跌末端（底部反转），悬挂人在上涨末端（顶部反转）
3. 大阴线 + 十字星/小实体 + 大阳线
4. 低点
5. 被突破的阻力变成支撑，被跌破的支撑变成阻力

进阶题：
1. 看涨吞没：阳线实体完全包住阴线实体（更强）；刺透：阳线收盘超过阴线实体中点（较弱）
2. 下跌末端=变盘预警（可能反转）；横盘中间=无意义
3. 盘中可能是假突破/诱多诱空，收盘站上/下才说明多空力量真正逆转
4. 下影线只说明当天空方被打回，不等于明天还有支撑。需要后续K线确认
5. 第三根阳线实体缩小、上影线加长 = 上涨动能衰竭，可能见顶

---

*阶段一学完大约需要2-3周。重点是大量看图练习，形成对K线的"直觉"。下一阶段我们将学习如何把K线组合成更大的"图表形态"。*
