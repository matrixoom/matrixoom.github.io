---
title: AIHOT 技术架构与 AI 评分算法深度解析
date: 2026-05-08 17:00:00
tags:
  - AI
  - LLM
  - 内容聚合
  - 算法
  - 提示词工程
categories:
  - AI工程
---

> 当信息过载成为常态，AI 筛选不再是锦上添花，而是刚需。AIHOT 用一套完整的评分算法和提示词体系，把 20+ 渠道的海量信息，压缩成每天几十条高质量精选。

## 一、产品定位：AI 驱动的内容精选引擎

[AIHOT](https://aihot.virxact.com) 是一个面向 AI 行业的**精选内容聚合平台**，Slogan 很直白：「精选 — AI 自动挑选的高价值内容」。

### 核心功能

| 功能模块 | 说明 |
|---------|------|
| **AI 精选流** | 按时间倒序展示 AI 筛选后的内容，每条带「精选分数」(56-86 分) |
| **分类浏览** | 6 个标签：全部、大模型、产品、应用、研究、其他 |
| **AI 日报** | 每日汇总，按领域分类整理 |
| **公众号爆文** | 精选微信公众号优质文章 |
| **Agent API** | 提供 API 接入能力 |

### 内容来源矩阵

平台聚合了 20+ 优质信源：

- **官方渠道**：OpenAI Blog/Research、Anthropic Research、Google DeepMind、GitHub Blog
- **技术社区**：HuggingFace Papers/Daily Papers、Hacker News、Product Hunt
- **中文媒体**：IT之家、机器之心、量子位
- **社交媒体**：Twitter/X KOL（宝玉 @dotey、Rohan Paul、Dr. Katalin Karikó 等）
- **学术平台**：arXiv CS/AI 领域

---

## 二、技术架构全景

### 2.1 前端架构

```
┌─────────────────────────────────────────┐
│           Next.js 14+ (App Router)       │
│  ├─ SSR 服务端渲染                       │
│  ├─ 暗色主题优先 (data-theme="dark")     │
│  ├─ 分页加载 (?page=1, ?page=2)          │
│  └─ 图片代理 (/api/img-proxy?u=...)      │
└─────────────────────────────────────────┘
```

技术选型很务实：
- **Next.js App Router**：利用 SSR 提升首屏速度和 SEO
- **暗色主题优先**：符合开发者群体偏好
- **图片代理机制**：统一处理外部图片，解决跨域和防盗链问题

### 2.2 后端架构

```
┌─────────────────────────────────────────┐
│              数据采集层                   │
│  ├─ RSS 订阅 (IT之家、HN、The Decoder)   │
│  ├─ 官方 API (OpenAI、Anthropic、GitHub) │
│  ├─ Twitter/X API (KOL 监控)             │
│  └─ 学术平台 (HuggingFace、arXiv)        │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│              AI 处理管道                  │
│  ├─ 内容抓取 → 去重检测                   │
│  ├─ AI 摘要生成 (GPT-4o)                  │
│  ├─ 质量评分 (0-100)                      │
│  ├─ 标签提取 → 推荐理由生成               │
│  └─ 中文本地化 → 发布                     │
└──────────────────┬──────────────────────┘
                   ▼
┌─────────────────────────────────────────┐
│              数据存储层                   │
│  ├─ PostgreSQL (主数据库)                 │
│  └─ 内容缓存 + CDN                        │
└─────────────────────────────────────────┘
```

### 2.3 部署信息

从响应头可以推断：
- **Web 服务器**：nginx/1.18.0
- **Node.js 后端**：处理 SSR 和 API 请求
- **数据库**：PostgreSQL（从 ID 格式 `cm...` 推断，符合 CUID 特征）

---

## 三、核心数据结构

从网页源码中提取的 JSON 结构，揭示了整个内容处理流程：

```json
{
  "id": "cmb0...",
  "url": "https://openai.com/index/...",
  "title": "Original Title",
  "titleZh": "中文标题",
  "summaryZh": "中文摘要（AI生成）",
  "publishedAt": "2026-05-08T08:00:00.000Z",
  "aiSelected": true,
  "aiSelectedReason": "推荐理由（AI生成）",
  "qualityScore": 75,
  "finalScore": 82,
  "aiTags": ["大模型", "产品", "研究"],
  "source": {
    "name": "OpenAI Blog",
    "url": "https://openai.com/blog"
  },
  "duplicateCount": 3,
  "duplicateSources": ["Twitter", "Reddit", "HN"]
}
```

### 关键字段解读

| 字段 | 类型 | 说明 |
|-----|------|------|
| `qualityScore` | Integer | 原始质量评分 (0-100) |
| `finalScore` | Integer | 最终精选分数 (56-86，可见范围内) |
| `aiSelected` | Boolean | 是否被 AI 选中展示 |
| `aiSelectedReason` | String | AI 生成的推荐理由 |
| `aiTags` | Array | AI 提取的标签 |
| `duplicateCount` | Integer | 重复来源数量 |
| `duplicateSources` | Array | 重复来源列表 |

**注意**：`qualityScore` 和 `finalScore` 的差异暗示存在一个**评分调整机制**——可能是基于时效性、多样性或用户反馈的二次加权。

---

## 四、AI 评分算法深度解析

这是 AIHOT 的核心竞争力。从数据结构和产品表现推断，评分算法应该是**多维度加权模型**。

### 4.1 评分维度拆解

```
┌─────────────────────────────────────────────────────────┐
│                    最终分数 (56-86)                       │
│                      finalScore                         │
└──────────────────────────┬──────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │ 内容质量分  │  │ 时效性加权  │  │ 多样性调整  │
    │  (0-100)   │  │  (±10)     │  │  (±5)      │
    └────────────┘  └────────────┘  └────────────┘
```

### 4.2 内容质量评分模型

最可能的实现是 **LLM-based 评分**，Prompt 大致结构：

```python
CONTENT_SCORING_PROMPT = """
你是一位资深 AI 行业编辑，需要对以下文章进行质量评分。

【评分维度】
1. 信息价值 (25分): 是否包含独家信息、深度分析或重要公告
2. 专业深度 (25分): 技术细节是否充分，论证是否严谨
3. 时效性 (20分): 内容是否新颖，是否首发或早期报道
4. 可读性 (15分): 结构是否清晰，表达是否流畅
5. 影响力 (15分): 来源权威性，潜在传播价值

【评分标准】
- 90-100: 顶级内容，必推
- 80-89: 优质内容，强烈推荐
- 70-79: 良好内容，推荐
- 60-69: 一般内容，可选
- <60: 低质内容，过滤

【待评分内容】
标题: {title}
来源: {source}
发布时间: {published_at}
正文: {content[:3000]}

请输出 JSON 格式:
{{
  "quality_score": 整数(0-100),
  "dimension_scores": {{
    "information_value": 整数,
    "professional_depth": 整数,
    "timeliness": 整数,
    "readability": 整数,
    "influence": 整数
  }},
  "reasoning": "评分理由简述"
}}
"""
```

### 4.3 评分调整机制

从 `qualityScore` 到 `finalScore` 的转换，可能涉及以下调整：

```python
def calculate_final_score(quality_score, published_at, current_tags, all_today_tags):
    """
    计算最终精选分数
    """
    score = quality_score
    
    # 1. 时效性加权：越新的内容加分
    hours_old = (now - published_at).total_seconds() / 3600
    if hours_old < 6:
        score += 8
    elif hours_old < 24:
        score += 5
    elif hours_old < 72:
        score += 0
    else:
        score -= 5  # 旧内容降权
    
    # 2. 多样性调整：避免同一标签过度集中
    tag_overlap = len(set(current_tags) & set(all_today_tags)) / len(current_tags)
    if tag_overlap > 0.5:
        score -= 3  # 标签重复度太高，降权
    
    # 3. 来源权威性加成
    if source in PREMIUM_SOURCES:
        score += 3
    
    # 4. 去重惩罚
    if duplicate_count > 0:
        score -= min(duplicate_count * 2, 10)
    
    return clamp(score, 56, 86)  # 最终分数范围 56-86
```

### 4.4 评分范围的设计意图

为什么最终分数显示为 **56-86** 而不是 0-100？

| 设计考量 | 说明 |
|---------|------|
| **过滤低质内容** | <56 的内容直接被过滤，用户看不到 |
| **控制展示数量** | 只有 >56 的才进入精选流，保证质量底线 |
| **避免满分焦虑** | 86 封顶，留出"完美内容"的理论空间 |
| **心理锚定** | 56 是及格线，70+ 算优质，80+ 算精品 |

---

## 五、提示词工程详解

AIHOT 的提示词体系应该是**分层设计**，不同任务用不同 Prompt。

### 5.1 摘要生成 Prompt

```python
SUMMARY_PROMPT = """
你是一位专业的 AI 行业译者和内容编辑。请将以下英文文章翻译成中文，并生成简洁的摘要。

【任务要求】
1. 标题翻译：准确传达原意，符合中文表达习惯
2. 摘要生成：
   - 长度：100-150 字
   - 内容：概括核心观点 + 关键数据/结论
   - 风格：客观、专业、信息密度高
3. 标签提取：从预定义标签中选择最相关的 1-3 个

【预定义标签】
- 大模型：LLM、基础模型、训练技术
- 产品：AI 产品发布、功能更新
- 应用：行业应用、落地案例
- 研究：学术论文、技术突破
- 其他：政策、观点、杂项

【原文】
标题: {title}
正文: {content}

【输出格式】
{{
  "titleZh": "中文标题",
  "summaryZh": "中文摘要（100-150字）",
  "tags": ["标签1", "标签2"]
}}
"""
```

### 5.2 推荐理由生成 Prompt

```python
REASON_PROMPT = """
你是一位 AI 行业内容策展人。请为以下文章生成一句推荐理由。

【要求】
- 长度：20-40 字
- 角度：突出文章的独特价值或关键信息
- 风格：简洁有力，像朋友推荐好文

【示例】
- "GPT-4o 多模态能力实测，图像理解准确率提升 15%"
- "DeepMind 最新研究：小模型也能有大智慧"
- "OpenAI 安全团队负责人离职内幕，值得关注的信号"

【文章信息】
标题: {titleZh}
摘要: {summaryZh}
来源: {source}
评分: {qualityScore}

【输出】
只输出推荐理由，不要任何其他内容。
"""
```

### 5.3 日报生成 Prompt

```python
DAILY_DIGEST_PROMPT = """
你是一位 AI 行业日报编辑。请根据今天精选的文章，生成一份结构化日报。

【任务】
1. 按领域分类整理文章
2. 每个领域写一段综述（100-200 字）
3. 标出今日重点（Top 3）

【输入数据】
{articles_json}

【输出格式】
{{
  "date": "YYYY-MM-DD",
  "sections": [
    {{
      "name": "大模型动态",
      "summary": "领域综述...",
      "articles": [{{"title": "...", "url": "...", "highlight": true/false}}]
    }}
  ],
  "top3": [
    {{"rank": 1, "title": "...", "reason": "..."}}
  ]
}}
"""
```

---

## 六、技术亮点与可借鉴之处

### 6.1 工程层面的巧思

| 亮点 | 实现方式 | 价值 |
|-----|---------|------|
| **图片代理** | `/api/img-proxy?u=` | 解决跨域、防盗链、HTTPS 混合内容问题 |
| **双重转义 JSON** | HTML 中嵌入转义后的 JSON | SSR 时直接注入数据，减少 API 请求 |
| **暗色主题优先** | `data-theme="dark"` | 符合开发者审美，减少眼部疲劳 |
| **CUID 主键** | `cm...` 格式 ID | 分布式友好，可排序，防枚举 |

### 6.2 算法层面的设计

| 设计 | 说明 |
|-----|------|
| **多维度评分** | 不依赖单一指标，综合质量、时效、多样性 |
| **评分压缩** | 56-86 的显示范围，既过滤低质内容，又保留心理空间 |
| **去重追踪** | 记录 `duplicateCount` 和 `duplicateSources`，用于算法优化 |
| **标签体系** | 预定义标签 + AI 提取，平衡结构化与灵活性 |

### 6.3 产品层面的取舍

| 取舍 | 决策 | 理由 |
|-----|------|------|
| 不做用户评论 | 纯 AI 筛选 | 降低运营成本，保持中立性 |
| 不做个性化推荐 | 统一精选流 | 避免信息茧房，突出编辑价值 |
| 不做实时推送 | 定时更新 | 降低系统复杂度，保证内容质量 |
| 展示分数但不排序 | 时间倒序 | 分数是质量信号，时间是新鲜度信号 |

---

## 七、如何复刻一个类似系统

如果你想搭建一个类似的 AI 内容精选平台，以下是技术路线建议：

### 7.1 最小可行产品（MVP）

```
Week 1: 数据采集
- RSS 订阅：用 feedparser 抓取 5-10 个信源
- 存储：PostgreSQL + 简单表结构

Week 2: AI 处理管道
- 摘要：OpenAI GPT-4o-mini（成本低）
- 评分：GPT-4o + 结构化 Prompt
- 标签：简单关键词匹配或 LLM 提取

Week 3: 前端展示
- Next.js + Tailwind CSS
- 暗色主题 + 卡片式布局
- 分页加载

Week 4: 部署上线
- Vercel 或自托管
- 定时任务：GitHub Actions 或 cron
```

### 7.2 成本估算

| 项目 | 月成本（估算） | 说明 |
|-----|--------------|------|
| **服务器** | ¥100-300 | Vercel Hobby 免费，或阿里云/腾讯云 |
| **数据库** | ¥50-100 | PostgreSQL，轻量应用 |
| **LLM API** | ¥500-2000 | 取决于内容量，GPT-4o-mini 可大幅降低成本 |
| **域名** | ¥50-100/年 | 普通域名 |
| **总计** | ¥700-2500/月 | MVP 阶段可控制在 ¥1000 以内 |

### 7.3 核心代码框架

```python
# 内容处理器核心类
class ContentProcessor:
    def __init__(self):
        self.llm = OpenAI(api_key=OPENAI_API_KEY)
        self.db = PostgreSQLConnection()
    
    async def process_article(self, article: Article) -> ProcessedArticle:
        # 1. 去重检查
        if await self.is_duplicate(article.url):
            return None
        
        # 2. 获取全文
        content = await self.fetch_content(article.url)
        
        # 3. AI 处理（并行）
        summary_task = self.generate_summary(content)
        score_task = self.calculate_quality_score(content)
        tags_task = self.extract_tags(content)
        
        summary, score, tags = await asyncio.gather(
            summary_task, score_task, tags_task
        )
        
        # 4. 计算最终分数
        final_score = self.adjust_score(
            base_score=score,
            published_at=article.published_at,
            tags=tags
        )
        
        # 5. 生成推荐理由
        reason = await self.generate_reason(
            title=summary.title,
            summary=summary.text,
            score=final_score
        )
        
        # 6. 保存到数据库
        processed = ProcessedArticle(
            title_zh=summary.title,
            summary_zh=summary.text,
            quality_score=score,
            final_score=final_score,
            ai_tags=tags,
            ai_selected=final_score >= 56,
            ai_selected_reason=reason
        )
        
        await self.db.save(processed)
        return processed
    
    async def generate_summary(self, content: str) -> Summary:
        """生成中文摘要"""
        response = await self.llm.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": SUMMARY_PROMPT.format(content=content[:5000])
            }],
            response_format={"type": "json_object"}
        )
        return Summary.parse_raw(response.choices[0].message.content)
    
    async def calculate_quality_score(self, content: str) -> int:
        """计算质量分数"""
        response = await self.llm.chat.completions.create(
            model="gpt-4o",
            messages=[{
                "role": "user",
                "content": SCORING_PROMPT.format(content=content[:3000])
            }],
            response_format={"type": "json_object"}
        )
        result = json.loads(response.choices[0].message.content)
        return result["quality_score"]
    
    def adjust_score(self, base_score: int, published_at: datetime, tags: list) -> int:
        """调整最终分数"""
        score = base_score
        
        # 时效性加权
        hours = (datetime.now() - published_at).total_seconds() / 3600
        if hours < 6:
            score += 8
        elif hours < 24:
            score += 5
        elif hours > 72:
            score -= 5
        
        # 限制范围
        return max(56, min(86, score))
```

---

## 八、总结

AIHOT 是一个**工程与算法结合得相当漂亮**的内容聚合产品。它的核心优势不在于技术复杂度，而在于：

1. **精准的评分算法**：多维度加权 + 时效性调整 + 多样性控制，保证内容质量的同时避免同质化

2. **务实的提示词工程**：分层 Prompt 设计，不同任务用不同模型（摘要用轻量级，评分用强模型），控制成本

3. **克制的产品设计**：不做个性化、不做评论、不做实时推送——这些"减法"反而让产品定位更清晰

4. **工程细节到位**：图片代理、SSR、暗色主题、CUID 主键——每个细节都服务于用户体验

对于想搭建类似系统的开发者，最大的启示是：**AI 内容筛选的核心不是技术，而是标准**。你需要先定义什么是"好内容"，然后用算法和提示词把这个标准量化。AIHOT 的 56-86 评分体系，就是一个很好的参考范本。

---

**参考链接**
- AIHOT 官网：https://aihot.virxact.com
- 本文基于网页源码分析，部分内容为合理推测
