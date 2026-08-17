---
layout: post
title: "Python Redis 实战讲义：缓存、限流、会话、队列与分布式锁"
date: 2026-08-17 14:00:00 +0800
categories: [Python基础]
tags: [Python, Redis, 缓存, 限流, 会话, 消息队列, 分布式锁]
---

# Python Redis 实战讲义：缓存、限流、会话、队列与分布式锁

Redis 是内存中的键值数据库。对于大模型服务，它常保存热点结果、限制请求、保存登录态、派发异步任务，或让多个实例在短时间内协调。学习 Redis 的重点不是背命令，而是回答四个问题：数据放在哪个键里、多久过期、并发时是否原子、Redis 失败时服务如何降级。

> Redis 通常不是业务事实的唯一来源。订单、用户资料、支付结果等关键数据仍要写入主数据库；Redis 中的数据可能因 TTL、逐出、重启或故障切换而消失。

## 1. 学习目标与场景地图

完成本文后，你应能：

- 用 Docker 启动 Redis，使用 `redis-cli` 观察键和过期。
- 用 `redis-py` 操作字符串、哈希、列表、集合和有序集合。
- 实现 cache-aside 缓存、固定/滑动窗口限流、Redis 会话。
- 理解 List 队列和 Streams 消费组，并处理重复投递。
- 用 `SET NX PX` 加锁，以唯一令牌安全释放锁。
- 识别缓存穿透/击穿/雪崩、队列积压和锁过期。

| 场景 | 常用结构 | 最关键的机制 | 常见错误 |
| --- | --- | --- | --- |
| 缓存 | String / Hash | TTL、回源 | 把 Redis 当唯一真相 |
| 限流 | String / ZSET | 原子递增/脚本 | 在 Python 里拼多条非原子命令 |
| 会话 | String / Hash | 随机 ID、TTL、注销 | Cookie 直接放敏感会话数据 |
| 队列 | List / Stream | 阻塞读取、确认、重试 | 处理成功前确认消息 |
| 分布式锁 | String | 租约、唯一 token | 不校验 token 就删锁 |

## 2. 键设计和本地环境

键名推荐 `系统:环境:实体:标识:用途`：

```text
llm:prod:cache:answer:7f0c...
llm:prod:rate:tenant:42:202608171430
llm:prod:session:3f3f...
llm:prod:stream:embedding
llm:prod:lock:document:901
```

键名要包含业务前缀，便于监控和清理；长输入先规范化并哈希，不直接拼入键名。临时数据必须设置 TTL。生产排查使用 `SCAN`，不要执行会阻塞服务器的 `KEYS *`。

### 2.1 启动学习环境

```bash
docker run --name redis-study -p 6379:6379 -d redis:7-alpine \
  redis-server --appendonly yes

docker exec -it redis-study redis-cli
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SET hello world EX 60
OK
127.0.0.1:6379> TTL hello
(integer) 56
```

`EX 60` 表示 60 秒后过期。`TTL` 返回 `-1` 表示没有过期时间，返回 `-2` 表示键不存在。

### 2.2 Python 连接

```bash
python -m pip install "redis>=5,<7"
```

```python
import redis

redis_client = redis.Redis(
    host="127.0.0.1",
    port=6379,
    db=0,
    decode_responses=True,
    socket_connect_timeout=1,
    socket_timeout=1,
)

assert redis_client.ping()
```

`decode_responses=True` 会把 Redis 的字节串转为 Python `str`。一个 Web 服务应复用一个客户端实例，不要每个请求都新建连接。线上地址、密码、TLS 配置应放在环境变量或密钥系统中。

### 2.3 五种结构速览

```python
# String：缓存 JSON、计数器、锁。
redis_client.set("demo:name", "Vicky", ex=60)
redis_client.incr("demo:count")

# Hash：对象字段，会话元数据。
redis_client.hset("demo:user:1", mapping={"name": "Vicky", "role": "student"})

# List：简单先进先出队列。
redis_client.lpush("demo:jobs", "job-1")
redis_client.brpop("demo:jobs", timeout=5)

# Set：不重复标签、去重。
redis_client.sadd("demo:tags", "redis", "python")

# ZSET：按分数排序，常把时间戳作为分数。
redis_client.zadd("demo:requests", {"request-id": 1723876200})
```

## 3. 缓存：从 cache-aside 模式开始

