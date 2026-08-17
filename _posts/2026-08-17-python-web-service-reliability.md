---
layout: post
title: "Python Web 服务可靠性讲义：认证、授权、会话、限流、幂等、超时、重试、CORS 与错误码"
date: 2026-08-17 11:00:00 +0800
categories: [Python基础]
tags: [Python, FastAPI, Web开发, API设计, 安全, 可靠性]
---

# Python Web 服务可靠性讲义

> 面向刚开始做大模型应用或 Python 后端服务的学习者。示例使用 FastAPI，概念同样适用于 Flask、Django、RPC 服务与 LLM 工具服务。

## 1. 一次请求的完整边界

大模型服务不只是把用户输入交给模型。它还要面对不可信输入、身份与权限、重复提交、突发流量、不稳定的数据库和模型服务，以及浏览器跨域限制。

~~~text
客户端
  -> CORS 来源检查
  -> 限流与并发保护
  -> 认证：你是谁？
  -> 授权：你能对当前资源做这个动作吗？
  -> 幂等：是否是重复的写操作？
  -> 业务处理：数据库、检索、LLM、工具调用（带超时）
  -> 有条件重试
  -> 统一错误响应、trace_id、日志与指标
~~~

| 能力 | 需要回答的问题 |
| --- | --- |
| 认证 | 请求者是谁？ |
| 授权 | 该身份能否操作当前资源？ |
| 会话 | 多次请求如何保持同一登录状态？ |
| 限流 | 单位时间内可以消耗多少资源？ |
| 幂等 | 重复请求如何保证副作用只发生一次？ |
| 超时 | 等待下游多久后放弃？ |
| 重试 | 哪些失败值得再试一次？ |
| CORS | 哪些浏览器来源可读取 API 响应？ |
| 错误码 | 客户端与运维如何理解失败？ |

认证回答“你是谁”，授权回答“你能做什么”。每层都是边界：上游输入不可信，下游依赖不可靠，重复请求一定会发生。

## 2. 认证：确认请求者身份

认证成功后，服务端得到可信的最小身份上下文，例如用户 ID、租户 ID、token 有效期和权限范围。

| 方式 | 请求携带内容 | 常见场景 | 主要注意点 |
| --- | --- | --- | --- |
| 密码登录 | 用户名和密码 | 首次登录 | 密码专用慢哈希 |
| Cookie 会话 | 浏览器自动带 Cookie | 传统 Web 站点 | Cookie 属性与 CSRF |
| Bearer token / JWT | Authorization 请求头 | 前后端分离、移动端 | 签名、过期、吊销 |
| API Key | 请求头中的密钥 | 服务对服务 | 密钥泄露、最小权限 |
| OAuth 2.0 / OIDC | 授权码和 token | 企业 SSO | 回调、签发者、受众、scope |

### 2.1 密码哈希

不要存密码明文，也不要自行做 sha256(password)。密码应使用 Argon2 或 bcrypt 这类专用慢哈希算法，每个密码有独立盐值，并由成熟库负责验证。

~~~python
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()
stored_hash = password_hash.hash("correct horse battery staple")

assert password_hash.verify("correct horse battery staple", stored_hash)
assert not password_hash.verify("wrong password", stored_hash)
~~~

慢哈希的目的，是在数据库泄露后提高离线猜测密码的成本。日志中不得出现密码、密码哈希、完整 access token 或 refresh token。

### 2.2 JWT 的正确认知

JWT 的 payload 通常只是 Base64URL 编码，拿到 token 的人可以读取内容；签名用于防篡改，不用于保密。

短期 access token 可包含 sub（不可变用户 ID）、exp（过期时间）、iat（签发时间）、iss（签发者）、aud（目标受众）、少量稳定 scope 或角色，以及可选 jti（token 唯一 ID）。不要放密码、身份证号、完整资料、数据库配置，或高频变化的权限状态。

生产环境验证 JWT 时必须校验签名、算法白名单、exp、iss 与 aud；绝不能只解码 payload。

### 2.3 FastAPI 认证示例

~~~python
from typing import Annotated

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

app = FastAPI()
bearer_scheme = HTTPBearer(auto_error=False)


def decode_and_verify_access_token(token: str) -> dict[str, str]:
    # 教学示例使用固定 token。生产环境必须使用成熟 JWT 或 OIDC 组件。
    if token != "demo-token":
        raise ValueError("invalid token")
    return {"sub": "user-123", "tenant_id": "tenant-a"}


