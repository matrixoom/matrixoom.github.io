---
title: "Redis 7 使用规范与最佳实践：从性能陷阱到 7.0 新特性全解析"
date: 2026-05-05
categories:
  - Redis
  - 后端开发
  - 性能优化
tags:
  - Redis 7
  - 性能优化
  - 使用规范
  - 最佳实践
  - ACL
  - Redis Functions
description: "针对 Redis 7 线上环境，系统梳理开发规范与最佳实践。深度剖析 HGETALL、LRANGE 0 -1、SCAN 大 count 等常见性能陷阱，并提供正例代码。专章讲解 Redis 7.0 相对 5.0 的重大新特性（Functions / ACL v2 / Sharded Pub/Sub 等）。"
cover: /img/redis7-cover.svg
---

> "Redis 用好了是神器，用不好是炸弹。" 本文基于 Redis 7.0 官方文档与生产经验，系统梳理开发规范，帮你避开那些让线上 CPU 飙升、流量打满的坑。

---

## 一、为什么需要 Redis 使用规范？

Redis 是单线程模型（主要命令执行线程），**一个慢命令会阻塞所有后续请求**。线上遇到的问题，90% 都来自以下几类：

| 问题命令 | 典型后果 | 严重程度 |
|---------|---------|---------|
| `HGETALL` 大 Hash（万级字段）| 网络风暴 + 阻塞 | 🔴 高危 |
| `LRANGE key 0 -1` | 返回全量数据，O(N) | 🔴 高危 |
| `MGET` / `MSET` 批量 5000+ key | 网络阻塞 + 超时 | 🟠 中高危 |
| `SCAN` / `HSCAN` count 过大或在 while 循环 | CPU 飙升 | 🟠 中高危 |
| `KEYS *` | 全量遍历，直接阻塞 | ⚫ 严禁 |
| 大 Value（> 10KB 字符串 / > 5000 元素集合）| 网络 / 内存碎片 | 🟡 中等 |

<svg viewBox="0 0 680 240" xmlns="http://www.w3.org/2000/svg" font-family="'PingFang SC','Microsoft YaHei',sans-serif">
<rect width="680" height="240" fill="#f8f9fa" rx="12"/>
<text x="340" y="28" text-anchor="middle" font-size="15" font-weight="bold" fill="#2c3e50">Redis 单线程模型：一个慢命令的影响</text>
<rect x="40" y="50" width="600" height="36" rx="6" fill="#3498db"/>
<text x="55" y="73" font-size="12" fill="white">时间轴 →</text>
<rect x="140" y="50" width="80" height="36" rx="6" fill="#2ecc71"/>
<text x="180" y="73" text-anchor="middle" font-size="11" fill="white">SET (快)</text>
<rect x="230" y="50" width="80" height="36" rx="6" fill="#2ecc71"/>
<text x="270" y="73" text-anchor="middle" font-size="11" fill="white">GET (快)</text>
<rect x="320" y="50" width="180" height="36" rx="6" fill="#e74c3c"/>
<text x="410" y="67" text-anchor="middle" font-size="11" fill="white">HGETALL big_hash</text>
<text x="410" y="83" text-anchor="middle" font-size="10" fill="#fadbd8">⚠️ 阻塞 500ms</text>
<rect x="510" y="50" width="130" height="36" rx="6" fill="#95a5a6"/>
<text x="575" y="73" text-anchor="middle" font-size="11" fill="white">后续请求排队</text>
<path d="M140,100 L140,140" stroke="#e74c3c" stroke-width="2" fill="none" marker-end="url(#arrRed)"/>
<rect x="100" y="142" width="240" height="70" rx="8" fill="#fadbd8" stroke="#e74c3c" stroke-width="1.5"/>
<text x="220" y="167" text-anchor="middle" font-size="13" font-weight="bold" fill="#c0392b">单线程阻塞效应</text>
<text x="220" y="188" text-anchor="middle" font-size="11" fill="#7f8c8d">HGETALL 执行期间，</text>
<text x="220" y="204" text-anchor="middle" font-size="11" fill="#7f8c8d">所有其他客户端请求被阻塞</text>
<path d="M510,100 L510,142" stroke="#e67e22" stroke-width="2" fill="none" marker-end="url(#arrO)"/>
<rect x="460" y="142" width="200" height="70" rx="8" fill="#fdebd0" stroke="#e67e22" stroke-width="1.5"/>
<text x="560" y="167" text-anchor="middle" font-size="13" font-weight="bold" fill="#d68910">排队请求飙升</text>
<text x="560" y="188" text-anchor="middle" font-size="11" fill="#7f8c8d">客户端超时、连接池耗尽、</text>
<text x="560" y="204" text-anchor="middle" font-size="11" fill="#7f8c8d">上游服务响应变慢</text>
<defs>
<marker id="arrRed" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto"><path d="M0,0 L4,4 L0,8" fill="#e74c3c" stroke="none"/></marker>
<marker id="arrO" markerWidth="8" markerHeight="8" refX="4" refY="4" orient="auto"><path d="M0,0 L4,4 L0,8" fill="#e67e22" stroke="none"/></marker>
</defs>
</svg>