读流程：先查 Redis，命中就返回；未命中查询主库/下游服务，写入 Redis 后返回。写流程：先更新主库，再删除缓存。删除比立即更新缓存更简单，下一次读取自然回填；它是最终一致，不是强一致。

```python
import json
from collections.abc import Callable
from typing import Any


def get_or_set_json(
    key: str,
    loader: Callable[[], dict[str, Any]],
    ttl_seconds: int = 300,
) -> dict[str, Any]:
    """按 cache-aside 读取 JSON 缓存，未命中时回源并回填。"""
    cached = redis_client.get(key)
    if cached is not None:
        return json.loads(cached)

    value = loader()
    redis_client.set(key, json.dumps(value, ensure_ascii=False), ex=ttl_seconds)
    return value


def load_user_from_database() -> dict[str, Any]:
    """模拟可信主数据源。"""
    return {"id": 42, "name": "Vicky", "plan": "pro"}


user = get_or_set_json("app:dev:cache:user:42", load_user_from_database)
```

更新时必须先提交主库再删缓存。若先删缓存、后更新主库，并发读可能在中间读到旧主库并把旧值重新缓存。

```python
def update_user_name(user_id: int, new_name: str) -> None:
    """提交主库更新后删除对应缓存。"""
    # database.execute("UPDATE users SET name = ? WHERE id = ?", (new_name, user_id))
    redis_client.delete(f"app:dev:cache:user:{user_id}")
```

### 3.1 缓存键必须包含全部影响结果的参数

LLM 回答缓存至少应区分：规范化问题、模型版本、系统提示词版本、知识库版本、用户或租户权限范围。漏掉权限范围会把一个用户的结果给另一个用户，是数据泄露。

### 3.2 三类缓存事故

**缓存穿透**：用户不断查询不存在的数据，每次都打到主库。可短暂缓存明确的“不存在”，同时做参数校验、认证和限流。

```python
NOT_FOUND = "__not_found__"


def get_user(user_id: int) -> dict[str, Any] | None:
    key = f"app:dev:cache:user:{user_id}"
    cached = redis_client.get(key)
    if cached == NOT_FOUND:
        return None
    if cached is not None:
        return json.loads(cached)

    user = None  # 使用真实主库查询替换。
    if user is None:
        redis_client.set(key, NOT_FOUND, ex=30)
        return None

    redis_client.set(key, json.dumps(user), ex=300)
    return user
```

**缓存击穿**：热点键过期时，许多请求同时回源。先给 TTL 加随机抖动，必要时再实现“单飞”回源；不要一开始就用分布式锁包住所有缓存读取。

```python
import random

redis_client.set("app:dev:cache:hot:topic", "...", ex=300 + random.randint(0, 60))
```

**缓存雪崩**：大量键同时过期或 Redis 不可用。需要短超时、回源并发控制、降级和限流。对 LLM 服务，缓存故障时更要保护模型服务，不能让全部流量直接回源。

应监控命中/未命中、回源耗时、缓存错误、内存、逐出数和键大小。高命中率并不自动代表正确，权限键设计同样重要。

## 4. 限流：保护共享资源

限流常按用户、租户、IP、模型或 token 用量设置。限流维度必须符合资源边界：模型 API 通常按租户；登录接口可能按 `IP + 账号`。超过配额时返回 HTTP `429 Too Many Requests`，并附 `Retry-After`。

### 4.1 固定窗口

每分钟允许 60 次：把“用户 ID + 当前分钟”作为键。`INCR` 是原子操作；首次设置 TTL 也必须可靠完成。

```python
import time


def allow_fixed_window(user_id: int, limit: int = 60, window_seconds: int = 60) -> bool:
    """判断用户在当前固定窗口内是否可继续请求。"""
    bucket = int(time.time() // window_seconds)
    key = f"app:dev:rate:user:{user_id}:{bucket}"

    with redis_client.pipeline() as pipe:
        pipe.incr(key)
        pipe.expire(key, window_seconds + 1, nx=True)
        count, _ = pipe.execute()

    return count <= limit
```

固定窗口会出现边界突发：用户在 `12:00:59` 发满 60 次，又在 `12:01:00` 发满 60 次，1 秒内通过 120 次。普通接口可接受；需要平滑限制时使用滑动窗口或令牌桶。

### 4.2 滑动窗口：ZSET 加 Lua 脚本