def get_current_user(
    credentials: Annotated[
        HTTPAuthorizationCredentials | None,
        Depends(bearer_scheme),
    ],
) -> dict[str, str]:
    if credentials is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="缺少访问凭据",
            headers={"WWW-Authenticate": "Bearer"},
        )

    try:
        return decode_and_verify_access_token(credentials.credentials)
    except ValueError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="访问凭据无效或已过期",
            headers={"WWW-Authenticate": "Bearer"},
        ) from exc
~~~

401 表示尚未通过认证，例如缺少、过期或无效 token；403 表示身份已确认，但无权执行当前操作。

## 3. 授权：确认你能做什么

前端隐藏删除按钮只能改善体验，攻击者仍可以直接构造 HTTP 请求。因此服务端必须在每个受保护动作上授权。

~~~python
# 危险：客户端传来的 role 不可信。
if request_body.role == "admin":
    delete_user()
~~~

正确做法是从可信 token、服务端会话或数据库获得主体身份，再判断主体能否对目标资源执行指定动作。

| 模型 | 判断方式 | 适用场景 |
| --- | --- | --- |
| RBAC | 用户是否有 admin、editor 等角色 | 后台管理 |
| ABAC | 用户、资源、环境属性是否满足策略 | 多租户、数据分级 |
| 资源归属 | resource.owner_id 是否等于 user.id | 用户只能看自己的资源 |
| Scope | token 是否有 documents:read | OAuth、API Key |

多租户系统中，tenant 必须来自已认证用户的可信上下文。URL 或 JSON 的 tenant_id 最多是待验证选择意图，不能成为权限边界。

~~~python
def get_document_for_current_tenant(
    document_id: str,
    current_user: dict[str, str],
) -> dict[str, str] | None:
    return db.get_document(
        document_id,
        tenant_id=current_user["tenant_id"],
    )
~~~

企业知识库、RAG 检索、工具调用都必须把用户、租户、文档级权限过滤放在模型之前。模型输出只能视为不可信输入，不能成为授权凭据。

## 4. 会话：保持登录状态

Cookie 会话流程：登录成功后服务端把 session 放入 Redis 或数据库，再把难以猜测的 session ID 写入 Cookie；浏览器后续自动携带 Cookie，服务端查询 session 得到用户。

无状态 access token 流程：服务端签发短期 token，客户端每次放入 Authorization，服务端验证签名和声明。Cookie 会话易于立即失效，但需要共享会话存储和 CSRF 防护；JWT 扩展方便，但用户禁用、权限变化、登出、refresh token 轮换和审计通常仍需要服务端状态。

~~~http
Set-Cookie: session_id=<opaque-random-value>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=1800
~~~

- HttpOnly：前端 JavaScript 不能读取 Cookie，降低 XSS 窃取会话风险。
- Secure：仅通过 HTTPS 发送，生产必须开启。
- SameSite：限制跨站请求携带 Cookie，通常从 Lax 起步。
- Max-Age、Path、Domain：限制生命周期与作用范围。

Cookie 自动发送，因此写操作还要考虑 CSRF token 和 Origin/Referer 校验。登录、改密、权限提升时应旋转 session ID；登出要使服务端 session 或 refresh token 失效。

## 5. 限流：限制资源消耗速度

限流防止密码猜测、API Key 滥用、模型配额耗尽和突发流量压垮下游。它不替代认证、授权、输入校验、并发控制和容量规划。

常用限流维度：登录接口使用 IP 加账号标识；已登录 API 按用户 ID；多租户模型服务按租户 ID、用户 ID、模型路由；开放平台按 API Key；高成本任务还要限制并发任务数或 token 预算。

令牌桶是常用算法：桶最大 10 个令牌，每秒补 1 个。空闲时可积攒令牌并允许短时突发，之后平均速率仍受控制。

~~~http
HTTP/1.1 429 Too Many Requests
Retry-After: 12

{"code":"RATE_LIMITED","message":"请求过于频繁，请稍后再试","trace_id":"..."}
~~~

下面代码只用于学习。多进程、多实例、重启或高并发生产环境中不可靠；生产应使用 Redis 原子操作、API 网关或专用限流服务。

~~~python
import time
from collections import defaultdict, deque


class FixedWindowRateLimiter:
    """用于学习的进程内滑动窗口限流器。"""

    def __init__(self, limit: int, window_seconds: float) -> None:
        self.limit = limit
        self.window_seconds = window_seconds
        self._requests: dict[str, deque[float]] = defaultdict(deque)

    def allow(self, key: str) -> tuple[bool, float]:
        now = time.monotonic()
        timestamps = self._requests[key]

        while timestamps and now - timestamps[0] >= self.window_seconds:
            timestamps.popleft()

        if len(timestamps) >= self.limit:
            retry_after = self.window_seconds - (now - timestamps[0])
            return False, max(retry_after, 0)

        timestamps.append(now)
        return True, 0
