---
title: MiroFish深度源码分析：群体智能引擎的架构设计与实现原理
cover: /images/Abstract_digital_art_for_blog__2026-04-09T00-11-03.png
sticky: 0
comments: true
categories:
  - AI技术
  - 开源项目
tags:
  - 多智能体
  - 群体智能
  - OASIS
  - Zep
  - 社会仿真
  - 开源
date: 2026-04-09 08:00:00
---

# MiroFish深度源码分析：群体智能引擎的架构设计与实现原理

## 引言：用数字沙盘预演未来

如果你能提前看到一条政策的发布会引发怎样的舆论风暴，如果你能推演一本小说失传的结局会走向何方，如果你能在零风险中试错每一个"如果"——这就是 MiroFish 想要做的事。

**MiroFish** 是一款基于多智能体群体智能技术的 AI 预测引擎，由盛大集团战略孵化和支持，仿真引擎基于 CAMEL-AI 团队的开源项目 OASIS。它的核心理念是：从现实世界的种子信息出发，构建高保真数字平行世界，让成千上万个具备独立人格、记忆与行为逻辑的智能体在其中自由交互与社会演化，通过"上帝视角"推演未来走向。

本文将基于源码，系统剖析 MiroFish 的架构设计、核心工作流、关键技术实现与设计权衡。

> 项目地址：https://github.com/666ghj/MiroFish

---

## 一、技术栈全景

| 层级 | 技术 | 说明 |
|------|------|------|
| **后端** | Flask 3.0+ (Python ≥3.11) | Web API 服务 |
| **前端** | Vue 3 + Vite | SPA 用户界面 |
| **LLM** | OpenAI 兼容 API 格式 | 推荐阿里百炼 qwen-plus |
| **知识图谱** | Zep Cloud | 图谱构建、实体抽取、GraphRAG |
| **仿真引擎** | OASIS (camel-oasis + camel-ai) | 双平台社交媒体模拟 |
| **进程通信** | 基于文件系统的 IPC | Flask ↔ 仿真脚本 |
| **包管理** | uv (Python) + npm (Node) | 统一依赖管理 |
| **部署** | Docker / 源码 | docker-compose 一键部署 |

LLM 采用 OpenAI 兼容格式，这意味着你可以自由切换任何兼容的 LLM 提供商（OpenAI、阿里百炼、DeepSeek 等），而不被单一供应商锁定。

---

## 二、项目目录结构

```
MiroFish/
├── backend/                          # 后端 (Python Flask)
│   ├── pyproject.toml               # Python 依赖配置
│   ├── run.py                       # 启动入口
│   └── app/
│       ├── __init__.py              # Flask 应用工厂
│       ├── config.py                # 配置管理
│       ├── api/                     # API 路由层
│       │   ├── graph.py            # 图谱 API (本体生成、图谱构建)
│       │   ├── simulation.py       # 模拟 API (准备、运行、监控、采访)
│       │   └── report.py           # 报告 API (生成、查询、对话)
│       ├── services/                # 业务逻辑层（核心）
│       │   ├── ontology_generator.py      # 本体生成服务
│       │   ├── graph_builder.py           # 图谱构建服务 (Zep API)
│       │   ├── text_processor.py          # 文本处理与切块
│       │   ├── zep_entity_reader.py       # Zep 实体读取与过滤
│       │   ├── oasis_profile_generator.py # OASIS Agent 人设生成
│       │   ├── simulation_config_generator.py # 模拟配置生成
│       │   ├── simulation_manager.py      # 模拟生命周期管理
│       │   ├── simulation_runner.py       # 模拟进程管理
│       │   ├── simulation_ipc.py          # 进程间通信 (IPC)
│       │   ├── zep_graph_memory_updater.py # 动态图谱记忆更新
│       │   ├── zep_tools.py               # Zep 检索工具集
│       │   └── report_agent.py            # ReACT 模式报告 Agent
│       ├── models/                  # 数据模型层
│       │   ├── project.py          # 项目数据模型 (文件持久化)
│       │   └── task.py             # 任务管理 (线程安全单例)
│       └── utils/                   # 工具层
│           ├── llm_client.py       # LLM 客户端封装
│           ├── file_parser.py      # 文件解析 (PDF/TXT/MD)
│           ├── zep_paging.py       # Zep API 分页工具
│           ├── logger.py           # 日志管理
│           ├── retry.py            # 重试机制
│           └── locale.py           # 国际化
├── frontend/                         # 前端 (Vue 3)
│   └── src/
│       ├── views/                   # 页面视图
│       ├── components/              # 业务组件 (Step1~Step5)
│       ├── api/                     # API 调用层
│       ├── router/                  # 路由
│       ├── store/                   # 状态管理
│       └── i18n/                    # 国际化
├── scripts/
│   └── run_parallel_simulation.py   # OASIS 双平台并行模拟脚本
├── docker-compose.yml               # Docker 部署
└── .env.example                     # 环境变量模板
```