Lua 让“删除窗口外记录、计数、加入本次请求、设置过期”在 Redis 内一次完成，避免多条 Python 命令被其他请求穿插。

```python
import time
import uuid

SLIDING_WINDOW_LUA = """
local key = KEYS[1]
local now_ms = tonumber(ARGV[1])
local window_ms = tonumber(ARGV[2])
local limit = tonumber(ARGV[3])
local member = ARGV[4]

redis.call('ZREMRANGEBYSCORE', key, 0, now_ms - window_ms)
if redis.call('ZCARD', key) >= limit then
    return 0
end
redis.call('ZADD', key, now_ms, member)
redis.call('PEXPIRE', key, window_ms)
return 1
"""

sliding_window = redis_client.register_script(SLIDING_WINDOW_LUA)


def allow_sliding_window(user_id: int, limit: int = 60, window_seconds: int = 60) -> bool:
    """按精确滑动窗口限制请求数。"""
    result = sliding_window(
        keys=[f"app:dev:rate:user:{user_id}:sliding"],
        args=[int(time.time() * 1000), window_seconds * 1000, limit, str(uuid.uuid4())],
    )
    return bool(result)
```

滑动窗口更精确，但每次请求都保存一条记录，高频用户占用更多内存。验证必须覆盖：阈值内允许、恰好到阈值、超出拒绝、窗口后恢复、用户间隔离、Redis 故障时的策略。安全敏感接口在 Redis 故障时通常偏拒绝；普通查询可以短暂放行但要保护下游。这是风险决策，不能机械套用。

## 5. 会话：服务端状态与 Cookie 凭据分离

HTTP 无状态。浏览器 Cookie 应仅保存高熵、不可猜测的 `session_id`；Redis 保存用户 ID、角色和签发时间，并用 TTL 控制会话生命周期。

```text
Cookie: session_id=随机值
Redis: app:prod:session:<session_id> -> JSON 会话内容 + TTL
```

```python
import json
import secrets
from typing import Any

SESSION_TTL_SECONDS = 60 * 60 * 24 * 7


def create_session(user_id: int, roles: list[str]) -> str:
    """创建随机会话 ID，并持久化带 TTL 的会话内容。"""
    session_id = secrets.token_urlsafe(32)
    payload: dict[str, Any] = {"user_id": user_id, "roles": roles}
    redis_client.set(
        f"app:dev:session:{session_id}",
        json.dumps(payload),
        ex=SESSION_TTL_SECONDS,
    )
    return session_id


def get_session(session_id: str) -> dict[str, Any] | None:
    """读取会话，已过期或不存在时返回 None。"""
    raw = redis_client.get(f"app:dev:session:{session_id}")
    return json.loads(raw) if raw is not None else None


def logout(session_id: str) -> None:
    """删除服务端会话，使旧凭据立即失效。"""
    redis_client.delete(f"app:dev:session:{session_id}")
```

Cookie 要设置 `HttpOnly`、`Secure` 和恰当的 `SameSite`。`HttpOnly` 阻止前端 JavaScript 读取，`Secure` 只允许 HTTPS 传输。若每次访问都延长 TTL（滑动过期），还要设置绝对最长登录时间，否则活跃会话可能永不过期。

改密码或权限变更时，删除所有会话或提升 `session_version`；不要只等 TTL。Redis 查会话失败应被视为未认证，而不是已认证。多实例服务把会话放在 Redis 后，任意实例都能验证用户，这正是它的价值。

## 6. 队列：把慢任务移出 HTTP 请求

文档解析、向量化、批量评测等任务耗时很长。接口只负责发布任务并返回 `job_id`；后台 worker 消费任务，成功后更新状态。这样请求不会一直占连接。

### 6.1 List：简单队列

`BRPOP` 在队列为空时阻塞等待，避免 worker 忙循环。

```python
import json
import time
import uuid

QUEUE_KEY = "app:dev:queue:email"


def enqueue_email(to: str, subject: str) -> None:
    """把邮件任务写入队列。"""
    job = {"id": str(uuid.uuid4()), "to": to, "subject": subject}
    redis_client.lpush(QUEUE_KEY, json.dumps(job))


def worker_forever() -> None:
    """阻塞消费任务；真实项目须加重试、日志和优雅退出。"""
    while True:
        result = redis_client.brpop(QUEUE_KEY, timeout=5)
        if result is None:
            continue
        _, raw_job = result
        job = json.loads(raw_job)
        print(f"sending to {job['to']}")
        time.sleep(1)
```