---

## 二、Redis 7.0 vs 5.0 新特性专题

Redis 7.0（2022 年 4 月 GA）相对 5.0 有多项重大升级，以下逐一讲解。

### 2.1 Redis Functions（取代 Lua 脚本的最佳实践）

**5.0 的问题**：Lua 脚本通过 `EVAL` / `EVALSHA` 执行，脚本存储在客户端，难以统一管理，集群模式下还有 hash slot 限制。

**7.0 的解决方案**：`Functions` 将 Lua 脚本**存储在服务端**，通过 `FUNCTION LOAD` 加载，全局可被所有客户端调用。

```bash
# Redis 5.0：客户端每次发送 Lua 脚本
EVAL "redis.call('set',KEYS[1],ARGV[1])" 1 mykey myval

# Redis 7.0：服务端注册函数，客户端只需调用函数名
FUNCTION LOAD "redis.register_function('my_set', function(keys, args)
    return redis.call('SET', keys[1], args[1])
end)"

# 调用（无需再传脚本内容）
FCALL my_set 1 mykey myval
```

**核心优势**：
- 脚本在服务端持久化，重启不丢失
- 支持 `FUNCTION LIST` / `FUNCTION DELETE` 管理
- 集群模式下自动广播到所有节点

### 2.2 ACL v2（更精细的权限控制）

**5.0 的 ACL**：只能控制键的**命令级别**权限（`ACL SETUSER alice on >pass ~* +@all`）。

**7.0 的 ACL v2** 新增：
- **Selector（选择器）**：对同一用户在不同 key 上设置不同权限

```bash
# Redis 7.0：精细到具体 key 模式 + 具体命令
ACL SETUSER dev on >pass
  ~app:* +@read
  ~admin:* -@all +HGET
  ~audit:* +APPEND
```

- **Pub/Sub 频道权限控制**（5.0 无法限制订阅哪个频道）

```bash
# 允许订阅 order.* 频道，禁止订阅 admin.*
ACL SETUSER subscriber on >pass
  +SUBSCRIBE +PSUBSCRIBE
  &order.*      # 允许订阅的频道模式
  -&admin.*     # 禁止订阅的频道模式
```

### 2.3 Sharded Pub/Sub（集群模式下的真正发布订阅）

**5.0 的问题**：集群模式下 `PUBLISH` 会**广播到所有节点**，订阅者无论在哪台节点都能收到，但带来大量内部集群流量。

**7.0 的 Sharded Pub/Sub**：通过 `SSUBSCRIBE` / `SPUBLISH`，消息只发到**负责对应 slot 的节点**，大幅降低集群内部流量。

```bash
# 传统 Pub/Sub（集群内广播，所有节点都收到）
PUBLISH order.created '{"id": 123}'

# Sharded Pub/Sub（只发到对应 slot 的节点）
SPUBLISH order.created '{"id": 123}'
SSUBSCRIBE order.created
```

### 2.4 Client-Eviction（客户端缓存淘汰）

**7.0 新特性**：允许 Redis 在内存达到上限时，**主动淘汰客户端缓存的数据**（配合 `CLIENT TRACKING`），而不是只依赖客户端自己处理。

### 2.5 其他重要改进

| 特性 | 说明 |
|------|------|
| `SET` 命令增强 | 7.0 支持 `GET` 选项：`SET key val GET`（设置并返回旧值，原子操作） |
| `EXPIRE` / `PEXPIRE` 支持 `NX` / `XX` / `GT` / `LT` 选项 | 更精细的过期时间控制 |
| `COPY` 命令 | 跨 key 复制 value（7.0 新增） |
| `LMOVE` / `BLMOVE` | 原子化列表元素移动（替代 `RPOPLPUSH`） |
| 内存优化 | 7.0 对 Hash/List/Set/ZSet 的内部编码做了优化，内存降低约 20% |
| RDB 格式升级 | 支持存储 Function 和 ACL 规则 |

