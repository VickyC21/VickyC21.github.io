---
layout: post
title: "Python 后端数据库入门：PostgreSQL 与 MySQL 的表设计、索引、事务和查询优化"
date: 2026-08-17 11:00:00 +0800
categories: [Python基础]
tags: [Python, PostgreSQL, MySQL, SQL, 数据库, 索引, 事务, 查询优化]
---

# Python 后端数据库入门：PostgreSQL 与 MySQL 的表设计、索引、事务和查询优化

这份讲义面向 Python 后端初学者。目标不是背 SQL，而是能为一个真实服务设计数据表、写出正确查询、理解并发下的数据一致性，并能用执行计划定位慢查询。

本文使用一个简化的 LLM 应用：用户创建会话，会话保存消息，每次模型调用产生一条用量记录。这个例子可覆盖表设计、索引、事务和查询优化四个核心主题。

> SQL 以 PostgreSQL 16 为主；MySQL 8.0 的关键差异单独标注。先掌握共同原理，再记忆差异。

## 1. 总览：数据库要解决什么

| 问题 | 要回答什么 | 初学者常见错误 |
| --- | --- | --- |
| 表设计 | 数据是什么，如何关联，什么值合法 | 缺约束、重复字段、把关系塞进字符串 |
| 索引 | 哪些查询必须快速定位、排序或连接 | 给每列建索引，或索引顺序与查询不匹配 |
| 事务 | 多步修改如何全部成功或全部失败 | 先查后改、长事务、忽略并发冲突 |
| 优化 | SQL 为什么慢，修改后是否真的更快 | 凭感觉加索引，不看执行计划 |

数据库不是“帮 Python 存字典”。它负责持久化、约束、并发控制与高效集合查询；应用代码表达业务意图，数据库负责守住数据规则。

## 2. 基本概念