List 的问题是 worker 取走消息后、完成前崩溃时，消息可能丢失。可用 `BRPOPLPUSH` 移到“处理中”列表，成功后删除；但可靠重试、超时认领和可观测性会迅速复杂。可靠场景应了解 Streams。

### 6.2 Streams：消费组与确认

Streams 是追加日志。消费者组让多个 worker 分摊工作；成功后才 `XACK`，崩溃前未确认的消息会留在待处理列表（PEL）中。

```python
STREAM = "app:dev:stream:embedding"
GROUP = "embedding-workers"


def create_group() -> None:
    """创建消费者组；流不存在时一起创建。"""
    try:
        redis_client.xgroup_create(STREAM, GROUP, id="0", mkstream=True)
    except redis.ResponseError as exc:
        if "BUSYGROUP" not in str(exc):
            raise


def publish_embedding_job(document_id: int) -> str:
    """发布向量化任务并返回消息 ID。"""
    return redis_client.xadd(STREAM, {"document_id": str(document_id)})


def consume_once(consumer_name: str) -> None:
    """读取一条消息，业务成功后才确认。"""
    messages = redis_client.xreadgroup(
        groupname=GROUP,
        consumername=consumer_name,
        streams={STREAM: ">"},
        count=1,
        block=5000,
    )
    for _, entries in messages:
        for message_id, fields in entries:
            try:
                document_id = int(fields["document_id"])
                print(f"embedded document {document_id}")
                # build_embedding(document_id)
            except Exception:
                # 不确认：消息留在 PEL，之后重试或人工处理。
                raise
            else:
                redis_client.xack(STREAM, GROUP, message_id)
```

Streams 常提供**至少一次投递**：worker 做完业务动作、但 `XACK` 前崩溃时，同一消息可能再次投递。因此业务必须幂等：使用稳定 `job_id`，在主库加唯一约束或状态机；第三方调用使用幂等键；失败任务记录次数并转死信队列。还要用 `XPENDING` 监控 PEL，并通过 `XAUTOCLAIM`/`XCLAIM` 处理长期未确认消息。

| 需求 | 建议 |
| --- | --- |
| 本地练习、可快速完成任务 | List + `BRPOP` |
| Redis 内的消费者组、确认、重放 | Streams |
| 复杂定时、重试与任务运维 | Celery / Dramatiq 等框架 |
| 高吞吐持久消息、跨系统事件 | Kafka / RabbitMQ 等专用系统 |

## 7. 分布式锁：短互斥，不是事务替代

多个实例处理同一文档时，本地 `threading.Lock` 无效。Redis 锁适合短时间协调，例如“同一文档一次只允许一个 worker 重建索引”。它不替代数据库事务；若任务超出租约时间，另一个实例可能获得锁，旧实例仍可能继续执行。

### 7.1 获取锁

`SET key token NX PX ttl` 是原子命令：仅键不存在时设置，并设置毫秒级租约。token 必须随机且唯一。

```python
import secrets


def acquire_lock(key: str, ttl_ms: int) -> str | None:
    """获取短租约锁，成功返回唯一 token，失败返回 None。"""
    token = secrets.token_urlsafe(24)
    return token if redis_client.set(key, token, nx=True, px=ttl_ms) else None
```

### 7.2 只释放自己的锁

下面是错误示例：锁可能已过期并被其他实例重新获得，旧实例会误删新锁。

```python
# 错误：禁止这样做。
if redis_client.get("app:dev:lock:document:901"):
    redis_client.delete("app:dev:lock:document:901")
```

正确做法是在 Redis 内原子比较 token 后再删除：

```python
RELEASE_LOCK_LUA = """
if redis.call('GET', KEYS[1]) == ARGV[1] then
    return redis.call('DEL', KEYS[1])
end
return 0
"""
release_lock = redis_client.register_script(RELEASE_LOCK_LUA)


def release_own_lock(key: str, token: str) -> bool:
    """仅在 token 仍对应当前锁时释放。"""
    return bool(release_lock(keys=[key], args=[token]))
```

