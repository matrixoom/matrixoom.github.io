---
title: "DBHub 源码深度解析：MCP 数据库网关是怎么炼成的"
date: 2026-06-04 20:50:00
tags:
  - MCP
  - 数据库
  - AI Agent
  - 开源分析
  - DevOps
categories: 项目分析
---

## 一、先说这是干什么的

你让 AI 帮你查数据库，AI 说"你先手动导出 schema，我看看有什么表"。这件事的体验有多差，用过的都知道。

DBHub 解决的就是这个问题：**一个 MCP 服务器，让 AI 助手能直接探索数据库结构、执行 SQL 查询，还能把常用操作封装成可复用的命名工具。** GitHub: [bytebase/dbhub](https://github.com/bytebase/dbhub)，MIT 协议，2900+ star。

背后的团队是 Bytebase，做开源数据库 DevSecOps 平台的。DBHub 是他们把数据库治理经验浓缩进 MCP 协议的一个工具——轻量、零依赖、token 友好。

<svg viewBox="0 0 680 360" width="100%" role="img" aria-label="DBHub系统架构总览">
<rect width="680" height="360" fill="#FAFAFA" rx="8"/>
<text x="340" y="28" text-anchor="middle" font-family="sans-serif" font-size="16" font-weight="bold" fill="#18191C">DBHub 系统架构总览</text>
<!-- AI Clients Layer -->
<rect x="20" y="48" width="160" height="280" fill="#E8F0FE" stroke="#1a73e8" stroke-width="1.5" rx="6"/>
<text x="100" y="72" text-anchor="middle" font-family="sans-serif" font-size="13" font-weight="bold" fill="#1a73e8">MCP 客户端</text>
<text x="70" y="98" font-family="sans-serif" font-size="11" fill="#444">Claude Desktop</text>
<text x="70" y="118" font-family="sans-serif" font-size="11" fill="#444">Claude Code</text>
<text x="70" y="138" font-family="sans-serif" font-size="11" fill="#444">Cursor</text>
<text x="70" y="158" font-family="sans-serif" font-size="11" fill="#444">VS Code</text>
<text x="70" y="178" font-family="sans-serif" font-size="11" fill="#444">WorkBuddy</text>
<text x="70" y="198" font-family="sans-serif" font-size="11" fill="#444">任意 MCP 客户端</text>

<!-- MCP Protocol arrows -->
<line x1="180" y1="140" x2="220" y2="140" stroke="#555" stroke-width="2" marker-end="url(#arrowMCP)"/>
<text x="200" y="130" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#888">MCP</text>
<text x="200" y="160" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#888">JSON-RPC</text>
<defs>
<marker id="arrowMCP" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#555"/></marker>
<marker id="arrowDB" markerWidth="8" markerHeight="6" refX="8" refY="3" orient="auto"><path d="M0,0 L8,3 L0,6" fill="#1a73e8"/></marker>
</defs>

<!-- DBHub Core Box -->
<rect x="230" y="48" width="220" height="280" fill="#FFF" stroke="#1a73e8" stroke-width="2" rx="8"/>
<text x="340" y="70" text-anchor="middle" font-family="sans-serif" font-size="14" font-weight="bold" fill="#18191C">DBHub MCP Server</text>

<!-- Internal modules -->
<rect x="245" y="82" width="190" height="40" fill="#E8F5E9" rx="4"/>
<text x="340" y="98" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="bold" fill="#2e7d32">Transport Layer</text>
<text x="340" y="114" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">HTTP (stateless) · STDIO (local)</text>

<rect x="245" y="130" width="190" height="40" fill="#FFF3E0" rx="4"/>
<text x="340" y="146" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="bold" fill="#e65100">Tool Registry</text>
<text x="340" y="162" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">Built-in · Custom Tools · Multi-source</text>

<rect x="245" y="178" width="190" height="40" fill="#F3E5F5" rx="4"/>
<text x="340" y="194" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="bold" fill="#6a1b9a">Security Layer</text>
<text x="340" y="210" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">RO Check · Row Limit · DSN Redaction</text>

<rect x="245" y="226" width="190" height="40" fill="#E0F7FA" rx="4"/>
<text x="340" y="242" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="bold" fill="#00695c">Connector Manager</text>
<text x="340" y="258" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">Pool · SSH · IAM · Lazy Connect</text>

<rect x="245" y="274" width="190" height="40" fill="#FFFDE7" rx="4"/>
<text x="340" y="290" text-anchor="middle" font-family="sans-serif" font-size="11" font-weight="bold" fill="#f57f17">Config Layer</text>
<text x="340" y="306" text-anchor="middle" font-family="sans-serif" font-size="9" fill="#555">TOML · CLI Env · Hot Reload</text>

<!-- Databases Layer -->
<rect x="480" y="48" width="180" height="280" fill="#E8F5E9" stroke="#2e7d32" stroke-width="1.5" rx="6"/>
<text x="570" y="72" text-anchor="middle" font-family="sans-serif" font-size="13" font-weight="bold" fill="#2e7d32">Databases</text>
<rect x="495" y="85" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="103" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">PostgreSQL</text>
<rect x="495" y="118" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="136" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">MySQL</text>
<rect x="495" y="151" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="169" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">MariaDB</text>
<rect x="495" y="184" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="202" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">SQL Server</text>
<rect x="495" y="217" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="235" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#333">SQLite</text>
<rect x="495" y="250" width="150" height="28" fill="#FFF" rx="3"/><text x="570" y="268" text-anchor="middle" font-family="sans-serif" font-size="10" fill="#555">+ 更多可扩展</text>

<!-- connector arrows -->
<line x1="450" y1="140" x2="480" y2="140" stroke="#1a73e8" stroke-width="2" marker-end="url(#arrowDB)"/>
</svg>

**核心价值一句话：AI 不用复制粘贴数据库 Schema 给你了，它自己就能看。**

---

## 二、系统架构总览

DBHub 代码量不大，`src/` 下一共约 40 个核心源文件，但结构非常清晰。我把它抽象成五个层次：

```
┌──────────────────────────────────────────┐
│  Transport Layer  (server.ts)            │
│  HTTP (stateless) · STDIO (local)        │
├──────────────────────────────────────────┤
│  Tool Registry    (tools/registry.ts)    │
│  管理 tools/list、多数据源工具路由        │
├──────────────────────────────────────────┤
│  Tool Handlers    (tools/*.ts)           │
│  execute_sql · search_objects · custom   │
├──────────────────────────────────────────┤
│  Security Layer   (allowed-keywords.ts,  │
│   sql-row-limiter.ts, dsn-obfuscate.ts)  │
├──────────────────────────────────────────┤
│  Connector Layer  (connectors/*.ts)      │
│  pg · mysql2 · mssql · better-sqlite3    │
│  mariadb + SSH · SSL · AWS IAM          │
└──────────────────────────────────────────┘
```

代码入口在 `src/index.ts`，一共就 20 行。核心逻辑是**懒加载**：启动时注册 5 个数据库驱动模块路径，但不 import 它们——只有需要某个数据库连接时，对应的 npm 包（`pg`、`mysql2` 等）才会被动态 `import()`：

```typescript
// src/index.ts
const connectorModules = [
  { load: () => import("./connectors/postgres/index.js"), name: "PostgreSQL", driver: "pg" },
  { load: () => import("./connectors/sqlserver/index.js"), name: "SQL Server", driver: "mssql" },
  { load: () => import("./connectors/sqlite/index.js"), name: "SQLite", driver: "better-sqlite3" },
  { load: () => import("./connectors/mysql/index.js"), name: "MySQL", driver: "mysql2" },
  { load: () => import("./connectors/mariadb/index.js"), name: "MariaDB", driver: "mariadb" },
];
```

五个驱动包全部放在 `optionalDependencies` 里，npm 安装时不会自动装，只有在对应数据库的连接器被加载时才会 require。这就做到了"零依赖"的效果——如果你只连 PostgreSQL，那 `mysql2`、`mssql` 这些包根本不会进 node_modules。

---

## 三、核心设计拆解

### 3.1 再多数据库，只暴露两个工具

这是 DBHub 最聪明的地方。别的数据库 MCP server 可能暴露一堆工具——`list_tables`、`describe_table`、`get_schema`、`run_query`……每个工具都要在 MCP 的 tools/list 响应里占一坨空间。

DBHub 的选择是：**只给 AI 两个工具，但每个都足够灵活。**

- **`execute_sql`**：执行 SQL，支持多语句（分号分隔）、事务、只读模式、行数上限
- **`search_objects`**：渐进式探索数据库结构——从 schema → table → column → index → procedure，分三层粒度：names / summary / full

多数据源场景下，工具命名会加后缀：`execute_sql_local_pg`、`execute_sql_prod_mysql`、`search_objects_dev_db`。但对 AI 来说始终是同一套指令格式，不需要学新东西。

**渐进式探索**是 `search_objects` 的核心设计：

```
detail_level: "names"     → 只看名字，最少 token
detail_level: "summary"   → 名字 + 行数/列数/注释
detail_level: "full"      → 完整结构（列类型、索引、存储过程定义）
```

AI 先用 `names` 扫一眼有哪些表，发现 `orders` 表可能有数据，再切 `full` 看列定义和索引。这套"先扫再深"的策略，把 token 消耗控制在最小。

### 3.2 Connector 接口：五类数据库，一套抽象

`src/connectors/interface.ts` 定义了所有数据库驱动必须实现的 `Connector` 接口，核心方法：

```typescript
export interface Connector {
  id: ConnectorType;           // "postgres" | "mysql" | ...
  name: string;
  dsnParser: DSNParser;        // 解析连接字符串
  clone(): Connector;          // 多源支持：每个源一个实例
  connect(dsn: string, initScript?: string, config?: ConnectorConfig): Promise<void>;
  disconnect(): Promise<void>;
  getSchemas(): Promise<string[]>;
  getTables(schema?: string): Promise<string[]>;
  getTableSchema(tableName: string, schema?: string): Promise<TableColumn[]>;
  getTableIndexes(tableName: string, schema?: string): Promise<TableIndex[]>;
  getStoredProcedures(schema?: string, routineType?: "procedure" | "function"): Promise<string[]>;
  executeSQL(sql: string, options: ExecuteOptions, parameters?: any[]): Promise<SQLResult>;
}
```

每个数据库有自己的一套系统表查询逻辑。以 PostgreSQL 为例，`getTables` 实际执行的是：

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema = $1 AND table_type = 'BASE TABLE'
ORDER BY table_name
```

`search_objects` 里的行数估算也是通过系统表统计（`pg_class.reltuples`），而不是 `COUNT(*)`——百万级别的表秒出结果。

多数据源场景下，每个源的连接是独立 clone 的。`ConnectorManager` 内部维护了一个 `Map<string, Connector>`，source ID 做 key。还支持 **lazy 连接**——配置了 `lazy = true` 的数据源启动时不连，只在第一次查询时才建立连接：

```typescript
// src/connectors/manager.ts
private lazySources: Map<string, SourceConfig> = new Map();
private pendingConnections: Map<string, Promise<void>> = new Map(); // 防并发竞态

async ensureConnected(sourceId?: string): Promise<void> {
  // 已连接：直接返回
  if (this.connectors.has(id)) return;
  // 正在连接中（并发保护）：共享同一个 Promise
  const pending = this.pendingConnections.get(id);
  if (pending) return pending;
  // 启动连接
  const connectionPromise = (async () => {
    await this.connectSource(lazySource);
    this.lazySources.delete(id);
  })();
  this.pendingConnections.set(id, connectionPromise);
  return connectionPromise;
}
```

### 3.3 SQL 安全引擎：不只是关键字白名单

安全性代码集中在两个文件：

**`allowed-keywords.ts`** — 读操作白名单：

```typescript
export const allowedKeywords: Record<ConnectorType, string[]> = {
  postgres: ["select", "with", "explain", "show"],
  mysql:    ["select", "with", "explain", "show", "describe", "desc"],
  sqlite:   ["select", "with", "explain", "pragma"],
  // ...
};
```

但光检查第一个关键字远远不够。`isReadOnlySQL` 做了**三层防御**：

1. **剥离注释和字符串**：先把 SQL 里的注释（`--`、`/**/`）和字符串字面量全部替换为空格，防止 `SELECT 1; DROP TABLE /* comment */ users` 这种注入
2. **CTE 内 DML 检测**：`WITH cte AS (UPDATE users SET ...) SELECT * FROM cte` 会被拦截，因为 `WITH` 体内含 mutating 关键字
3. **`SELECT INTO` 检测**：`SELECT * INTO new_table FROM old_table` 不是纯读操作
4. **`EXPLAIN ANALYZE` 防御**：PostgreSQL 的 EXPLAIN ANALYZE 会真正执行语句，里面不能套 DML

**`sql-parser.ts`** 实现了按数据库方言的 tokenizer：

```typescript
const dialectScanners: Record<ConnectorType, TokenScanner> = {
  postgres: scanTokenPostgres,   // 支持 $tag$ dollar-quote
  mysql:    scanTokenMySQL,      // 支持 `` 反引号、/*! 条件注释
  sqlite:   scanTokenSQLite,     // 支持 [] 方括号标识符
  sqlserver: scanTokenSQLServer, // 支持 [] 方括号
};
```

特别要提 MySQL/MariaDB 的条件注释处理。MySQL 的 `/*!50708 SELECT ... */` 在实际执行时不是注释——它是"版本条件执行"，低版本当注释，高版本执行内部 SQL。DBHub 的 scanner 不会把这类块当注释剥离，防止恶意语句夹带其中：

```typescript
function scanMultiLineCommentMySQL(sql: string, i: number): SQLToken | null {
  if (sql[i] !== "/" || sql[i + 1] !== "*") return null;
  const next = sql[i + 2];
  // /*! 和 /*M! 开头的是条件注释，必须保留
  if (next === "!" || (next === "M" && sql[i + 3] === "!")) return null;
  return scanMultiLineComment(sql, i);
}
```

### 3.4 Custom Tools：把 SQL 封装成"技能"

在 `dbhub.toml` 的 `[[tools]]` 段里可以定义可复用的参数化 SQL 操作：

```toml
[[tools]]
name = "salary_search"
description = "Find employees earning at least min_salary, optionally capped by max_salary"
source = "local_pg"
readonly = true
max_rows = 1000
statement = """
  SELECT e.emp_no, e.first_name, e.last_name, s.amount as salary
  FROM employee e JOIN salary s ON e.emp_no = s.emp_no
  WHERE s.amount >= $1 AND ($2::int IS NULL OR s.amount <= $2)
  ORDER BY s.amount DESC LIMIT 100
"""

[[tools.parameters]]
name = "min_salary"
type = "integer"
description = "Minimum salary (required)"
required = true

[[tools.parameters]]
name = "max_salary"
type = "integer"
description = "Maximum salary (optional, defaults to no limit)"
required = false
```

定义后 MCP 客户端会看到一个新工具 `salary_search`，AI 可以直接调用它。

底层实现也不复杂：`custom-tool-handler.ts` 用 Zod 从参数定义自动生成输入校验 Schema，`parameter-mapper.ts` 把命名参数映射到数据库的占位符顺序（PostgreSQL 用 `$1/$2`，MySQL 用 `?/?`，SQL Server 用 `@p1/@p2`）。

这本质上是把 SQL 能力**模板化**了。DBA 可以提前写好经过审核的查询模板，程序员用自然语言调用，不用每个人都会写 SQL，也不用担心 SQL 注入。

---

## 四、部署方式详解

### 4.1 最简单的：Demo 模式

```bash
npx @bytebase/dbhub@latest --transport http --port 8080 --demo
```

启动一个内存 SQLite 数据库，自带员工示例数据（来自 MySQL 的公开 employees 数据集）。0 配置，适合体验。

### 4.2 单数据库：DSN 直连

```bash
# Docker
docker run --rm --init --name dbhub --publish 8080:8080 \
  bytebase/dbhub --transport http --port 8080 \
  --dsn "postgres://user:password@localhost:5432/mydb?sslmode=disable"

# NPM
npx @bytebase/dbhub@latest --transport http --port 8080 \
  --dsn "mysql://root:mysql@localhost:3306/mydb"
```

### 4.3 多数据库：TOML 配置

在项目根目录创建 `dbhub.toml`：

```toml
[[sources]]
id = "dev_pg"
description = "PostgreSQL development database"
dsn = "postgres://dev_user:dev_pass@localhost:5432/myapp_dev?sslmode=disable"

[[sources]]
id = "prod_pg"
description = "Production PostgreSQL (read-only)"
type = "postgres"
host = "10.0.1.100"
port = 5432
database = "myapp"
user = "readonly_user"
password = "secure_password"
sslmode = "require"

[[sources]]
id = "analytics_sqlite"
dsn = "sqlite:///path/to/analytics.db"

# 生产库只读 + 行数限制
[[tools]]
name = "execute_sql"
source = "prod_pg"
readonly = true
max_rows = 500

# 自定义工具
[[tools]]
name = "recent_orders"
description = "查询最近N天的订单"
source = "dev_pg"
readonly = true
statement = "SELECT * FROM orders WHERE created_at >= NOW() - ($1 || ' days')::INTERVAL ORDER BY created_at DESC LIMIT 100"

[[tools.parameters]]
name = "days"
type = "integer"
description = "查询最近多少天的订单"
required = true
```

然后启动：

```bash
npx @bytebase/dbhub@latest --transport http --port 8080
```

DBHub 自动读取当前目录的 `dbhub.toml`。

`buildDSNFromSource` 函数负责把 TOML 配置转成标准连接字符串。它分两种情况处理：有 `dsn` 字段直接用；没有时从 `type`、`host`、`port`、`database`、`user`、`password` 等离散字段拼接。还支持环境变量插值（`${DB_PASSWORD}`），避免密码硬编码。

### 4.4 两种 Transport

| Transport | 适用场景 | 特点 |
|-----------|---------|------|
| **HTTP** | 远程/共享部署 | stateless、每个请求创建新实例、支持 Workbench 前端 |
| **STDIO** | 本地 IDE | 进程间通信、更少的网络开销 |

HTTP 模式下用了 `StreamableHTTPServerTransport`，关键设计是 **stateless**——每个 POST 请求创建一个全新的 MCP server 实例和 transport，完全隔离。这解决了多客户端并发时的请求 ID 冲突问题，代价是每个请求都有初始化开销。

STDIO 模式通过 `StdioServerTransport` 运行，适合 Claude Desktop、Cursor 这类直接启动子进程的客户端。

### 4.5 配置热重载

`config-watcher.ts` 监听 `dbhub.toml` 文件变更（通过 Node.js `fs.watch`）。配置文件改了之后，无需重启 DBHub 服务——新的数据源和工具配置自动生效。但 STDIO 模式下工具列表在启动时已经注册到 MCP 客户端，所以新工具需要客户端重连才能看到。

---

## 五、使用场景实战

### 5.1 AI 自然语言查数据库

这是最直接的用法。在 WorkBuddy、Cursor、Claude Desktop 里连接 DBHub，然后：

- "帮我看下 orders 表有没有缺索引" → AI 先调用 `search_objects` 看表结构和现有索引，再用 `execute_sql` 跑 EXPLAIN 分析执行计划
- "统计过去30天各产品线的 GMV" → AI 自己看 schema，写 SQL，执行
- "users 表里有没有重复 email？" → AI 写 `SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1`

全程不需要你复制粘贴 schema、不需要手动导出 CSV、不需要在数据库客户端和 AI 对话之间切窗口。

### 5.2 多环境数据库管理

同一个 DBHub 实例连接开发、测试、生产三套数据库：

```toml
[[sources]]
id = "dev"    # 开发库：可读写
[[sources]]
id = "staging" # 测试库：只读
[[sources]]
id = "prod"   # 生产库：只读 + 最多1000行
```

然后你可以问："生产库和测试库的 user 表结构有什么差异？"——AI 自动比较两个数据源的 schema。

### 5.3 SaaS 工单自动化

用 Custom Tools 把常见运维查询封装成工具，让非技术人员通过 AI 对话就能查到数据。比如：

- `find_user_by_email` — 输入邮箱查用户信息
- `recent_login_logs` — 查某用户最近的登录记录
- `order_status` — 输入订单号查状态

这些工具在 TOML 里定义好 `readonly = true`，不会有人误操作修改数据。

### 5.4 数据探索和代码生成

开发新功能时，先让 AI 搞清楚数据结构：

- "这个库有多少张表？每张表的关联关系是怎样的？"
- "根据 orders 和 order_items 表，生成一个 TypeScript 类型定义"
- "帮我写一个查询，找出购买了X商品又买了Y商品的用户"

AI 自己探索 schema，生成的代码是基于真实表结构的，不会用 `any` 瞎编字段名。

---

## 六、技术原理深入

### 6.1 连接串传递的安全处理

数据库连接字符串里有密码，在日志、错误信息、API 响应里绝不能明文出现。DBHub 用了三层保护：

1. **`obfuscateDSNPassword`**：启动日志输出 `postgres://user:***@host:5432/db` 而不是真实密码
2. **API 响应脱敏**：`/api/sources` 返回的 source 信息里通过 `buildSourceDisplayInfo` 生成 `display_dsn` 字段，已脱敏
3. **TOML 环境变量插值**：`${DB_PASSWORD}` 这种写法在 TOML 里解析后从 `process.env` 获取，文件本身可以不包含密码

### 6.2 多语句支持与事务

`execute_sql` 支持用分号分隔多条 SQL 语句。`splitSQLStatements` 按方言识别分号在字符串/注释/引号内的情况，不会错误分割：

```typescript
// PostgreSQL 的 $$ 块内可能包含分号，不会被分割
const sql = `CREATE FUNCTION foo() RETURNS void AS $$ 
  SELECT 1; SELECT 2;  
$$ LANGUAGE sql;
SELECT * FROM bar;`;

const stmts = splitSQLStatements(sql, "postgres");
// → ["CREATE FUNCTION foo() ...", "SELECT * FROM bar"]
// $$ 块内的 SELECT 1; SELECT 2; 被正确保留在一个 statement 里
```

### 6.3 SSH 隧道

支持通过 SSH 跳板连接不可公网直达的数据库：

```toml
[[sources]]
id = "prod_pg"
dsn = "postgres://app_user:pass@10.0.1.100:5432/myapp?sslmode=require"
ssh_host = "bastion.company.com"
ssh_user = "deploy"
ssh_key = "~/.ssh/id_ed25519"
```

实现原理很简单：`ssh-tunnel.ts` 通过 `ssh2` 库建立 SSH 连接，在本机开一个随机端口，然后把 DSN 的 host:port 替换为 `127.0.0.1:随机端口`。后续数据库客户端 unknowingly 通过这个本地端口走了加密隧道。

还支持直接从 `~/.ssh/config` 读取主机别名和配置。

### 6.4 AWS RDS IAM 认证

不需要密码，用 AWS IAM token 认证：

```toml
[[sources]]
id = "rds_pg"
type = "postgres"
host = "mydb.abc123.eu-west-1.rds.amazonaws.com"
port = 5432
database = "myapp"
user = "dbuser@example.com"
aws_iam_auth = true
aws_region = "eu-west-1"
sslmode = "require"
```

IAM token 有效期 15 分钟，`ConnectorManager` 在初始化时会设置一个定时器，每 14 分钟自动刷新：断开旧连接 → 生成新 token → 重新连接。三个连接阶段无缝切换。

### 6.5 内置 Workbench

DBHub 不仅是一个 MCP server，还自带了一个 React 前端（`frontend/` 目录），发布在 npm 包里。启动 HTTP 模式后访问 `http://localhost:8080/` 就能打开：

- 可视化执行 SQL
- 运行 Custom Tools
- 查看请求追踪（耗时、SQL、结果行数）
- 浏览数据源配置

前端用 Vite + React + shadcn/ui 构建，打包后约 300KB gzip，对 MCP server 本身的体积影响不大。后端通过 `/api/sources` 和 `/api/requests` 暴露 REST API 供前端消费。

### 6.6 DNS 重绑定保护

HTTP 模式下有一个容易被忽略但很重要的安全处理——DNS rebinding 检测：

```typescript
app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin) {
    const host = (req.headers.host ?? '').split(':')[0].toLowerCase();
    const originHost = new URL(origin).hostname.toLowerCase();
    if (originHost !== host) {
      return res.status(403).json({ error: 'Forbidden' });
    }
  }
  // ...
});
```

防止恶意网站通过 DNS 重绑定攻击访问你本机运行的 DBHub 实例。无 Origin 头的请求（非浏览器 MCP 客户端）不受影响。

---

## 七、亮点与局限

### 亮点

1. **极简工具设计**：只有 2 个内置工具，但探索能力不缩水。Token 效率做到了同类产品里最好。对比之下 Supabase MCP 有 15+ 个工具，每次 tools/list 都吃掉大片上下文。
2. **数据库方言感知的 SQL 解析**：不是简单的字符串匹配，而是按方言（PostgreSQL dollar-quote、MySQL 反引号/条件注释、SQL Server 方括号）实现 tokenizer。安全防护建立在正确的语法解析之上。
3. **纯 stateless HTTP 模式**：每个请求创建全新的 MCP server + transport 实例，天然支持多客户端无状态部署。这样就不需要 session 管理、不需要分布式锁。
4. **lazy connect + 并发保护**：标记了 `lazy = true` 的源启动不连、第一次用才连，而且通过 `pendingConnections` Promise 去重，多人同时触发不会建立多余连接。
5. **配置驱动的 Custom Tools**：DBA 维护 TOML 文件 → 程序员通过自然语言调用 → AI 执行经过审核的 SQL。这比每个人自己写 SQL 安全得多。

### 局限和取舍

1. **STDIO 模式不支持热重载**：工具列表在启动时注册，改了 TOML 配置不会自动让 MCP 客户端看到新工具。HTTP 模式不存在这个问题。
2. **没有内置权限系统**：所有配置好的工具对 MCP 客户端平等暴露。如果你需要"A 组只能看 dev 库、B 组能看 prod 库"，需要跑多个 DBHub 实例或在网络层做隔离。
3. **不支持 NoSQL**：目前只覆盖关系型数据库。MongoDB、Elasticsearch、Redis 不在支持列表中。
4. **单测覆盖不平衡**：`src/__tests__/` 只有约 10 个核心测试文件，集成测试依赖真实的数据库容器（testcontainers），CI 配置依赖较多。
5. **Custom Tools 的 SQL 注入风险**：虽然参数化查询是安全的，但如果有人在 TOML 里写了拼字符串的 SQL（比如 `${param}` 插值），DBHub 不会拦截。这个责任在配置编写者。

---

## 八、总结

DBHub 不是第一个数据库 MCP server，但它做到了目前最清爽的设计。

核心洞察就一条：**AI 不需要 15 个工具来操作数据库，两个就够了——一个探索结构，一个执行 SQL。** 剩下的能力通过 Custom Tools 按需扩展，由熟悉数据库的人提前定义好模板。

从代码质量来看，参数校验（Zod）、方言感知的 SQL 解析、stateless HTTP 设计、懒加载驱动这些细节都体现了一个有经验的团队的产品思维——不是"能跑就行"，而是"怎么跑才安全"。

如果你团队的 AI 工具有数据库查询需求，DBHub 是目前 MCP 生态里最值得考虑的方案。

---

*报告来源：DBHub GitHub 仓库 [github.com/bytebase/dbhub](https://github.com/bytebase/dbhub)（v0.21.2），MIT 协议。文章撰写基于源码阅读，非官方文档搬运。*
