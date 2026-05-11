---
title: "AI-Agent-Stock 源码深度分析：多AI Agent盯盘系统"
date: 2026-05-12 08:00:00
cover: /images/aiagents-stock-cover.png
sticky: 0
comments: true
categories:
  - 技术
tags:
  - AI Agent
  - 量化交易
  - A股
  - 开源项目
  - 源码分析
---

# AI-Agent-Stock 源码深度分析：多AI Agent盯盘系统

> 关键词：量化交易、A股、LLM Agent、开源项目、Python、Streamlit

---

## 一、项目概览

[AI-Agent-Stock](https://github.com/oficcejo/aiagents-stock) 是一个基于 **DeepSeek LLM 的多AI智能体股票团队分析系统**，核心设计理念是：**模拟证券公司分析师团队，从多个维度对股票进行全方位分析，最终给出投资建议**。

系统支持 A股、港股、美股三大市场，集成了智能选股、板块策略、龙虎榜分析、AI盯盘、实时监测等完整工作流。

| 指标 | 数据 |
|------|------|
| 🌟 GitHub | https://github.com/oficcejo/aiagents-stock |
| 主要语言 | Python |
| 开源协议 | MIT |
| 前端框架 | Streamlit |
| AI模型 | DeepSeek（支持任意 OpenAI 兼容模型）|
| 数据源 | AKShare + yfinance + 问财(pywencai) |
| 部署方式 | 本地部署 / Docker 一键部署 |

项目最引人注目的三个数字：**6位AI分析师 × 4大选股策略 × 全流程自动化**。

---

## 二、系统架构总览

### 2.1 整体架构

```
AI-Agent-Stock/
├── app.py                             # Streamlit 主界面
├── ai_agents.py                       # 多AI智能体核心
├── deepseek_client.py                 # DeepSeek API 客户端
├── stock_data.py                     # 股票数据获取（三市场）
│
├── sector_strategy_agents.py          # 智策板块AI智能体（4位）
├── sector_strategy_data.py           # 板块数据获取
├── sector_strategy_engine.py         # 板块分析引擎
├── sector_strategy_ui.py            # 板块分析UI
│
├── longhubang_agents.py            # 智瞰龙虎AI智能体（5位）
├── longhubang_data.py              # 龙虎榜数据获取
├── longhubang_engine.py            # 龙虎榜分析引擎
│
├── main_force_selector.py          # 主力选股模块
├── low_price_bull_ui.py           # 低价擒牛策略
├── small_cap_ui.py                # 小市值策略
├── value_stock_ui.py              # 低估值策略
│
├── smart_monitor_ui.py            # AI盯盘模块
├── monitor_manager.py              # 实时监测管理
├── notification_service.py         # 通知服务（邮件/Webhook）
│
└── data_source_manager.py         # 数据源管理器（TDX/AKShare/Tushare）
```

### 2.2 分析流程 Pipeline

系统采用经典的 **多智能体协同 + 团队讨论 + 最终决策** 三步流程：

```
用户输入股票代码
      ↓
获取股票数据（三市场自适应）
      ↓
┌─────────────────────────────────────┐
│  并行运行多位AI分析师（可配置）          │
│  • 技术面分析师                     │
│  • 基本面分析师                    │
│  • 资金面分析师                    │
│  • 风险管理师                     │
│  • 市场情绪分析师（可选）          │
│  • 新闻分析师（可选）             │
└─────────────────────────────────────┘
      ↓
   团队讨论（综合研判）
      ↓
   最终投资决策（JSON结构化输出）
      ↓
   PDF报告 / 实时监测 / 数据库保存
```

---

## 三、6位AI分析师智能体设计

这是项目最核心的部分。`ai_agents.py` 定义了6位专业分析师，每位都有明确的职责和关注领域。

### 3.1 分析师团队架构

| 分析师 | 职责 | 关注领域 | Prompt设计重点 |
|----------|------|----------|----------------|
| 🔍 技术分析师 | 技术指标分析、图表形态识别、趋势判断 | MA均线、RSI、MACD、布林带、KDJ、量比 | 7维度深度分析 |
| 💼 基本面分析师 | 财务分析、行业研究、估值分析 | ROE/ROA、毛利率、资产负债率、季报趋势 | 季报8期趋势分析 |
| 💰 资金面分析师 | 资金流向、主力行为、市场情绪 | 主力/超大单/大单/中单/小单 | 20日资金流向趋势 |
| ⚠️ 风险管理师 | 风险识别、风险评估、控制策略 | 限售解禁、大股东减持、重要事件 | 10维度风险量化 |
| 📈 市场情绪分析师 | 情绪指标、投资者心理、热点追踪 | AR/BR、换手率、涨跌停统计 | 情绪量化评分 |
| 📰 新闻分析师 | 新闻事件分析、舆情研究 | 最新新闻、重要事件影响评估 | 事件驱动分析 |

### 3.2 技术分析师实现（示例）

```python
# ai_agents.py 片段
def technical_analyst_agent(self, stock_info: Dict, stock_data: Any, indicators: Dict):
    """技术面分析智能体"""
    prompt = f"""
你是一名资深的技术分析师。请基于以下股票数据进行专业的技术面分析：

股票信息：
- 股票代码：{stock_info.get('symbol')}
- 当前价格：{stock_info.get('current_price')}

最新技术指标：
- RSI：{indicators.get('rsi')}
- MA5/MA20/MA60：{indicators.get('ma5')}, {indicators.get('ma20')}
- MACD：{indicators.get('macd')}
- 布林带：{indicators.get('bb_upper')} / {indicators.get('bb_lower')}

请从以下角度进行分析：
1. 趋势分析（均线系统、价格走势）
2. 超买超卖分析（RSI、KDJ）
3. 动量分析（MACD）
4. 支撑阻力分析（布林带）
5. 成交量分析
6. 短期、中期、长期技术判断
7. 关键技术位分析
"""
    messages = [
        {"role": "system", "content": "你是一名经验丰富的股票技术分析师..."},
        {"role": "user", "content": prompt}
    ]
    return self.deepseek_client.call_api(messages)
```

### 3.3 团队讨论机制

这是项目非常巧妙的设计——**让AI智能体进行"讨论"**，模拟真实投资决策会议：

```python
def conduct_team_discussion(self, agents_results: Dict, stock_info: Dict):
    """进行团队讨论"""
    # 收集所有分析师报告
    reports = []
    for agent_name, result in agents_results.items():
        reports.append(f"【{result['agent_name']}报告】\n{result['analysis']}")
    
    discussion_prompt = f"""
现在进行投资决策团队会议，参会人员包括：{participants}

请模拟一场真实的投资决策会议讨论：
1. 各分析师观点的一致性和分歧
2. 不同维度分析的权重考量
3. 风险收益评估
4. 投资时机判断
5. 策略制定思路
6. 达成初步共识

请以对话形式展现讨论过程，体现专业团队的思辨过程。
"""
    # 调用LLM生成讨论记录
    discussion_result = self.deepseek_client.call_api(messages)
    return discussion_result
```

讨论结果会作为最终决策的重要依据，**这让AI分析从"单点判断"升级为"群体智慧"**。

---

## 四、数据源架构与三层Fallback机制

### 4.1 三市场数据适配

`stock_data.py` 实现了 **A股/港股/美股** 三个市场的统一数据接口：

```python
class StockDataFetcher:
    
    def get_stock_info(self, symbol):
        if self._is_chinese_stock(symbol):
            return self._get_chinese_stock_info(symbol)   # A股
        elif self._is_hk_stock(symbol):
            return self._get_hk_stock_info(symbol)     # 港股
        else:
            return self._get_us_stock_info(symbol)      # 美股
```

**A股数据源三层Fallback**：

| 层级 | 数据源 | 用途 |
|------|--------|------|
| 主源 | AKShare | 实时行情、财务数据、资金流向 |
| 备用 | Tushare | 财务数据降级源（需积分）|
| 兜底 | yfinance | 美股/港股数据 |

### 4.2 数据 Source Manager（`data_source_manager.py`）

项目专门设计了 `DataSourceManager` 类来实现智能数据源切换：

```python
class DataSourceManager:
    def get_stock_hist_data(self, symbol, start_date, end_date):
        # 优先使用TDX本地数据源（响应<50ms）
        if self.tdx_available:
            try:
                return self._from_tdx(symbol, start_date, end_date)
            except:
                pass
        
        # 降级到Tushare
        if self.tushare_available:
            try:
                    return self._from_tushare(symbol, start_date, end_date)
            except:
                pass
        
        # 最后降级到AKShare
        return self._from_akshare(symbol, start_date, end_date)
```

---

## 五、智策板块分析系统

这是项目的一个亮点功能——**4位AI智能体协同分析板块轮动**。

### 5.1 四位板块分析师

| 分析师 | 职责 | 数据来源 |
|----------|------|----------|
| 🌐 宏观策略师 | 经济政策、新闻事件、市场情绪 | 东方财富新闻（150条）|
| 📊 板块诊断师 | 板块走势、估值、轮动 | AKShare行业/概念板块行情 |
| 💰 资金流向分析师 | 主力资金、北向资金 | AKShare资金流向数据 |
| 📈 市场情绪解码员 | 情绪量化、热点识别 | 涨跌家数、涨跌停统计 |

### 5.2 板块分析引擎

```python
# sector_strategy_engine.py 片段
def run_sector_analysis():
    # 1. 并行获取四类数据
    market_data = get_market_data()          # 大盘指数、涨跌统计
    sectors_data = get_sectors_data()        # 行业板块行情
    concepts_data = get_concepts_data()      # 概念板块行情
    fund_flow_data = get_fund_flow_data()   # 板块资金流向
    north_flow_data = get_north_flow_data() # 北向资金
    news_data = get_financial_news()        # 财经新闻
    
    # 2. 并行运行4位AI分析师
    agents = SectorStrategyAgents()
    results = {
        'macro': agents.macro_strategist_agent(market_data, news_data),
        'sector': agents.sector_diagnostician_agent(sectors_data, concepts_data),
        'fund_flow': agents.fund_flow_analyst_agent(fund_flow_data, north_flow_data),
        'sentiment': agents.market_sentiment_decoder_agent(market_data, sectors_data)
    }
    
    # 3. 综合研判
    final_result = synthesize_results(results)
    return final_result
```

---

## 六、智瞰龙虎榜分析

这是另一个亮点——**基于龙虎榜数据的游资行为分析**。

### 6.1 五位龙虎榜分析师

| 分析师 | 职责 | 核心关注 |
|----------|------|----------|
| 🎯 游资行为分析师 | 识别活跃游资及操作风格 | 赵老哥、章盟主、炒股养家 |
| 📈 个股潜力分析师 | 挖掘次日上涨概率高的股票 | 核心推荐逻辑 |
| 🔥 题材追踪分析师 | 热点题材和炒作周期 | 萌芽期/爆发期/衰退期 |
| ⚠️ 风险控制专家 | 高风险股票和资金陷阱 | 游资出货信号 |
| 👔 首席策略师 | 综合研判，给出最终推荐 | 推荐股票清单 + 确定性评级 |

### 6.2 龙虎榜数据源

```python
# longhubang_data.py 片段
def get_longhubang_data(date):
    """获取龙虎榜数据（StockAPI接口）"""
    # 调用StockAPI（免费1000次/天）
    api_url = f"http://api.stockapi.com/longhubang?date={date}"
    response = requests.get(api_url)
    
    # 解析龙虎榜数据
    # - 上榜股票List
    # - 游资席位交易明细
    # - 个股涨跌幅、换手率
    # - 题材概念标签
    return parsed_data
```

---

## 七、主力选股与策略系统

### 7.1 四种选股策略对比

| 策略 | 筛选条件 | 适用场景 | AI分析深度 |
|------|----------|----------|-------------|
| 💰 主力选股 | 主力资金净流入TOP100 + 市值/涨跌幅筛选 | 跟随主力资金 | 3位分析师精选3-5只 |
| 🐂 低价擒牛 | 股价<10元 + 净利润增长≥100% | 小盘高成长 | 基本面 + 技术面 |
| 📊 小市值策略 | 总市值≤50亿 + 营收/净利润增长 | 小盘成长股 | 成长能力评估 |
| 📈 净利增长 | 净利润增长≥10% + 营收增长 | 稳健成长股 | 财务趋势分析 |
| 💎 低估值策略 | PE≤20 + PB≤1.5 + 股息率≥1% | 价值投资 | 安全边际评估 |

### 7.2 主力选股实现（问财数据源）

```python
# main_force_selector.py 片段
def get_main_force_stocks(start_date, min_market_cap, max_market_cap):
    """获取主力资金净流入前100名股票"""
    # 使用问财（pywencai）获取主力资金排名
    query = f"{start_date}以来主力资金净流入排名，并计算区间涨跌幅，\
             市值{min_market_cap}-{max_market_cap}亿之间，非科创非st，\
             所属同花顺行业，总市值，净利润，营收，市盈率，市净率"
    
    result = pywencai.get(query=query, loop=True)
    
    # 返回前100名
    df = pd.DataFrame(result)
    return df.nlargest(100, '主力资金净流入')
```

---

## 八、AI盯盘与自动化交易

### 8.1 AI盯盘工作流程

```
启动盯盘任务
      ↓
每N秒检查一次目标股票
      ↓
获取实时行情 + 技术指标
      ↓
DeepSeek AI 决策（买入/持有/卖出）
      ↓
┌─────────────────────────────┐
│ 决策：买入                          │
│  • 检查是否T+1规则                │
│  • 检查持仓状态                   │
│  • 执行买入（MiniQMT）           │
└─────────────────────────────┘
      ↓
   通知用户（邮件/Webhook）
      ↓
   记录交易日志
```

### 8.2 MiniQMT集成

```python
# miniqmt_interface.py 片段
class MiniQMTInterface:
    def execute_trade(self, symbol, action, price, quantity):
        """执行交易指令"""
        if action == "买入":
            # 检查T+1规则
            if self._is_t_plus_one_violation(symbol):
                return {"success": False, "error": "T+1规则限制"}
            
            # 下买单
            order_id = self.client.order(
                symbol=symbol,
                direction="买入",
                price=price,
                quantity=quantity
            )
            return {"success": True, "order_id": order_id}
```

---

## 九、通知系统集成

项目支持 **邮件 + Webhook（钉钉/飞书）** 双通道通知。

### 9.1 通知配置

```python
# notification_service.py 片段
class NotificationService:
    def send_notification(self, title, message, channels=['email', 'webhook']):
        """发送通知"""
        if 'email' in channels and self.email_enabled:
            self._send_email(title, message)
        
        if 'webhook' in channels and self.webhook_enabled:
            self._send_webhook(title, message)
    
    def _send_webhook(self, title, message):
        """发送Webhook通知"""
        if self.webhook_type == 'dingtalk':
            # 钉钉Markdown格式
            payload = {
                "msgtype": "markdown",
                "markdown": {
                    "title": title,
                    "text": f"## {title}\n\n{message}"
                }
            }
        elif self.webhook_type == 'feishu':
            # 飞书卡片消息
            payload = {
                "msg_type": "interactive",
                "card": {...}
            }
        
        requests.post(self.webhook_url, json=payload)
```

---

## 十、项目亮点总结

### 10.1 技术创新点

1. **多智能体协同架构**：6位分析师 + 团队讨论 + 最终决策，模拟真实投资团队
2. **三市场适配**：A股/港股/美股一套代码，数据源自动切换
3. **四层数据源Fallback**：TDX → Tushare → AKShare → yfinance，保障数据可用性
4. **可配置分析师团队**：用户可选择哪些分析师参与（降低Token消耗）
5. **批量分析 + 并行处理**：ThreadPoolExecutor实现多股票并行分析
6. **完整交易闭环**：分析 → 监测 → 盯盘 → 交易 → 通知

### 10.2 与类似项目对比

| 维度 | AI-Agent-Stock | UZI-Skill |
|------|-------------------|------------|
| AI分析师数量 | 6位（可配置） | 51位（固定） |
| 分析深度 | 中等（适合实战） | 极深（22维数据） |
| 数据源 | AKShare + 问财 + yfinance | AKShare + 雪球 + 东财 |
| 前端界面 | Streamlit（Web UI） | CLI + HTML报告 |
| 自动化交易 | ✅ MiniQMT集成 | ❌ 无 |
| 实时监测 | ✅ 内置 | ❌ 无 |
| 部署难度 | 低（Docker一键） | 中（需要Claude Code） |

---

## 十一、部署与使用

### 11.1 Docker部署（推荐）

```bash
# 1. 克隆项目
git clone https://github.com/oficcejo/aiagents-stock.git
cd aiagents-stock

# 2. 配置环境变量
cp .env.example .env
# 编辑.env，填入DEEPSEEK_API_KEY

# 3. Docker启动
docker-compose up -d

# 4. 访问
浏览器打开 http://localhost:8503
```

### 11.2 本地部署

```bash
# 1. 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
.\venv\Scripts\Activate.ps1  # Windows

# 2. 安装依赖
pip install -r requirements.txt

# 3. 启动
streamlit run app.py
```

---

## 十二、综合评价

### 优点

1. **功能完整度高**：从数据获取到分析、从选股到交易、从监测到通知，覆盖量化交易全流程
2. **架构设计合理**：模块化程度高，每个功能都有独立的UI/Engine/Agents文件
3. **多市场支持**：A股/港股/美股一套代码适配，扩展性强
4. **用户友好**：Streamlit Web界面，无需命令行操作
5. **数据源健壮**：三层Fallback机制，适应不同网络环境
6. **可扩展性强**：支持任意OpenAI兼容模型（DeepSeek/Qwen/GPT-4o）

### 不足 / 注意点

1. **依赖LLM质量**：分析结果上限取决于底层LLM的推理能力（DeepSeek-V3效果较好）
2. **Token消耗较大**：完整6位分析师分析一次约消耗 8000-12000 tokens
3. **实时性有限**：AKShare数据有延迟（约1-5分钟）
4. **A股侧重**：虽然支持港股/美股，但问财、龙虎榜等功能仅限A股
5. **交易风险**：AI盯盘功能需谨慎使用，建议先模拟盘验证

---

## 十三、总结

AI-Agent-Stock 是一个在"AI + 量化 + 自动化交易"交叉点上做得相当完整的开源项目。它没有被做成黑盒SaaS，而是以 **本地部署 + 开源代码** 的形式交付，让用户完全掌控自己的数据和策略。

对于想要理解"如何让LLM做结构化投资决策"的开发者，这个项目的源码（尤其是 `ai_agents.py`、`sector_strategy_agents.py`、`longhubang_agents.py` 三个模块）非常值得精读。

**项目 GitHub**：https://github.com/oficcejo/aiagents-stock

*本文基于 2026-05-12 源码分析，如有偏差欢迎指正。*

---

## 附录：快速参考表

| 功能模块 | 核心文件 | 分析师数量 | 数据源 |
|----------|----------|-------------|--------|
| 单股分析 | ai_agents.py | 6位（可配） | AKShare + yfinance |
| 智策板块 | sector_strategy_agents.py | 4位 | AKShare + 东财新闻 |
| 智瞰龙虎 | longhubang_agents.py | 5位 | StockAPI + AKShare |
| 主力选股 | main_force_selector.py | 3位 | 问财（pywencai） |
| 低价擒牛 | low_price_bull_ui.py | 2位 | AKShare |
| AI盯盘 | smart_monitor_ui.py | 1位（DeepSeek）| MiniQMT + TDX |