```python
import time


def rebuild_document_once(document_id: int) -> bool:
    """避免同一文档被多个 worker 同时重建。"""
    key = f"app:dev:lock:document:{document_id}"
    token = acquire_lock(key, ttl_ms=10_000)
    if token is None:
        return False
    try:
        time.sleep(1)  # 真实临界区必须明显短于 TTL。
        print(f"rebuilt document {document_id}")
        return True
    finally:
        release_own_lock(key, token)
```

锁 TTL 太短会并发执行，太长会在崩溃后阻塞任务。获取不到锁时不要忙等，应退避、限制重试或返回“处理中”。对于会写外部资源的高风险流程，进一步使用数据库条件更新、版本号或 fencing token 拒绝旧锁持有者的写入；只靠 Redis 锁无法保证绝对安全。

## 8. 可靠性、安全与排错

### 8.1 明确 Redis 不可用时的行为

- 缓存读取失败：短超时后回源，同时限制回源并发。
- 会话读取失败：安全地拒绝认证或要求重新登录，不能默认通过。
- 限流读取失败：安全敏感接口偏拒绝；普通接口可短暂放行但要保护下游。
- 队列故障：生产者返回可重试错误；worker 退避、告警，不能静默丢任务。
- 锁故障：不得假装已获得锁。

Redis 超时通常应很短。无边界自动重试会在故障时制造请求风暴。所有关键业务写入都要幂等，因为网络超时、队列重投和客户端重试都会造成重复执行。

### 8.2 内存、持久化与安全

Redis 达到 `maxmemory` 后会拒绝写入或按 `maxmemory-policy` 逐出键。缓存可容忍逐出，但会话、限流和队列被逐出会改变安全/正确性；应按业务隔离实例或最少隔离监控与容量策略。RDB/AOF 能降低丢失风险，却不等于零丢失。

Redis 不应暴露公网；生产启用认证，必要时 TLS 和 ACL，按服务最小授权。不要把连接串、密码、会话 ID 和完整 Prompt 写入日志。限制危险运维命令，备份与监控也需要访问控制。

### 8.3 常用排错命令

```bash
redis-cli PING
redis-cli --scan --pattern 'app:dev:*'
redis-cli TTL app:dev:session:some-id
redis-cli XLEN app:dev:stream:embedding
redis-cli XPENDING app:dev:stream:embedding embedding-workers
redis-cli SLOWLOG GET 10
```

至少监控：连接数、命令延迟、内存、逐出键、命中/未命中、缓存错误、队列长度、PEL、重试/死信数量与锁竞争率。指标要配合告警阈值和处置步骤。

## 9. 测试、练习与上线检查

测试不只验证正常返回，还要验证并发和失败：两个线程/进程争锁、缓存刚过期时并发回源、worker 在 `XACK` 前崩溃、Redis 连接超时后的降级。以下是最小限流测试：

```python
def test_fixed_window_allows_until_limit() -> None:
    user_id = 999_001
    for key in redis_client.scan_iter(match=f"app:dev:rate:user:{user_id}:*"):
        redis_client.delete(key)

    assert allow_fixed_window(user_id, limit=2) is True
    assert allow_fixed_window(user_id, limit=2) is True
    assert allow_fixed_window(user_id, limit=2) is False
```

建议按顺序完成五个小项目：

1. 为模拟 `POST /chat` 做回答缓存，并把模型/Prompt/权限版本放入键。
2. 为每个租户做每分钟 20 次的固定窗口限流，再升级为滑动窗口。
3. 实现登录、当前用户、退出登录；确认 Cookie 只有 session ID。
4. 上传文档只写入 Stream，worker 消费后更新任务状态；故意在确认前崩溃并实现重试。
5. 启动两个 worker 重建同一文档，观察 Redis 锁；故意让任务超过 TTL，理解为什么最终写入仍需版本保护。

上线或面试前，确认以下问题都有明确答案：

- 哪些 Redis 数据可丢，哪些有主库或可靠消息系统兜底？
- 每个临时键是否都有合理 TTL？
- 缓存键是否覆盖权限、版本和全部输入？
- 限流主体、窗口、资源指标和故障策略是否已定义？
- 队列是否成功后才确认？消费者是否幂等？失败任务去哪？
- 锁是否使用 `SET NX PX`、随机 token、原子比较释放？
- Redis 重启、逐出、超时和故障切换时，服务具体会返回什么？

Redis 的价值不在于命令数量，而在于你能为共享状态定义过期、并发语义和失败路径。把这五种场景做成可测试的小项目，你就具备了大模型服务后端的关键基础。