~~~

## 6. 幂等：重复提交只能生效一次

用户会双击，网络会超时，SDK、网关和消息消费者会重试。对创建订单、扣款、发邮件、调用有副作用工具的 POST，请求重复到达是常态。

客户端为一个逻辑操作生成 UUID：

~~~http
POST /api/orders
Idempotency-Key: 2e03074d-0f11-4af0-9618-...
~~~

服务端按“主体身份 + 操作名 + 幂等键”保存状态：

~~~text
不存在 -> PROCESSING（第一个请求获得执行权）
PROCESSING -> SUCCEEDED（保存最终响应，重复请求回放）
PROCESSING -> FAILED（是否允许重试由业务规则决定）
~~~

必须做到：

1. 同一个键但请求体不同，返回 409 IDEMPOTENCY_KEY_REUSED。
2. 同键同体且已成功，回放原状态码和响应体。
3. 同键正在处理，可短暂等待、返回 202 或 409；策略必须明确。
4. 依靠数据库唯一约束或 Redis 原子操作争夺执行权，不能只先查再插。
5. TTL 应覆盖客户端重试窗口。
6. 外部副作用还要配合事务、outbox 模式或供应商幂等能力。

幂等不是内容去重。两次金额相同的订单可能是两个真实业务操作，不能仅凭内容相同擅自合并。

## 7. 超时与重试

超时是一套层级预算。若请求总预算 10 秒，不能让鉴权查询、检索、模型调用各等 10 秒。

~~~text
总预算 10s
  鉴权查询：0.5s
  检索：2s
  LLM 首次响应：6s
  序列化与网络余量：1.5s
~~~

常见超时：连接超时、读超时、写超时、连接池获取超时、端到端 deadline。

~~~python
import httpx

timeout = httpx.Timeout(
    timeout=10.0,
    connect=1.0,
    read=6.0,
    write=3.0,
    pool=1.0,
)

async with httpx.AsyncClient(timeout=timeout) as client:
    response = await client.get("https://example.com/api/health")
    response.raise_for_status()
~~~

应用层超时通常只表示调用方不再等待，下游可能已经收到请求并继续执行。因此写操作超时后不能直接盲目重试，必须先设计幂等机制。

通常可谨慎重试：网络瞬断、连接重置、幂等读调用的超时、502/503/504，以及遵从 Retry-After 的 429。通常不应重试：400、401、403、404、409、422，以及不知道是否已成功的非幂等写操作。

指数退避加抖动可避免故障恢复时所有客户端同时重试：

~~~text
delay = min(max_delay, base * 2^(attempt - 1)) + random_jitter
~~~

~~~python
import asyncio
import random
from collections.abc import Awaitable, Callable


async def retry_transient(
    operation: Callable[[], Awaitable[str]],
    attempts: int = 3,
) -> str:
    """对已确认幂等的瞬时超时执行有限重试。"""
    for attempt in range(1, attempts + 1):
        try:
            return await operation()
        except TimeoutError:
            if attempt == attempts:
                raise

            delay = 0.2 * (2 ** (attempt - 1)) + random.uniform(0, 0.1)
            await asyncio.sleep(delay)

    raise RuntimeError("不可达")
~~~

不要捕获 Exception 后全部重试，否则参数错误、权限错误、编程 bug 都会被掩盖并放大。重试还需要最大次数、总时间上限、可重试错误白名单、取消传播、日志和熔断/降级边界。

## 8. CORS：浏览器的跨源读取保护

CORS 是浏览器安全机制，限制网页能否读取另一个源的响应。它不是认证机制：curl、Python 脚本和攻击者不会被 CORS 阻挡，服务端认证授权仍不可少。

origin 由协议、主机、端口组成：

~~~text
http://localhost:3000
http://localhost:8000
https://app.example.com
~~~

三者均是不同 origin。带 Authorization 或 JSON 的跨域 POST 常先发送 OPTIONS 预检，服务端必须允许对应来源、方法和请求头。

~~~python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.example.com",
        "http://localhost:5173",
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "Idempotency-Key"],
    expose_headers=["Retry-After", "X-Request-ID"],
    max_age=600,
)
~~~

生产环境列出精确来源。使用 allow_credentials=True 时，不要把来源、方法、请求头笼统设为通配符；跨域 Cookie 还会扩大 CSRF 风险。开发 localhost 与生产域名必须分环境配置。

## 9. 错误码：HTTP 语义加业务语义