<svg viewBox="0 0 680 360" xmlns="http://www.w3.org/2000/svg" font-family="'PingFang SC','Microsoft YaHei',sans-serif">
<rect width="680" height="360" fill="#f0f8ff" rx="12"/>
<text x="340" y="28" text-anchor="middle" font-size="15" font-weight="bold" fill="#2c3e50">Redis 7.0 新特性架构示意</text>
<rect x="30" y="48" width="190" height="280" rx="10" fill="#e8f8f5" stroke="#1abc9c" stroke-width="1.5"/>
<text x="125" y="72" text-anchor="middle" font-size="13" font-weight="bold" fill="#16a085">Redis 5.0</text>
<rect x="45" y="84" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="105" text-anchor="middle" font-size="11" fill="white">Lua 脚本（EVAL）</text>
<rect x="45" y="124" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="145" text-anchor="middle" font-size="11" fill="white">基础 ACL</text>
<rect x="45" y="164" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="185" text-anchor="middle" font-size="11" fill="white">Pub/Sub 全集群广播</text>
<rect x="45" y="204" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="225" text-anchor="middle" font-size="11" fill="white">Stream 数据类型</text>
<rect x="45" y="244" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="265" text-anchor="middle" font-size="11" fill="white">RDB 持久化</text>
<rect x="45" y="284" width="160" height="32" rx="6" fill="#1abc9c"/>
<text x="125" y="305" text-anchor="middle" font-size="11" fill="white">MODULE 命令</text>
<rect x="240" y="48" width="200" height="280" rx="10" fill="#fef9e7" stroke="#f39c12" stroke-width="2"/>
<text x="340" y="72" text-anchor="middle" font-size="13" font-weight="bold" fill="#d68910">⬆️ Redis 7.0 升级</text>
<rect x="255" y="84" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="95" text-anchor="middle" font-size="10" fill="white">⬆ FUNCTION（服务端脚本）</text>
<text x="340" y="110" text-anchor="middle" font-size="10" fill="#fdebd0">替代 EVAL，支持集群</text>
<rect x="255" y="124" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="135" text-anchor="middle" font-size="10" fill="white">⬆ ACL v2（Selector）</text>
<text x="340" y="150" text-anchor="middle" font-size="10" fill="#fdebd0">按 key 模式控制权限</text>
<rect x="255" y="164" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="175" text-anchor="middle" font-size="10" fill="white">⬆ Sharded Pub/Sub</text>
<text x="340" y="190" text-anchor="middle" font-size="10" fill="#fdebd0">SPUBLISH / SSUBSCRIBE</text>
<rect x="255" y="204" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="215" text-anchor="middle" font-size="10" fill="white">⬆ Client-Eviction</text>
<text x="340" y="230" text-anchor="middle" font-size="10" fill="#fdebd0">客户端缓存淘汰</text>
<rect x="255" y="244" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="255" text-anchor="middle" font-size="10" fill="white">⬆ COPY / LMOVE 命令</text>
<text x="340" y="270" text-anchor="middle" font-size="10" fill="#fdebd0">原子化操作增强</text>
<rect x="255" y="284" width="170" height="32" rx="6" fill="#f39c12"/>
<text x="340" y="295" text-anchor="middle" font-size="10" fill="white">⬆ 内存编码优化 ~20%</text>
<text x="340" y="310" text-anchor="middle" font-size="10" fill="#fdebd0">Hash/List/Set/ZSet</text>
<rect x="460" y="48" width="200" height="280" rx="10" fill="#fdf2f8" stroke="#e91e63" stroke-width="1.5"/>
<text x="560" y="72" text-anchor="middle" font-size="13" font-weight="bold" fill="#c2185b">实践建议</text>
<rect x="470" y="84" width="180" height="50" rx="6" fill="#f48fb1"/>
<text x="560" y="100" text-anchor="middle" font-size="11" fill="white">✅ 新项目用 FUNCTION</text>
<text x="560" y="116" text-anchor="middle" font-size="10" fill="#fce4ec">替代所有 EVAL 脚本</text>
<rect x="470" y="142" width="180" height="50" rx="6" fill="#f48fb1"/>
<text x="560" y="158" text-anchor="middle" font-size="11" fill="white">✅ 集群环境用 SPUBLISH</text>
<text x="560" y="174" text-anchor="middle" font-size="10" fill="#fce4ec">降低跨节点流量</text>
<rect x="470" y="200" width="180" height="50" rx="6" fill="#f48fb1"/>
<text x="560" y="216" text-anchor="middle" font-size="11" fill="white">✅ 用 ACL v2 精细授权</text>
<text x="560" y="232" text-anchor="middle" font-size="10" fill="#fce4ec">按业务划分用户权限</text>
<rect x="470" y="258" width="180" height="50" rx="6" fill="#f48fb1"/>
<text x="560" y="274" text-anchor="middle" font-size="11" fill="white">⚠️ 升级前测试内存变化</text>
<text x="560" y="290" text-anchor="middle" font-size="10" fill="#fce4ec">编码变化可能影响内存</text>
</svg>