---

## 三、核心工作流：五步流水线

MiroFish 的核心工作流分为五个步骤，每一步职责明确、环环相扣：

### Step 1：图谱构建

**目标**：从种子材料中提取实体与关系，构建知识图谱

1. 用户上传文档（PDF/TXT/MD）+ 填写模拟需求
2. **OntologyGenerator** 调用 LLM 分析文本，生成本体定义：
   - 实体类型（entity_types）：人、组织、媒体、政府部门等
   - 关系类型（edge_types）：从属、关注、合作、对立等
   - 关键约束：实体必须是可在社交媒体上发声的真实主体，排除抽象概念
3. **TextProcessor** 将提取的文本切块（chunk_size=500, overlap=50）
4. **GraphBuilderService** 调用 Zep Cloud API：
   - 创建 Standalone Graph
   - 设置本体（Ontology）
   - 分批（batch_size=3）将文本块作为 Episode 导入
   - Zep 自动完成实体抽取、关系识别、GraphRAG 构建

本体生成的 System Prompt 非常值得一看——它明确要求 LLM 只生成"可以在社媒上发声的主体"，而非抽象概念。这是整个系统仿真真实性的基石。

### Step 2：环境搭建

**目标**：将知识图谱实体转化为 OASIS 仿真 Agent，配置模拟参数

1. **ZepEntityReader** 从 Zep Graph 读取所有实体节点和边，按类型过滤
2. **OasisProfileGenerator** 将实体转化为 OASIS Agent Profile：
   - 利用 LLM 为每个实体生成详细人设（bio, personality, posting_style）
   - 生成 Twitter/Reddit 双平台用户画像
3. **SimulationConfigGenerator** 通过 LLM 生成模拟配置：
   - 时间模拟参数（起始/结束时间、加速倍率）
   - Agent 活动配置（发帖频率、活跃时段）
   - 事件注入（突发新闻、热点话题）
   - 平台配置（Twitter/Reddit 动作集合）

### Step 3：开始模拟

**目标**：运行 OASIS 双平台并行社会仿真

1. **SimulationRunner** 以后台子进程方式启动 `run_parallel_simulation.py`
2. 同时运行 Twitter 和 Reddit 两个平台的仿真：
   - Twitter 动作集：CREATE_POST, LIKE_POST, REPOST, FOLLOW, QUOTE_POST, DO_NOTHING
   - Reddit 动作集：CREATE_POST, CREATE_COMMENT, LIKE/DISLIKE, SEARCH, TREND, FOLLOW, MUTE 等
3. 每个 Round，Agent 根据人设和环境信息自主决策行动
4. **ZepGraphMemoryUpdater** 动态将 Agent 行为写回 Zep 图谱，实现记忆的时序更新
5. **SimulationIPC** 实现 Flask ↔ 模拟脚本的进程间通信（支持 Interview 和 CLOSE_ENV 命令）

### Step 4：报告生成

**目标**：Report Agent 基于 Zep 图谱深度检索生成预测报告

1. Report Agent 采用 **ReACT 模式**（Reasoning + Acting）：
   - 先规划报告目录结构
   - 分段生成，每段多轮思考与反思
2. 拥有丰富的 Zep 检索工具集：
   - **InsightForge**（深度洞察检索）：自动生成子问题，多维度混合检索
   - **PanoramaSearch**（广度搜索）：获取全貌，包括过期内容
   - **QuickSearch**（简单搜索）：快速关键词检索
3. **ReportLogger** 记录 Agent 每步详细动作到 `agent_log.jsonl`

### Step 5：深度互动

**目标**：与模拟世界中的 Agent 对话，与 Report Agent 对话

1. **Interview 功能**：通过 IPC 向运行中的模拟环境发送采访命令
   - 支持单个或批量 Agent 采访
   - 支持指定平台（Twitter/Reddit/双平台）