~~~sql
CREATE TABLE app_user (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email TEXT NOT NULL UNIQUE,
    display_name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
~~~

- **表**：同类实体集合，例如 app_user。
- **行**：一个具体用户；**列**：用户的一项属性。
- **主键**：稳定且唯一标识一行，通常是 id。
- **外键**：引用另一张表的主键，表达关系。
- **约束**：数据库自动执行的规则，例如非空、唯一和范围。
- **索引**：用额外空间换取部分查询速度。
- **事务**：要么全部成功、要么全部失败的一组操作。

创建字段前先问：它是单值属性还是独立实体？会被过滤、排序、连接吗？必须存在或唯一吗？删除父记录时如何处理关联记录？这些问题比字段命名更重要。

## 3. 关系建模与表设计

业务关系：

~~~text
app_user 1 --- N conversation 1 --- N message
app_user 1 --- N model_usage
message  0 --- N model_usage
~~~

### 3.1 一对多：外键放在“多”的一侧

~~~sql
CREATE TABLE conversation (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES app_user(id) ON DELETE CASCADE,
    title TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
~~~

conversation.user_id 保证会话指向真实用户。ON DELETE CASCADE 表示删除用户时删除会话；若数据需审计，使用 RESTRICT 或逻辑删除更合适。

> MySQL 用 BIGINT AUTO_INCREMENT 代替 identity。MySQL 没有 TIMESTAMPTZ，通常统一存 UTC TIMESTAMP，并在应用层明确时区。

### 3.2 多对多：使用中间表

~~~sql
CREATE TABLE tag (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL UNIQUE
);

CREATE TABLE conversation_tag (
    conversation_id BIGINT NOT NULL REFERENCES conversation(id) ON DELETE CASCADE,
    tag_id BIGINT NOT NULL REFERENCES tag(id) ON DELETE CASCADE,
    PRIMARY KEY (conversation_id, tag_id)
);
~~~

不要把多个 ID 存为 "1,8,12"：无法用外键验证、难以查询、易产生重复和脏数据。中间表还可保存“谁添加标签”“置信度”等关联属性。

### 3.3 类型、NULL 和约束

| 数据 | PostgreSQL | MySQL | 原则 |
| --- | --- | --- | --- |
| 主键 | BIGINT | BIGINT | 配合 identity/auto increment |
| 金额 | NUMERIC(12, 2) | DECIMAL(12, 2) | 不用 FLOAT |
| 时间点 | TIMESTAMPTZ | UTC TIMESTAMP | 全服务统一 UTC |
| 布尔值 | BOOLEAN | BOOLEAN/TINYINT(1) | 不要让 0、1、2 表示多状态 |
| 扩展字段 | JSONB | JSON | 不代替核心关系 |

NULL 是“未知、未提供或不适用”，不是空字符串或 0。SQL 是三值逻辑：

~~~sql
SELECT * FROM message WHERE deleted_at = NULL;  -- 错误
SELECT * FROM message WHERE deleted_at IS NULL; -- 正确
~~~

关键规则必须由数据库保护，而不只写在 Python 中：

~~~sql
CREATE TABLE message (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    conversation_id BIGINT NOT NULL
        REFERENCES conversation(id) ON DELETE CASCADE,
    role TEXT NOT NULL CHECK (role IN ('system', 'user', 'assistant', 'tool')),
    content TEXT NOT NULL,
    sequence_no INTEGER NOT NULL CHECK (sequence_no >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE (conversation_id, sequence_no)
);

CREATE TABLE model_usage (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES app_user(id) ON DELETE RESTRICT,
    message_id BIGINT REFERENCES message(id) ON DELETE SET NULL,
    model_name TEXT NOT NULL,
    prompt_tokens INTEGER NOT NULL CHECK (prompt_tokens >= 0),
    completion_tokens INTEGER NOT NULL CHECK (completion_tokens >= 0),
    cost_usd NUMERIC(12, 6) NOT NULL CHECK (cost_usd >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
~~~

NOT NULL 表示必填；UNIQUE 防重复；CHECK 约束范围；FOREIGN KEY 保护引用完整性。MySQL 8.0 支持 CHECK，旧版本可能不执行，必须确认版本。

### 3.4 范式与反范式

范式的目标是让“一个事实只存一处”。

- 第一范式：不在一列存可变列表，例如标签建中间表。
- 第二范式：联合键表的非键列依赖整个键。
- 第三范式：非键列不依赖另一个非键列。例如会话只保存 user_id，不复制 user_email。

只有通过指标证明连接或聚合是瓶颈，并能保证同步时，才反范式化。例如缓存会话最后消息时间；写入消息时必须在同一事务更新它。

## 4. 正确执行 SQL

绝不拼接 SQL：

~~~python
# 错误：会产生注入和引号问题
sql = f"SELECT * FROM app_user WHERE email = '{email}'"
~~~

使用驱动参数绑定。以 psycopg 为例：

~~~python
from psycopg import connect


def find_user_by_email(dsn: str, email: str) -> tuple | None:
    """按邮箱查询用户，并由驱动安全绑定参数。

    Args:
        dsn: PostgreSQL 连接字符串。
        email: 待查询邮箱。

    Returns:
        匹配到的用户行；不存在时返回 None。
    """
    with connect(dsn) as conn:
        with conn.cursor() as cur:
            cur.execute(
                "SELECT id, email, display_name FROM app_user WHERE email = %s",
                (email,),
            )
            return cur.fetchone()
~~~

%s 是驱动占位符，不是 Python 格式化。MySQL 的常见驱动也使用 %s，但应以具体驱动文档为准。

PostgreSQL 用 RETURNING 获取新 ID：

~~~sql
INSERT INTO conversation (user_id, title)
VALUES (42, '数据库学习计划')
RETURNING id;
~~~

MySQL 一般读驱动的 lastrowid。不要查询 MAX(id)，并发下会拿到别人的 ID。

更新和删除应携带权限边界，并检查影响行数：

~~~sql
UPDATE conversation
SET title = '新的标题',
    updated_at = CURRENT_TIMESTAMP
WHERE id = 1001
  AND user_id = 42;
~~~

## 5. 索引：从真实查询出发

最常用 B-tree 索引适合等值、范围、排序与部分前缀匹配。它有成本：占空间，写入时需维护；低区分度列或返回绝大多数行的查询未必受益。

外键保证正确性，但不一定自动创建查询索引：

~~~sql
SELECT id, title, updated_at
FROM conversation
WHERE user_id = 42
ORDER BY updated_at DESC
LIMIT 20;

SELECT id, role, content
FROM message
WHERE conversation_id = 1001
ORDER BY sequence_no;
~~~

~~~sql
CREATE INDEX idx_conversation_user_updated
ON conversation (user_id, updated_at DESC);

CREATE INDEX idx_message_conversation_sequence
ON message (conversation_id, sequence_no);
~~~

联合索引 (user_id, updated_at) 先按 user_id，再按 updated_at 排序，适合按用户过滤后按时间排序，也适合只按 user_id；通常不适合只按 updated_at。记忆法：先按姓排列的电话簿，不能高效只按名查找。

常见索引类型：

- UNIQUE：规则和查询兼顾；UNIQUE 约束通常已创建索引，不要重复建。
- 部分索引：PostgreSQL 可只索引活跃行，例如 WHERE deleted_at IS NULL；MySQL 没有同等直接语法。
- 覆盖索引：PostgreSQL 可用 INCLUDE 放入少量返回列；不要把大文本列塞进去。

索引不生效的典型原因：返回行比例太大、列上套函数、隐式类型转换、统计信息过期。把函数条件改为范围条件：

~~~sql
-- 不推荐
WHERE DATE(created_at) = DATE '2026-08-17'

-- 推荐
WHERE created_at >= TIMESTAMPTZ '2026-08-17 00:00:00+00'
  AND created_at < TIMESTAMPTZ '2026-08-18 00:00:00+00'
~~~

## 6. 用执行计划优化，而不是猜

PostgreSQL：

~~~sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title, updated_at
FROM conversation
WHERE user_id = 42
ORDER BY updated_at DESC
LIMIT 20;
~~~

MySQL 8.0：

~~~sql
EXPLAIN ANALYZE
SELECT id, title, updated_at
FROM conversation
WHERE user_id = 42
ORDER BY updated_at DESC
LIMIT 20;
~~~

| 节点 | 含义 | 何时正常 |
| --- | --- | --- |
| Seq Scan | 顺序扫描 | 小表或返回大比例数据 |
| Index Scan | 索引定位后回表 | 需读取索引外的列 |
| Index Only Scan | 从索引直接获得大部分结果 | 仍受可见性信息影响 |
| Sort | 显式排序 | 索引不能提供目标顺序 |
| Nested Loop | 外层逐行驱动内层查询 | 外层小且内层有索引 |
| Hash Join | 建哈希表后连接 | 两边数据较多 |
| Aggregate | COUNT、SUM 等聚合 | 关注输入行数和耗时 |

重点看估计行数与实际行数是否接近、哪个节点耗时最高、loops 是否异常大、是否大量读盘。MySQL 额外关注 key、rows、type、Extra。全表扫描或 filesort 并非天然错误，结论应基于数据量和总耗时。

优化流程：

1. 确认慢的是哪条 SQL，而不是外部 API 或序列化。
2. 记录 SQL 模板、参数类型、调用频率、返回行数和 P95。
3. 用执行计划提出一个假设。
4. 只做一个最小改动，例如一个匹配索引。
5. 用近似数据量复测耗时、扫描行数、结果正确性和写入代价。

## 7. 事务与并发

ACID：

- Atomicity：一组操作全部提交或全部回滚。
- Consistency：提交前后满足约束与业务规则。
- Isolation：并发事务不以不可接受方式干扰。
- Durability：提交成功后故障也能恢复。

保存一轮对话时，用户消息、助手消息、用量记录与会话更新时间必须一起成功。任一步失败都应回滚，不能留下半成品。

~~~python
from decimal import Decimal

from psycopg import connect


def save_usage(
    dsn: str,
    user_id: int,
    message_id: int,
    model_name: str,
    prompt_tokens: int,
    completion_tokens: int,
    cost_usd: Decimal,
) -> None:
    """在同一事务中写入模型用量。

    Args:
        dsn: PostgreSQL 连接字符串。
        user_id: 当前用户 ID。
        message_id: 助手消息 ID。
        model_name: 实际调用模型名。
        prompt_tokens: 输入 token 数。
        completion_tokens: 输出 token 数。
        cost_usd: 调用费用，使用 Decimal 避免浮点误差。
    """
    with connect(dsn) as conn:
        with conn.cursor() as cur:
            cur.execute(
                """
                INSERT INTO model_usage (
                    user_id, message_id, model_name,
                    prompt_tokens, completion_tokens, cost_usd
                )
                VALUES (%s, %s, %s, %s, %s, %s)
                """,
                (
                    user_id,
                    message_id,
                    model_name,
                    prompt_tokens,
                    completion_tokens,
                    cost_usd,
                ),
            )
~~~

成功离开上下文时提交，发生异常时回滚。事务中不要调用 LLM API、HTTP 或处理大文件，否则持锁过久、连接池被占满。

PostgreSQL 默认 READ COMMITTED；MySQL InnoDB 默认 REPEATABLE READ。不要假设不同数据库的隔离行为相同。

### 7.1 用条件 UPDATE 防止超扣

错误方式是“先 SELECT 余额，再让 Python 判断，再 UPDATE”，两个并发请求可能都看到相同余额。把检查和修改放入一条 SQL：

~~~sql
UPDATE account
SET remaining_credits = remaining_credits - 1
WHERE user_id = 42
  AND remaining_credits >= 1
RETURNING remaining_credits;
~~~

返回一行表示成功；返回零行表示不存在或额度不足。这是一个原子动作。

### 7.2 行锁、死锁与重试

必须基于当前值做多步决策时使用行锁：

~~~sql
BEGIN;

SELECT remaining_credits
FROM account
WHERE user_id = 42
FOR UPDATE;

-- 在同一短事务内检查并更新

COMMIT;
~~~

FOR UPDATE 会让竞争同一行的事务等待。多表操作必须统一加锁顺序，例如总是先锁小 ID。死锁发生时数据库会回滚一个事务；对明确可重试的死锁或序列化失败进行有限重试并记录次数，绝不无限重试。

## 8. 常见查询陷阱

### 8.1 少取列、避免深 OFFSET

~~~sql
SELECT id, role, content, created_at
FROM message
WHERE conversation_id = 1001
ORDER BY sequence_no;
~~~

不要随手 SELECT *。它会扩大传输、带出敏感列，也会让新增大字段拖慢接口。

深 OFFSET 会越来越慢：

~~~sql
SELECT id, title, updated_at
FROM conversation
WHERE user_id = 42
ORDER BY updated_at DESC
LIMIT 20 OFFSET 10000;
~~~

使用键集分页：

~~~sql
SELECT id, title, updated_at
FROM conversation
WHERE user_id = 42
  AND (updated_at, id) < (
      TIMESTAMPTZ '2026-08-17 10:30:00+00',
      8001
  )
ORDER BY updated_at DESC, id DESC
LIMIT 20;

CREATE INDEX idx_conversation_cursor
ON conversation (user_id, updated_at DESC, id DESC);
~~~

把最后一行的 (updated_at, id) 编码为不透明 cursor；id 作为并列排序键，避免跳页时重复或漏项。

### 8.2 N+1、JOIN 与聚合

错误流程：查 20 个会话，再循环查每个会话最后消息，共 21 条 SQL。改为批量查询。PostgreSQL 示例：

~~~sql
SELECT DISTINCT ON (m.conversation_id)
    m.conversation_id, m.id, m.role, m.content, m.created_at
FROM message AS m
WHERE m.conversation_id = ANY(%s)
ORDER BY m.conversation_id, m.sequence_no DESC;
~~~

跨数据库可用窗口函数 ROW_NUMBER() OVER (PARTITION BY conversation_id ORDER BY sequence_no DESC)。

JOIN 不天然慢；缺索引、海量返回、无界排序和 N+1 才常是问题。聚合也应在数据库做：

~~~sql
SELECT model_name,
       COUNT(*) AS request_count,
       SUM(prompt_tokens + completion_tokens) AS total_tokens,
       SUM(cost_usd) AS total_cost
FROM model_usage
WHERE user_id = 42
  AND created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY model_name
ORDER BY total_cost DESC;
~~~

不要把几十万条记录取回 Python 再求和。

## 9. 连接池与 PostgreSQL/MySQL 差异

连接有握手、认证与资源成本。Web 服务应在启动时创建有限连接池；请求借用连接，执行短事务后立即归还。连接数并非越大越好，慢查询、长事务和泄漏会耗尽连接池。同步 PostgreSQL 常用 psycopg_pool；异步服务使用异步连接池或 SQLAlchemy 异步引擎。ORM 不会自动解决索引、N+1 和事务边界。

| 主题 | PostgreSQL | MySQL 8.0 / InnoDB |
| --- | --- | --- |
| 默认隔离级别 | READ COMMITTED | REPEATABLE READ |
| 自增 ID | IDENTITY / sequence | AUTO_INCREMENT |
| 时间 | TIMESTAMPTZ | 常用 UTC TIMESTAMP |
| JSON | JSONB 功能强、索引丰富 | JSON |
| 部分索引 | 原生支持 WHERE 条件 | 无同等直接语法 |

优先使用团队既有数据库。新项目若需要复杂查询、强约束、丰富 JSON 与扩展能力，PostgreSQL 往往是好默认；团队基础设施围绕 MySQL 时，MySQL 同样可承担常见业务。

## 10. 四周学习计划与速查

- **第一周**：安装 PostgreSQL；创建本文四张表；故意插入重复邮箱、负 token、无效外键，观察约束。
- **第二周**：导入十万条模拟数据；为会话列表、消息列表和月度用量写查询、索引与 EXPLAIN ANALYZE。
- **第三周**：实现“扣积分并写用量”事务；用两个并发请求复现超扣，再改为条件 UPDATE；练习死锁和统一锁顺序。
- **第四周**：接入 FastAPI 或 Flask 连接池；所有 SQL 参数化；为越权、重复写、额度不足、事务失败和并发写补测试。

~~~text
表设计：主键 + 外键 + NOT NULL/UNIQUE/CHECK，先保证正确。
索引：从真实 WHERE、JOIN、ORDER BY 设计，联合索引顺序很重要。
查询：参数化、少取列、避免 N+1，深分页使用 cursor。
事务：必须一起成功的写入放短事务；并发用原子更新/锁/有限重试。
优化：先量化，再 EXPLAIN ANALYZE，再做最小改动并复测。
Python：连接池借还连接；异常回滚；绝不拼接 SQL。
~~~

完成后请能回答：为什么多对多需要中间表？为什么外键不能替代索引？为什么索引 (a, b) 通常不能只服务 b？如何防止并发超卖？如何证明 SQL 优化真的有效？能回答并在小项目实践，就具备了大模型后端工程师需要的数据库基础。