---

## 三、常见反例与正例

### 3.1 ❌ 反例：`HGETALL` 大 Hash

```bash
# ❌ 反例：直接获取整个大 Hash
HGETALL user:10086
# 假设该 Hash 有 50000 个字段，返回值大小可能超过 10MB
# 后果：网络传输耗时 + Redis 阻塞

# ❌ 反例：用 HKEYS 获取所有字段后再逐个 HGET
HKEYS user:10086         # 先拿所有 key（同样慢）
# ... 然后在客户端循环 HGET（N 次网络往返）
```

**✅ 正例：分页读取**

```bash
# ✅ 正例 1：用 HSCAN 分批获取
HSCAN user:10086 0 COUNT 100
# 返回 cursor + 100 个字段，循环直至 cursor 回到 0

# ✅ 正例 2：只获取需要的字段（已知字段名）
HMGET user:10086 name age city
# 只返回 3 个字段，O(N) 但 N 很小

# ✅ 正例 3：大对象改用 String + 压缩 JSON
# 写入时
SET user:10086_json '{"name":"Alice","age":30,"city":"SH"}' EX 3600
# 读取时一次 GET，客户端反序列化
GET user:10086_json
```

### 3.2 ❌ 反例：`LRANGE key 0 -1`（获取全量列表）

```python
# ❌ 反例：Redis 列表当消息队列，直接 LRANGE 0 -1 拿全部
import redis
r = redis.Redis()
all_msgs = r.lrange("msg_queue", 0, -1)
# 如果队列里有 10 万条消息，这次操作直接阻塞
```

**✅ 正例：分页 + 用 List 正确姿势**

```python
# ✅ 正例 1：分页读取
def paginated_lrange(key, page=1, page_size=100):
    start = (page - 1) * page_size
    end = start + page_size - 1
    return r.lrange(key, start, end)

# ✅ 正例 2：用 LPOP/RPOP 逐条消费（消息队列场景）
while True:
    # BRPOP 阻塞式弹出，每次只处理 1 条
    result = r.brpop("msg_queue", timeout=5)
    if result:
        process(result[1])

# ✅ 正例 3：超长列表拆分，按时间分 key
# 每天一个新 key，避免单个 key 过大
today = datetime.date.today().isoformat()
r.lpush(f"msg_queue:{today}", json.dumps(msg))
# 过期自动清理
r.expire(f"msg_queue:{today}", 86400 * 7)
```

### 3.3 ❌ 反例：`MGET` / `MSET` 批量过大

```python
# ❌ 反例：一次 MGET 5000 个 key
keys = [f"cache:user:{i}" for i in range(5000)]
values = r.mget(keys)
# 网络包过大，Redis 处理耗时增加，客户端等待超时

# ❌ 反例：一次 MSET 10000 个 key
pipe = r.pipeline()
for i in range(10000):
    pipe.set(f"cache:obj:{i}", f"val_{i}")
pipe.execute()  # 一次发送 10000 个命令
```

**✅ 正例：分批管道**

```python
# ✅ 正例：MGET 分批，每批最多 500 个
def batch_mget(r, keys, batch_size=500):
    results = {}
    for i in range(0, len(keys), batch_size):
        batch = keys[i:i + batch_size]
        vals = r.mget(batch)
        results.update(zip(batch, vals))
    return results

# ✅ 正例：Pipeline 分批执行，每批 500 个
def batch_set(r, kv_dict, batch_size=500):
    keys = list(kv_dict.keys())
    for i in range(0, len(keys), batch_size):
        batch_keys = keys[i:i + batch_size]
        pipe = r.pipeline()
        for k in batch_keys:
            pipe.set(k, kv_dict[k], ex=3600)
        pipe.execute()
```