2. **Report Agent 对话**：用户可就报告内容追问，Agent 自主调用检索工具回答

---

## 四、架构设计要点

### 4.1 Flask 应用工厂模式

```python
def create_app(config_class=Config):
    app = Flask(__name__)
    CORS(app, resources={r"/api/*": {"origins": "*"}})
    app.register_blueprint(graph_bp, url_prefix='/api/graph')
    app.register_blueprint(simulation_bp, url_prefix='/api/simulation')
    app.register_blueprint(report_bp, url_prefix='/api/report')
```

三个蓝图对应三个核心业务域：图谱、模拟、报告。职责清晰，互不干扰。

### 4.2 基于文件系统的持久化

项目采用 JSON 文件存储，无数据库依赖：

- Project 数据：`uploads/projects/{project_id}/project.json`
- Simulation 状态：`uploads/simulations/{sim_id}/`

这种设计简化了部署，适合中小规模使用。当然，在大规模场景下会成为瓶颈。

### 4.3 线程安全的任务管理

```python
class TaskManager:
    _instance = None  # 单例模式
    _lock = threading.Lock()  # 线程安全
```

TaskManager 使用单例模式 + 线程锁，管理长时间运行的任务状态（图谱构建、模拟运行等），确保并发安全。

### 4.4 基于文件系统的进程间通信（IPC）

```
Flask 进程                    模拟脚本进程
    │                              │
    │  写入 commands/{id}.json     │
    │─────────────────────────────>│
    │                              │  轮询 commands/
    │                              │  执行命令
    │  <──────────────────────────│
    │  轮询 responses/{id}.json   │  写入 responses/{id}.json
```

Flask 与仿真脚本通过文件系统通信，避免进程间耦合，稳定可靠。但轮询机制有延迟，不如 Socket/gRPC 实时。支持三种命令类型：INTERVIEW（单个采访）、BATCH_INTERVIEW（批量采访）、CLOSE_ENV（关闭环境）。

### 4.5 LLM 客户端统一封装

```python
class LLMClient:
    def __init__(self, api_key=None, base_url=None, model=None):
        self.client = OpenAI(api_key=self.api_key, base_url=self.base_url)
    
    def chat(self, messages, temperature=0.7, max_tokens=4096, response_format=None) -> str
    def chat_json(self, messages, temperature=0.3, max_tokens=4096) -> Dict
```

兼容 OpenAI SDK 格式的任意 LLM API，推荐阿里百炼 qwen-plus（性价比高）。`chat_json` 方法支持 JSON 模式输出，并自动清理部分模型（如 MiniMax M2.5）的 `<think/>` 思考标签。

---

## 五、数据流全景

```
┌──────────────┐
│   用户上传    │  PDF/TXT/MD + 模拟需求
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Step 1      │  OntologyGenerator → LLM → 本体定义
│  图谱构建    │  TextProcessor → 文本切块
│              │  GraphBuilderService → Zep API → 知识图谱
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Step 2      │  ZepEntityReader → 读取实体/关系
│  环境搭建    │  OasisProfileGenerator → Agent 人设
│              │  SimulationConfigGenerator → 模拟配置
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Step 3      │  SimulationRunner → 后台进程
│  开始模拟    │  OASIS → Twitter + Reddit 双平台并行
│              │  ZepGraphMemoryUpdater → 动态记忆更新
│              │  SimulationIPC → 采访通信
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Step 4      │  ReportAgent → ReACT 模式
│  报告生成    │  ZepTools → InsightForge / Panorama / QuickSearch
│              │  ReportLogger → 详细动作日志
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Step 5      │  Interview → 与任意 Agent 对话
│  深度互动    │  Chat → 与 Report Agent 继续讨论
└──────────────┘
```

---

## 六、Zep Cloud 的三重角色

MiroFish 对 Zep Cloud 的运用堪称巧妙，它在系统中扮演三重角色：

1. **知识图谱存储**：Step 1 构建的实体、关系、属性全部存储在 Zep Graph 中
2. **GraphRAG 检索引擎**：Step 4 的 Report Agent 通过 Zep 的检索 API 进行多维度信息检索
3. **动态记忆更新器**：Step 3 模拟过程中，Agent 的行为（发帖、评论、互动）作为新 Episode 写回 Zep，实现实体记忆的时序更新

