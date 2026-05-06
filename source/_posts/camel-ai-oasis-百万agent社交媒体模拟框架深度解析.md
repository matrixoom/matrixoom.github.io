---
title: "OASIS：百万 LLM Agent 的社交媒体模拟框架深度解析"
date: 2025-05-15 10:00:00
tags:
  - AI Agent
  - LLM
  - 开源项目
  - 社会仿真
  - Python
categories:
  - AI技术
cover: false
description: 深度解析 camel-ai/oasis 开源项目：一个支持百万级 LLM Agent 并发运行的社交媒体仿真平台，涵盖系统架构、推荐算法、异步通信机制及代码实现细节。
---

如果你对「用 AI Agent 模拟整个 Twitter 生态」这件事感兴趣，`camel-ai/oasis` 是目前最值得深挖的开源项目之一。

这篇文章记录我完整阅读其源码后的理解——不只是"它能做什么"，更重要的是"它怎么做到的"。

---

## 项目背景

OASIS（**O**pen **A**gent **S**ocial **I**nteraction **S**imulations）来自 CAMEL-AI 团队，配套论文 [arXiv:2411.11581](https://arxiv.org/abs/2411.11581)。

核心野心：**用 LLM Agent 模拟真实社交网络的涌现行为**。

不是一两个 Agent 对话，而是：

- 支持 **100 万** Agent 并发运行
- 模拟 Twitter / Reddit 两种平台机制
- 每个 Agent 有完整的社交图谱（关注/被关注/屏蔽）
- 内置四种推荐算法（含 TwHIN-BERT 个性化推荐）
- 支持信息传播、谣言扩散、意见极化等社会动力学研究

研究场景举例：

> 向 1000 个具有不同政治倾向 Agent 的网络注入一条假新闻，24 小时后谣言覆盖率是多少？哪些节点是超级传播者？

这类问题在真实平台上无法实验，但在 OASIS 里可以做 A/B 测试。

---

## 系统架构全景

OASIS 的整体架构分为五层：

```
┌─────────────────────────────────────────────────────────┐
│                     用户/研究员                           │
│              oasis.make() / OasisEnv                     │
└────────────────────────┬────────────────────────────────┘
                         │  step(actions)
┌────────────────────────▼────────────────────────────────┐
│                    OasisEnv（环境层）                     │
│         reset / step / close  (PettingZoo 风格)          │
└──────┬──────────────────────────┬───────────────────────┘
       │                          │
┌──────▼──────┐          ┌────────▼────────┐
│  AgentGraph  │          │    Platform      │
│  社交关系图   │          │  平台核心逻辑    │
│  igraph/Neo4j│          │  SQLite 数据库   │
└──────┬──────┘          └────────┬────────┘
       │                          │
┌──────▼──────────────────────────▼───────────────────────┐
│                   Channel（异步通信层）                    │
│           asyncio.Queue + AsyncSafeDict                  │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                  SocialAgent（Agent 层）                  │
│        ChatAgent + perform_action_by_llm                 │
│        LLM 决策 / 个人画像 / 推荐类型                     │
└─────────────────────────────────────────────────────────┘
```

五个核心组件，每个职责清晰，下面逐一深入。

---

## 核心组件一：SocialAgent

`SocialAgent` 继承自 CAMEL 框架的 `ChatAgent`，是整个系统的"原子单元"。

### 初始化参数

```python
agent_alice = SocialAgent(
    agent_id=0,
    user_info=UserInfo(
        user_name="alice",
        name="Alice",
        description="A tech enthusiast and a fan of OASIS",
        profile=None,
        recsys_type="reddit",   # 推荐算法类型：reddit / twhin-bert / random
    ),
    agent_graph=agent_graph,
    model=openai_model,
    available_actions=[
        ActionType.LIKE_POST,
        ActionType.CREATE_POST,
        ActionType.CREATE_COMMENT,
        ActionType.FOLLOW,
    ],
)
```

`recsys_type` 在这里决定了这个 Agent 将使用哪种推荐算法获取内容——不同 Agent 可以使用不同的推荐策略，模拟平台算法的差异化效果。

### LLM 驱动的行为决策

`perform_action_by_llm` 是 Agent 行为的核心方法。执行流程：

1. 从平台拉取推荐帖子（`ActionType.REFRESH`）
2. 将帖子内容 + Agent 个人信息构造 Prompt
3. 调用 LLM 决策：执行哪个动作，参数是什么
4. 解析 LLM 输出，执行对应 ActionType

```python
async def perform_action_by_llm(self):
    # 1. 刷新推荐内容
    await self.env.step({self: [ManualAction(ActionType.REFRESH)]})

    # 2. 构造 prompt（包含个人画像 + 推荐帖子列表）
    prompt = self._build_action_prompt(rec_posts)

    # 3. LLM 决策
    response = await self.step(HumanMessage(prompt))

    # 4. 解析并执行
    action = self._parse_action(response)
    await self.env.step({self: [action]})
```

LLM 会输出类似 `{"action": "create_post", "args": {"content": "..."}}` 的结构化响应，框架负责解析执行。

---

## 核心组件二：Platform（平台层）

`Platform` 是整个模拟的"服务端"，所有 Agent 的动作请求都汇聚到这里处理。

### 数据持久化：SQLite

平台使用 SQLite 存储所有状态：

| 表名 | 内容 |
|------|------|
| `user` | 用户信息（用户名、简介、关注数等） |
| `post` | 帖子内容（作者、内容、点赞数、时间戳） |
| `comment` | 评论（帖子ID、作者、内容） |
| `follow` | 关注关系（follower_id → followee_id） |
| `rec_table` | 推荐表（每个用户的待推内容列表） |

SQLite 的选择是务实的：单文件、零依赖、Python 内置支持，对模拟场景足够。

### 动作处理

`Platform` 实现了 30+ 种动作的处理逻辑，每个对应一个异步方法：

```python
async def _create_post(self, user_id, content) -> dict:
    post_id = await self.db.execute_insert(
        "INSERT INTO post (user_id, content, created_at) VALUES (?,?,?)",
        (user_id, content, self.clock.get_time())
    )
    # 触发推荐系统更新
    await self._update_rec_table_for_followers(user_id, post_id)
    return {"success": True, "post_id": post_id}

async def _like_post(self, user_id, post_id) -> dict:
    await self.db.execute(
        "UPDATE post SET like_count = like_count + 1 WHERE post_id = ?",
        (post_id,)
    )
    return {"success": True}
```

---

## 核心组件三：Channel（异步通信）

这是 OASIS 能支持百万 Agent 并发的技术基础。

### 设计思路

Agent 和 Platform 之间不直接调用，而是通过 **消息队列** 通信：

```
Agent ──► Channel.send_msg() ──► asyncio.Queue (Platform 接收)
Agent ◄── Channel.receive_msg() ◄── AsyncSafeDict (Platform 发送)
```

```python
class Channel:
    def __init__(self):
        self.receive_queue = asyncio.Queue()        # Platform 接收消息
        self.send_dict = AsyncSafeDict()            # Platform 发回响应

    async def send_msg(self, msg: Message, msg_id: str):
        """Agent 发送动作请求"""
        await self.receive_queue.put((msg, msg_id))

    async def receive_msg(self, msg_id: str):
        """Agent 等待 Platform 响应"""
        while msg_id not in self.send_dict:
            await asyncio.sleep(0.01)
        return await self.send_dict.pop(msg_id)
```

`AsyncSafeDict` 是基于 `asyncio.Lock` 实现的线程安全字典，保证并发读写安全。

每条消息携带 UUID 作为消息 ID，Agent 发送后轮询等待同 ID 的响应，实现请求-响应的异步配对。

### 为什么不用 HTTP/RPC？

纯 asyncio 内存队列避免了序列化/反序列化和网络 I/O 开销，在单机模拟场景下性能远优于网络通信。

---

## 核心组件四：AgentGraph（社交关系图）

`AgentGraph` 管理所有 Agent 之间的社交关系，支持两种后端：

### igraph（默认，轻量级）

适合中小规模模拟（< 10 万 Agent）：

```python
class AgentGraph:
    def __init__(self, graph_type="igraph"):
        if graph_type == "igraph":
            import igraph as ig
            self.graph = ig.Graph(directed=True)

    def add_agent(self, agent: SocialAgent):
        self.graph.add_vertex(name=str(agent.agent_id))

    def add_edge(self, agent1: SocialAgent, agent2: SocialAgent):
        """添加关注关系：agent1 关注 agent2"""
        self.graph.add_edge(
            str(agent1.agent_id),
            str(agent2.agent_id)
        )

    def get_neighbors(self, agent: SocialAgent) -> list:
        """获取 agent 的关注列表"""
        return self.graph.neighbors(str(agent.agent_id), mode="out")
```

### Neo4j（可选，大规模）

当 Agent 数量超过 10 万时，igraph 的内存消耗和查询性能会成瓶颈，可切换 Neo4j：

```python
class Neo4jHandler:
    def __init__(self, uri, user, password):
        from neo4j import GraphDatabase
        self.driver = GraphDatabase.driver(uri, auth=(user, password))

    def add_follow_relationship(self, follower_id, followee_id):
        with self.driver.session() as session:
            session.run(
                "MERGE (a:Agent {id: $fid}) "
                "MERGE (b:Agent {id: $tid}) "
                "MERGE (a)-[:FOLLOWS]->(b)",
                fid=follower_id, tid=followee_id
            )
```

图结构支持高效的社交关系查询，比如"获取某 Agent 的 2 度朋友"、"计算图的聚类系数"等，这些在研究信息传播路径时非常有用。

---

## 核心组件五：推荐系统

推荐算法决定了每个 Agent 在刷新时能看到什么内容，直接影响信息传播的模式。OASIS 实现了四种：

### 1. Random（随机推荐）

基准对照组，从全量帖子中随机采样：

```python
def random_recsys(user_id, post_ids, k=10):
    return random.sample(post_ids, min(k, len(post_ids)))
```

### 2. Reddit 热度算法

模仿真实 Reddit 的排序公式：

```python
def reddit_score(ups, downs, date):
    score = ups - downs
    order = log(max(abs(score), 1), 10)
    sign = 1 if score > 0 else (-1 if score < 0 else 0)
    # Reddit epoch: 2005-12-08 07:46:43
    seconds = (date - datetime(1970, 1, 1)).total_seconds() - 1134028003
    return round(sign * order + seconds / 45000, 7)
```

这个公式的精妙之处：时间衰减（`seconds / 45000`）确保新帖子有机会超越旧的热门帖，同时点赞分（`sign * order`）用对数压缩，避免百万赞帖子永远霸榜。

### 3. Personalized（MiniLM 个性化）

使用 `sentence-transformers/all-MiniLM-L6-v2` 计算用户画像与帖子内容的语义相似度：

```python
def personalized_recsys(user_id, posts):
    model = SentenceTransformer('all-MiniLM-L6-v2')

    # 用户画像向量（历史帖子内容的平均嵌入）
    user_vector = model.encode(user_previous_posts[user_id])

    # 候选帖子向量
    post_vectors = model.encode([p['content'] for p in posts])

    # 余弦相似度排序
    similarities = cosine_similarity([user_vector], post_vectors)[0]
    return sorted(posts, key=lambda i: similarities[i], reverse=True)
```

### 4. TwHIN-BERT（Twitter 个性化）

专为 Twitter 场景设计，使用 Twitter 官方开源的 `Twitter/twhin-bert-base` 模型，在 Twitter 数据上预训练，对推文语义理解更准确：

```python
def twhin_recsys(user_id, posts, device):
    tokenizer = get_twhin_tokenizer()
    model = get_twhin_model(device)

    # Lazy loading：首次调用时从 HuggingFace 下载模型
    inputs = tokenizer(texts, return_tensors="pt", padding=True, truncation=True)
    with torch.no_grad():
        embeddings = model(**inputs).last_hidden_state[:, 0, :]  # [CLS] token

    # 与用户历史推文嵌入计算相似度
    scores = cosine_similarity(user_embedding, embeddings.numpy())
    return top_k_posts(posts, scores)
```

---

## Clock：模拟时间管理

`Clock` 类支持时间加速因子 `k`，允许在现实 1 秒内模拟 `k` 秒的社交平台时间：

```python
class Clock:
    def __init__(self, k: float = 1.0):
        self._k = k                    # 时间加速比
        self._start_real = time.time()
        self._start_sim = 0.0

    def get_time(self) -> float:
        """返回当前模拟时间（秒）"""
        elapsed_real = time.time() - self._start_real
        return self._start_sim + elapsed_real * self._k
```

设置 `k=3600` 时，模拟 1 秒 = 现实 1 小时，可以快速推演"24小时内谣言传播曲线"。

---

## 完整使用示例

下面是一个最简可运行示例，演示整个流程：

```python
import asyncio
import os
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType
import oasis
from oasis import ActionType, AgentGraph, LLMAction, ManualAction, SocialAgent, UserInfo

async def main():
    # 1. 创建 LLM 模型
    model = ModelFactory.create(
        model_platform=ModelPlatformType.OPENAI,
        model_type=ModelType.GPT_4O_MINI,
    )

    # 2. 定义可用动作集
    actions = [
        ActionType.LIKE_POST,
        ActionType.CREATE_POST,
        ActionType.CREATE_COMMENT,
        ActionType.FOLLOW,
    ]

    # 3. 初始化社交图谱
    agent_graph = AgentGraph()

    # 4. 创建 Agent（每个 Agent 有独立画像）
    alice = SocialAgent(
        agent_id=0,
        user_info=UserInfo(
            user_name="alice",
            name="Alice",
            description="科技爱好者，关注 AI 动态",
            recsys_type="reddit",
        ),
        agent_graph=agent_graph,
        model=model,
        available_actions=actions,
    )
    agent_graph.add_agent(alice)

    bob = SocialAgent(
        agent_id=1,
        user_info=UserInfo(
            user_name="bob",
            name="Bob",
            description="社交媒体研究员",
            recsys_type="reddit",
        ),
        agent_graph=agent_graph,
        model=model,
        available_actions=actions,
    )
    agent_graph.add_agent(bob)

    # 5. 创建环境（Reddit 模式）
    env = oasis.make(
        agent_graph=agent_graph,
        platform=oasis.DefaultPlatformType.REDDIT,
        database_path="./simulation.db",
    )
    await env.reset()

    # 6. 手动注入初始内容（种子帖子）
    await env.step({
        alice: [ManualAction(
            action_type=ActionType.CREATE_POST,
            action_args={"content": "大家怎么看 GPT-5 发布这件事？"}
        )]
    })

    # 7. 让所有 Agent 自主决策（LLM 驱动）
    await env.step({
        agent: LLMAction()
        for _, agent in env.agent_graph.get_agents()
    })

    await env.close()
    print("模拟完成，结果存储在 simulation.db")

asyncio.run(main())
```

---

## 研究用例：信息传播实验

OASIS 的设计明显面向社会科学研究，`data/` 目录包含真实 Twitter 和 Reddit 数据集用于初始化网络：

```
data/
├── twitter/
│   ├── user_all_id_time.json       # 真实用户关系图
│   └── post_all_id_time.json       # 历史推文
└── reddit/
    ├── user_info.json
    └── post_info.json
```

典型研究工作流：

```
真实社交网络数据  ──►  初始化 AgentGraph
                      + 注入历史帖子
                              │
                    设置实验条件（种子节点/初始内容）
                              │
                     运行 N 轮 env.step()
                              │
                  从 SQLite 分析传播轨迹
                    计算覆盖率/传播速度/节点影响力
```

---

## 工程亮点总结

| 特性 | 实现方式 | 优势 |
|------|---------|------|
| 高并发 | asyncio + Channel 异步消息队列 | 单机支持百万 Agent |
| 持久化 | SQLite 单文件数据库 | 零依赖、实验可复现 |
| 图扩展 | igraph / Neo4j 双后端 | 小规模轻量、大规模可扩 |
| 推荐算法 | 4 种可插拔推荐策略 | 模拟不同平台算法效果 |
| 时间模拟 | Clock + 加速因子 k | 快速推演长时序动力学 |
| LLM 兼容 | CAMEL ModelFactory | 支持 OpenAI / Anthropic / 开源模型 |

---

## 局限性与值得关注的问题

读完代码，有几点值得注意：

**1. Token 成本不低**

官方提供了参考数据：

| 规模 | 轮次 | 预估消耗 |
|------|------|---------|
| 10 Agent | 10 轮 | ~$0.01 |
| 100 Agent | 10 轮 | ~$0.1 |
| 1000 Agent | 10 轮 | ~$1 |
| 百万 Agent | 1 轮 | ~$1000+ |

百万 Agent 主要靠开源小模型（Llama/Qwen 本地部署）降成本，不能直接用 GPT-4o 跑。

**2. TwHIN-BERT 首次运行需下载 ~400MB 模型**

建议提前执行：

```bash
python -c "from transformers import AutoModel; AutoModel.from_pretrained('Twitter/twhin-bert-base')"
```

**3. 用户行为真实性依赖 Prompt 设计**

Agent 画像的 `description` 字段直接影响 LLM 行为倾向，复杂的政治/情感偏向建模需要精细的 Prompt 工程。

**4. 单机性能瓶颈**

asyncio 是单线程事件循环，CPU 密集型操作（如大量 BERT 推理）会阻塞事件循环。大规模实验建议分布式部署或使用 ProcessPoolExecutor 卸载 CPU 任务。

---

## 总结

OASIS 是一个设计清晰、工程成熟的社会仿真框架。它不是学术玩具，而是认真解决了"如何让百万 LLM Agent 同时在一个虚拟社交平台上运行"这个工程问题。

对研究者来说，它提供了一个可控实验室：真实算法（Reddit/TwHIN-BERT 推荐）、真实数据集初始化、可复现的数据库快照。

对开发者来说，Channel 的异步通信设计、AgentGraph 的双后端抽象、推荐系统的可插拔架构，都是值得借鉴的设计模式。

代码仓库：[https://github.com/camel-ai/oasis](https://github.com/camel-ai/oasis)

论文：[arXiv:2411.11581](https://arxiv.org/abs/2411.11581)