### 3.4 ❌ 反例：`SCAN` / `HSCAN` count 过大 + while 死循环

```python
# ❌ 反例 1：COUNT 设置过大
# COUNT 只是 hint，Redis 可能返回更多，网络压力巨大
scan_result = r.scan(cursor=0, count=100000)

# ❌ 反例 2：while 循环没有退出保护
cursor = 0
while True:   # ← 如果 cursor 永远不回到 0，死循环！
    cursor, keys = r.scan(cursor=cursor, count=1000)
    process(keys)
    if cursor == 0:
        break

# ❌ 反例 3：在 SCAN 循环里做耗时操作
cursor = 0
while True:
    cursor, keys = r.scan(cursor=cursor, count=1000)
    time.sleep(0.1)   # ← 每次循环 sleep，SCAN 全程耗时极长
    # ... 网络保持占用 ...
```

**✅ 正例：合理 count + 循环保护**

```python
# ✅ 正例 1：COUNT 设为 100~500（小对象）或 500~2000（大对象谨慎）
def safe_scan(r, match_pattern, batch_size=500, max_rounds=1000):
    """
    :param batch_size: SCAN count（建议 100~1000）
    :param max_rounds: 最大循环次数，防止死循环
    """
    cursor = 0
    rounds = 0
    all_keys = []
    while True:
        cursor, keys = r.scan(cursor=cursor, match=match_pattern, count=batch_size)
        all_keys.extend(keys)
        rounds += 1
        if cursor == 0 or rounds >= max_rounds:
            break
    return all_keys

# ✅ 正例 2：使用 Redis 7.0 的 SCAN 时不指定 count（让 Redis 自动决定）
# 或者在知道数据量时，精确设置 count
# 小 Hash（< 1000 字段）：count = 100
# 中等 Hash（1000~10000）：count = 500
# 大 Hash（> 10000）：考虑拆分 Hash，不要用 HSCAN

# ✅ 正例 3：生产环境用 Lua 脚本在服务端过滤（减少网络往返）
# 例如：找出所有过期 key 并删除（服务端执行，一次网络往返）
script = """
local keys = redis.call('SCAN', 0, 'MATCH', ARGV[1], 'COUNT', 500)
local result = {}
for i=1,#keys[2] do
    if redis.call('TTL', keys[2][i]) == -1 then
        redis.call('DEL', keys[2][i])
        table.insert(result, keys[2][i])
    end
end
return result
"""
r.eval(script, 0, "cache:*:*")
```

### 3.5 ❌ 反例：大 Value 问题

```python
# ❌ 反例：把一个 1MB 的 JSON 塞进一个 String
huge_json = json.dumps(get_all_user_data())  # 假设 2MB
r.set("user:all", huge_json)
# 每次 GET 都传输 2MB，网络带宽浪费严重

# ❌ 反例：Set 里加 10 万个成员
for i in range(100000):
    r.sadd("active_users", f"user:{i}")
# SADD 单个命令 O(1)，但 SMEMBERS 返回全部时灾难
```

**✅ 正例：拆分 + 压缩**

```python
# ✅ 正例 1：大对象拆分，用 Hash 的 field 存储
# 每个 field 很小，HGET 只取需要的部分
r.hset("user:10086", "name", "Alice")
r.hset("user:10086", "profile_json", json.dumps(profile))  # 拆出去
r.hset("user:10086", "settings_json", json.dumps(settings))

# ✅ 正例 2：超长列表/集合，用分片
import hashlib
def sharded_key(base_key, member, shards=16):
    shard = int(hashlib.md5(member.encode()).hexdigest(), 16) % shards
    return f"{base_key}:shard:{shard}"

# 10 万个成员分散到 16 个 key
for user_id in all_users:
    shard_key = sharded_key("active_users", user_id)
    r.sadd(shard_key, user_id)

# ✅ 正例 3：大 Value 启用压缩（Redis 7.0 支持）
# 在 redis.conf 中设置
# set-max-intset-entries 512      （小整数集合优化）
# hash-max-ziplist-entries 128    （小 Hash 用 ziplist 编码，省内存）
# list-max-ziplist-size -2         （每个 ziplist 节点不超过 8KB）
```