这种设计让 Zep 成为贯穿整个工作流的核心数据基础设施。

---

## 七、部署方式

### 方式一：源码部署（推荐）

```bash
# 1. 配置环境变量
cp .env.example .env
# 编辑 .env 填入 LLM_API_KEY, LLM_BASE_URL, LLM_MODEL_NAME, ZEP_API_KEY

# 2. 一键安装依赖
npm run setup:all

# 3. 启动服务
npm run dev
# 前端: http://localhost:3000
# 后端: http://localhost:5001
```

### 方式二：Docker 部署

```bash
cp .env.example .env
docker compose up -d
# 自动映射 3000/5001 端口
```

### 必需环境变量

| 变量 | 说明 | 推荐值 |
|------|------|--------|
| `LLM_API_KEY` | LLM API 密钥 | 阿里百炼 API Key |
| `LLM_BASE_URL` | LLM API 地址 | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| `LLM_MODEL_NAME` | 模型名称 | `qwen-plus` |
| `ZEP_API_KEY` | Zep Cloud API 密钥 | 免费额度即可 |

---

## 八、关键依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `flask` | ≥3.0 | Web 框架 |
| `openai` | ≥1.0 | LLM API 调用 |
| `zep-cloud` | 3.13.0 | 知识图谱服务 |
| `camel-oasis` | 0.2.5 | 社交媒体仿真引擎 |
| `camel-ai` | 0.2.78 | 多 Agent 框架 |
| `PyMuPDF` | ≥1.24 | PDF 文件解析 |

---

## 九、设计亮点与局限

### 设计亮点

1. **五步流水线清晰**：图谱构建 → 环境搭建 → 模拟运行 → 报告生成 → 深度互动，每一步职责明确
2. **Zep Cloud 三合一**：既是图谱存储，又是 GraphRAG 检索引擎，还是动态记忆更新载体，减少了技术栈复杂度
3. **IPC 解耦**：Flask 与仿真脚本通过文件系统通信，避免进程间耦合，稳定可靠
4. **ReACT 模式报告**：Report Agent 不是简单生成，而是先规划再分段深度检索，质量更高
5. **零数据库依赖**：基于文件系统的持久化，部署极简
6. **LLM 无关性**：OpenAI 兼容格式，可自由切换 LLM 提供商

### 局限

1. **文件持久化**：无数据库，不适合高并发和大数据量场景
2. **IPC 文件通信**：轮询机制有延迟，不如 Socket/gRPC 实时
3. **内存任务管理**：TaskManager 为进程内单例，重启丢失
4. **LLM 成本**：大规模模拟消耗大量 Token，成本可能很高
5. **Zep Cloud 依赖**：深度绑定 Zep 云服务，无本地替代方案

---

## 十、应用场景

| 场景 | 示例 |
|------|------|
| 舆情预测 | 高校舆情推演、品牌危机模拟 |
| 金融分析 | 政策信号推演、市场情绪预测 |
| 文学创作 | 《红楼梦》失传结局推演 |
| 政策模拟 | 政策发布后公众反应预测 |
| 社会仿真 | 突发事件信息传播路径模拟 |

---

## 总结

MiroFish 展示了一种令人兴奋的技术路线：用多智能体仿真来预测未来。它的架构设计务实而清晰——Flask 提供稳定的 API 层，Zep Cloud 作为知识图谱的统一数据基础设施，OASIS 提供经过验证的社交仿真引擎，ReACT 模式的 Report Agent 保证报告质量。

虽然在规模化（文件存储、IPC 通信、Token 成本）方面还有改进空间，但作为一个开源项目，MiroFish 已经提供了一个完整可用的群体智能预测平台。对于想要理解多智能体系统如何实现社会仿真的开发者来说，这是一个值得深入研究的优秀参考项目。

> "让未来在数字沙盘中预演，助决策在百战模拟后胜出。"

---

**参考资料**

1. MiroFish GitHub 仓库：https://github.com/666ghj/MiroFish
2. OASIS 项目：https://github.com/camel-ai/oasis
3. Zep Cloud 文档：https://help.getzep.com/
4. CAMEL-AI 框架：https://github.com/camel-ai/camel

**版本信息**

- 分析基于：MiroFish v0.1.0（2026年4月源码）
- 文档版本：v1.0（2026年4月9日）

*本文为基于源码的技术分析文档，所有结论基于公开可验证的信息。*