HTTP 状态码表达通用语义，业务错误码表达领域原因。建议统一响应：

~~~json
{
  "code": "DOCUMENT_ACCESS_DENIED",
  "message": "你没有访问该文档的权限",
  "trace_id": "01J...",
  "details": [{"field": "document_id", "reason": "forbidden"}]
}
~~~

| 状态码 | 使用场景 |
| --- | --- |
| 200 / 201 / 204 | 成功读取、创建、删除 |
| 202 | 已接收，异步处理中 |
| 400 | JSON 损坏、参数组合非法 |
| 401 | 缺失或无效认证凭据 |
| 403 | 已认证但无权限 |
| 404 | 资源不存在 |
| 409 | 幂等键冲突、版本冲突 |
| 413 / 415 | 请求体过大 / 媒体类型不支持 |
| 422 | 字段格式正确但校验失败 |
| 429 | 触发限流 |
| 500 | 未预期内部错误 |
| 502 / 503 / 504 | 上游坏响应、暂不可用、网关超时 |

不要把所有失败包装成 200，也不要将内部堆栈、SQL、密钥或依赖异常直接返回。trace_id 用于日志定位；对外信息要少而稳定，对内日志要完整但脱敏。

## 10. LLM 服务中的组合方式

企业知识库问答通常按以下顺序处理：

1. 精确 CORS 配置允许正式前端。
2. 按租户、用户和模型路由限流，并限制并发。
3. 验证 access token。
4. 校验用户对知识库与文档的访问权限。
5. 校验消息长度、附件类型、工具参数。
6. 对 Redis、数据库、检索和模型调用设置独立超时及总 deadline。
7. 仅对幂等的检索和读取调用做有限重试。
8. 客户端中断流式响应时，取消下游任务并释放连接。
9. 记录 trace_id、模型版本、token、延迟和错误类别，且脱敏。
10. 对发邮件、创建工单、扣费等副作用工具增加动作级授权、Idempotency-Key、审计与审批。

## 11. 必测场景与学习路线

| 能力 | 必测断言 |
| --- | --- |
| 认证 | 无效或过期 token 为 401，日志不含 token |
| 授权 | 跨租户与非资源所有者被拒绝 |
| 会话 | 登出/改密后失效，Cookie 属性正确 |
| 限流 | 超额为 429 且包含 Retry-After |
| 幂等 | 同键同体回放；同键异体 409；并发仅一个副作用 |
| 超时 | 下游卡住仍在总预算内结束且资源释放 |
| 重试 | 只重试白名单失败，到上限停止 |
| CORS | 合法来源通过预检，非法来源没有允许头 |
| 错误码 | 500 不泄露内部细节，响应能以 trace_id 定位 |

四周建议：

- 第 1 周：写 login、me、documents 接口，理解 401、403 与租户过滤。
- 第 2 周：分别实践 Cookie 会话和 Bearer token，配置本地与生产 CORS，加入统一错误结构。
- 第 3 周：实现 Redis 或网关级限流，配置 HTTP 超时，模拟 429、503 与超时。
- 第 4 周：完成创建工单接口，用 Idempotency-Key 模拟双击、响应丢失与并发提交。

## 12. 上线前检查表

- [ ] HTTPS 已启用；敏感 Cookie 有 Secure、HttpOnly、合适的 SameSite。
- [ ] 所有受保护接口均在服务端认证和授权；所有查询均带租户边界。
- [ ] 密钥不进入仓库、前端或日志。
- [ ] 登录与高成本接口使用分布式限流。
- [ ] 每个外部依赖均有连接、读取、连接池和总超时。
- [ ] 重试白名单、次数、退避、抖动与幂等边界明确。
- [ ] 有副作用的写操作使用幂等键或等价机制。
- [ ] CORS 仅开放必要的 origin、方法和请求头。
- [ ] 错误响应稳定，500 不泄露内部细节，所有请求带 trace_id。
- [ ] 自动化测试覆盖认证、越权、限流、超时、重复提交和跨域。

## 13. 官方资料

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [FastAPI CORS](https://fastapi.tiangolo.com/tutorial/cors/)
- [MDN：HTTP 状态码](https://developer.mozilla.org/docs/Web/HTTP/Status)
- [MDN：CORS](https://developer.mozilla.org/docs/Web/HTTP/Guides/CORS)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [RFC 9110：HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

先在一个小型 FastAPI 项目中做正确“可信身份、服务端授权、有限资源、可恢复失败、稳定错误语义”这五件事，再逐步接入 Redis、数据库、网关与可观测性。这就是从“会调用模型”走向“能交付可靠大模型服务”的基础。