---

## 四、键值设计规范

### 4.1 Key 命名规范

```bash
# ✅ 推荐格式：业务:子模块:唯一标识
user:profile:10086
order:detail:2024050512345
cache:home:recommend:v2

# ❌ 避免：无分隔符、含义不清
user10086          # 不知道是什么
order_20240505    # 下划线不统一
Cache_Home_Rec    # 大小写混乱
```

### 4.2 过期时间规范

```bash
# ✅ 所有 Key 必须设置 TTL（除非是永久配置）
SET session:user:10086 "token_xxx" EX 7200

# ✅ 防止缓存雪崩：TTL 加随机抖动
import random
ttl = 3600 + random.randint(0, 300)   # 3600~3900 秒随机
r.set("cache:hot:data", value, ex=ttl)

# ❌ 禁止：不设置 TTL 的 key 在线上累积
SET cache:temp:xxx "value"   # ← 没有 EX，永远不过期
```

### 4.3 大 Key 检测与处理

```bash
# 使用 redis-cli 自带的 --bigkeys 检测
redis-cli --bigkeys

# 使用 Redis 7.0 的 MEMORY USAGE 命令查看具体 key 的内存占用
MEMORY USAGE user:10086
# (integer) 4832   ← 该 key 占用 4832 字节

# 使用 SCAN + DEBUG OBJECT（7.0 可用 OBJECT ENCODING 查看编码类型）
OBJECT ENCODING user:10086
# "ziplist"  ← 好现象，紧凑编码
# "hashtable" ← 注意，可能比较大
```

<svg viewBox="0 0 680 260" xmlns="http://www.w3.org/2000/svg" font-family="'PingFang SC','Microsoft YaHei',sans-serif">
<rect width="680" height="260" fill="#f9f9f9" rx="12"/>
<text x="340" y="28" text-anchor="middle" font-size="15" font-weight="bold" fill="#2c3e50">大 Key 问题诊断与处理流程</text>
<rect x="30" y="48" width="140" height="44" rx="8" fill="#3498db"/>
<text x="100" y="65" text-anchor="middle" font-size="12" fill="white" font-weight="bold">发现慢查询</text>
<text x="100" y="82" text-anchor="middle" font-size="10" fill="#d6eaf8">SLOWLOG GET</text>
<path d="M170,70 L200,70" stroke="#7f8c8d" stroke-width="1.5" fill="none" marker-end="url(#arrowS)"/>
<rect x="200" y="48" width="140" height="44" rx="8" fill="#9b59b6"/>
<text x="270" y="65" text-anchor="middle" font-size="12" fill="white" font-weight="bold">定位大 Key</text>
<text x="270" y="82" text-anchor="middle" font-size="10" fill="#e8daef">--bigkeys / MEMORY</text>
<path d="M340,70 L370,70" stroke="#7f8c8d" stroke-width="1.5" fill="none" marker-end="url(#arrowS)"/>
<rect x="370" y="48" width="140" height="44" rx="8" fill="#e74c3c"/>
<text x="440" y="65" text-anchor="middle" font-size="12" fill="white" font-weight="bold">评估影响</text>
<text x="440" y="82" text-anchor="middle" font-size="10" fill="#fadbd8"> > 10KB 或 > 5000元素</text>
<path d="M510,70 L540,70" stroke="#7f8c8d" stroke-width="1.5" fill="none" marker-end="url(#arrowS)"/>
<rect x="540" y="48" width="120" height="44" rx="8" fill="#2ecc71"/>
<text x="600" y="65" text-anchor="middle" font-size="12" fill="white" font-weight="bold">选择方案</text>
<text x="600" y="82" text-anchor="middle" font-size="10" fill="#d5f5e3">拆分/压缩/归档</text>
<rect x="100" y="115" width="140" height="36" rx="6" fill="#f39c12"/>
<text x="170" y="136" text-anchor="middle" font-size="11" fill="white">🔪 拆分 Hash/Set</text>
<rect x="270" y="115" width="140" height="36" rx="6" fill="#f39c12"/>
<text x="340" y="136" text-anchor="middle" font-size="11" fill="white">📦 启用压缩编码</text>
<rect x="440" y="115" width="140" height="36" rx="6" fill="#f39c12"/>
<text x="510" y="136" text-anchor="middle" font-size="11" fill="white">⏰ 设置合理 TTL</text>
<rect x="600" y="115" width="50" height="36" rx="6" fill="#f39c12"/>
<text x="625" y="136" text-anchor="middle" font-size="11" fill="white">🗄 归档到 DB</text>
<path d="M170,155 L170,175" stroke="#f39c12" stroke-width="1.5" fill="none"/>
<path d="M340,155 L340,175" stroke="#f39c12" stroke-width="1.5" fill="none"/>
<path d="M510,155 L510,175" stroke="#f39c12" stroke-width="1.5" fill="none"/>
<path d="M625,155 L625,175" stroke="#f39c12" stroke-width="1.5" fill="none"/>
<rect x="60" y="175" width="560" height="60" rx="8" fill="#fff9e6" stroke="#f39c12" stroke-width="1"/>
<text x="80" y="198" font-size="12" fill="#7f8c8d">📌 经验值（Redis 7.0）：</text>
<text x="80" y="218" font-size="11" fill="#7f8c8d">• String Value &gt; 10KB  → 考虑拆分或压缩</text>
<text x="80" y="235" font-size="11" fill="#7f8c8d">• Hash/Set/ZSet &gt; 5000 元素 → 必须分片</text>
<text x="300" y="235" font-size="11" fill="#7f8c8d">• List 长度 &gt; 10000 → 考虑按时间分 key</text>
<defs>
<marker id="arrowS" markerWidth="8" markerHeight="8" refX="6" refY="3" orient="auto"><path d="M0,0 L6,3 L0,6" fill="#7f8c8d" stroke="none"/></marker>
</defs>
</svg>

---

## 五、连接与客户端规范

### 5.1 连接池配置

```python
# ✅ 正例：合理配置连接池
import redis

pool = redis.ConnectionPool(
    host='127.0.0.1',
    port=6379,
    password='your_password',
    db=0,
    max_connections=50,        # 根据业务 QPS 设置，不要盲目设大
    retry_on_timeout=True,
    health_check_interval=30,   # 定期心跳，避免使用断开的连接
)
r = redis.Redis(connection_pool=pool)

# ❌ 反例：每次操作创建新连接
def get_user(user_id):
    r = redis.Redis(host='127.0.0.1', port=6379)  # ← TCP 握手开销巨大
    return r.get(f"user:{user_id}")
```

### 5.2 Pipeline 与 Transaction

```python
# ✅ 正例：批量写操作用 Pipeline（不保证原子性，但减少 RTT）
pipe = r.pipeline()
for i in range(100):
    pipe.incr(f"counter:{i}")
pipe.execute()   # 1 次网络往返，100 个命令

# ✅ 正例：需要原子性时用 MULTI/EXEC（Transaction）
pipe = r.pipeline()
pipe.multi()
pipe.set("account:A", 100)
pipe.set("account:B", 200)
pipe.decrby("account:A", 50)
pipe.incrby("account:B", 50)
pipe.execute()   # 全部成功或全部失败

# ⚠️ 注意：Redis Transaction 不支持回滚（Rollback）
# 如果中间某条命令失败，已执行的命令不会撤销
```

---

## 六、Redis 7.0 专属最佳实践

### 6.1 用 FUNCTION 替代 Lua 脚本

```bash
# ✅ Redis 7.0：注册函数（只需一次）
FUNCTION LOAD REPLACE
"redis.register_function('rate_limit', function(keys, args)
    local key = keys[1]
    local limit = tonumber(args[1])
    local window = tonumber(args[2])
    local current = redis.call('TIME')[1]
    local clear_before = current - window

    redis.call('ZREMRANGEBYSCORE', key, 0, clear_before)
    local count = redis.call('ZCARD', key)
    if count >= limit then
        return 0   -- 被限流
    end
    redis.call('ZADD', key, current, current)
    redis.call('EXPIRE', key, window)
    return 1   -- 允许通过
end)"

# 调用（超简单，不需要传脚本）
FCALL rate_limit 1 api:req:127.0.0.1 100 60
# 含义：IP 127.0.0.1 在 60 秒内最多 100 次请求
```

### 6.2 用 ACL v2 做业务隔离

```bash
# ✅ Redis 7.0：为每个业务创建独立用户
# 订单服务：只能读写 order:* 键
ACL SETUSER order_svc on >secret_pass ~order:* +@all -@admin

# 报表服务：只读权限
ACL SETUSER report_svc on >secret_pass ~* +@read -@write

# 查看用户列表
ACL LIST

# 持久化 ACL 配置（重启后不丢失）
ACL SAVE
```

---

## 七、监控与预警规范

```bash
# ✅ 必须监控的关键指标

# 1. 慢查询（> 10ms 的命令）
SLOWLOG GET 10
# redis.conf 配置：slowlog-log-slower-than 10000（10ms）

# 2. 内存使用
INFO memory
# 重点关注：used_memory_human / used_memory_peak_human / mem_fragmentation_ratio

# 3. 连接数
INFO clients
# 重点关注：connected_clients（不应超过 maxclients 的 80%）

# 4. 键空间命中率
INFO stats
# 重点关注：keyspace_hits / (keyspace_hits + keyspace_misses)
# 命中率 < 90% 需要排查缓存策略

# 5. 延迟监控（Redis 7.0 增强）
LATENCY HISTORY           # 查看延迟事件历史
LATENCY LATEST            # 查看最新延迟
```

### 监控指标告警阈值建议

| 指标 | 告警阈值 | 处理建议 |
|------|---------|---------|
| 慢查询数量（每分钟）| > 5 条 | 排查具体慢命令，优化或限流 |
| 内存使用率 | > 80% | 清理过期 key 或扩容 |
| 连接数占比 | > 80% | 检查连接池配置，防止泄漏 |
| 缓存命中率 | < 90% | 检查缓存穿透/淘汰策略 |
| 延迟 P99 | > 50ms | 排查大 Key 或持久化阻塞 |

---

## 八、Redis 7.0 升级注意事项

如果你们是从 5.0 升级到 7.0，需要特别注意：

1. **RDB 兼容性**：7.0 的 RDB 格式与 5.0 不完全兼容，**升级前必须备份 RDB 文件**
2. **`redis-cli` 命令变化**：部分命令参数有调整，需要在测试环境验证
3. **内存使用可能变化**：编码优化后内存可能降低，但 Function/ACL 规则也会占用额外内存
4. **客户端库版本**：确保 `redis-py` / `Jedis` 等客户端库升级到支持 Redis 7.0 的版本

```bash
# Python redis-py 版本检查（需要 4.0+ 以完整支持 Redis 7.0）
pip show redis
# 升级
pip install -U redis>=4.5.0
```

---

## 九、规范检查清单

将以下清单发给团队开发人员，作为 Code Review 的 Redis 部分检查项：

```
□ Key 命名是否规范（业务:模块:标识）
□ 所有 Key 是否设置了 TTL（除非是永久配置）
□ 是否存在 HGETALL / HKEYS / LRANGE 0 -1 等全量操作
□ MGET/MSET 批量是否超过 500 个 key
□ SCAN/HSCAN 的 count 是否超过 2000
□ 是否存在未分页的 SCAN while 循环
□ Value 大小是否可能超过 10KB（大 String 需拆分）
□ Hash/Set/List/ZSet 元素是否可能超过 5000（需分片）
□ 是否使用了连接池（禁止每次创建新连接）
□ Pipeline 批量是否超过 500 个命令
□ 是否设置了合理的 max_connections
□ Lua 脚本是否改为了 FUNCTION（Redis 7.0）
□ 是否按业务划分了 ACL 用户（禁止共用 default 用户）
```

---

## 十、总结

Redis 7.0 在 **Function 管理能力、权限控制精细度、集群模式下 Pub/Sub 效率** 上相比 5.0 有质的飞跃。但无论版本如何升级，**慢命令的规避、大 Key 的拆分、连接池的合理配置**这三大原则始终不变。

| 原则 | 核心要点 |
|------|---------|
| **慢命令零容忍** | HGETALL / LRANGE 0 -1 / KEYS * 一律禁止 |
| **大 Key 必拆分** | String > 10KB / 集合 > 5000 元素必须处理 |
| **批量操作要分页** | MGET/SCAN 每批不超过 500~1000 |
| **连接池必须复用** | 禁止每次请求创建新连接 |
| **TTL 必须设置** | 防止过期数据累积 |
| **升级 7.0 用新特性** | FUNCTION 替代 Lua，ACL v2 做隔离 |

> 把这份规范打印出来贴在显示器边上，比事后救火要便宜得多。

---

*本文基于 Redis 7.0.15 官方文档及生产经验整理，适用 Redis 7.0.x 系列版本。如有问题欢迎在博客评论区讨论。*
