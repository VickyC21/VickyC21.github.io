---
layout: post
title: "Python 并发请求治理：超时、重试、取消、背压与资源释放入门讲义"
date: 2026-08-17 01:00:00 +0800
categories: [Python基础]
tags: [Python, asyncio, 并发, 异步编程, LLM]
---

# Python 并发请求治理：超时、重试、取消、背压与资源释放入门讲义

大模型开发不只是“把请求发给模型”。多个用户同时调用模型、上游偶尔变慢、客户端中途断开时，服务仍要保持可预测：不能无限堆积任务，不能无限等待，不能把暂时故障放大成流量风暴，也不能遗留连接、文件或后台任务。

本讲义围绕六项能力建立完整心智模型：

1. 同时处理多个 I/O 请求。
2. 为每个等待阶段设定合理的超时边界。
3. 只对适合的失败做有限、带抖动的重试。
4. 在请求不再需要时正确取消任务。
5. 用背压限制排队和并发，保护服务与下游。
6. 无论成功、失败还是取消，都释放资源。

示例以 Python 3.11+ 为主。HTTP 示例使用 `httpx`；标准库部分不需要安装它。

> 并发不是“尽可能多地同时发请求”，而是在容量有限时，有边界地管理等待中的工作。

## 1. 一次 LLM 请求的完整链路

```text
接收请求 -> 排队等待名额 -> 建立连接 -> 发送请求 -> 等待响应
        -> 解析结果 -> 返回客户端 -> 释放所有资源
```

| 阶段 | 常见问题 | 应对机制 |
| --- | --- | --- |
| 排队等待名额 | 高峰期任务无限积压，内存上涨 | 有界队列、拒绝或快速失败 |
| 建立连接 | DNS、网络或握手卡住 | 连接超时 |
| 等待模型响应 | 上游过慢或永不返回 | 读取超时、总超时 |
| 收到临时错误 | 429、503、连接重置 | 有条件重试与退避 |
| 客户端离开 | 浏览器刷新、用户停止生成 | 取消任务 |
| 任意结束路径 | 连接、文件、锁、会话未关闭 | 上下文管理器和 `finally` |

这些机制并不相互替代：

- **超时**回答“等多久”。
- **重试**回答“失败后是否再试”。
- **取消**回答“结果还值不值得继续要”。
- **背压**回答“系统还能否接下更多工作”。
- **资源释放**保证任何结束路径都不留下垃圾。

## 2. 并发、并行与协程

- **并发（concurrency）**：多个任务在时间上交错推进。一个任务等网络时，程序处理另一个。
- **并行（parallelism）**：多个任务在同一时刻由多个 CPU 核真正执行。
- **异步 I/O**：程序主动让出等待网络、数据库、文件的时间，特别适合 API 服务。
- **协程（coroutine）**：可以暂停、之后恢复的函数。`async def` 定义协程函数；调用它得到协程对象。

LLM API、数据库、HTTP、Redis 通常是 I/O 密集型工作，适合 `asyncio`。大规模本地推理、复杂图像处理和长时间计算属于 CPU 密集型，不能靠增加协程数解决，应考虑进程、任务队列或专门计算服务。

### 2.1 第一个 asyncio 程序

```python
import asyncio


async def fetch_one(name: str, delay: float) -> str:
    """模拟一次网络请求。

    Args:
        name: 请求名称。
        delay: 模拟网络等待的秒数。

    Returns:
        请求完成后的文本。
    """
    print(f"{name}: 开始")
    await asyncio.sleep(delay)
    print(f"{name}: 完成")
    return name


async def main() -> None:
    """并发执行三个等待型任务。"""
    results = await asyncio.gather(
        fetch_one("请求 A", 1.0),
        fetch_one("请求 B", 2.0),
        fetch_one("请求 C", 1.5),
    )
    print(results)


asyncio.run(main())
```

总耗时接近 2 秒，而不是 4.5 秒。原因不是 CPU 变快，而是一个任务等待时，事件循环运行另一个任务。

`await` 是协作式切换点。若在 `async def` 中执行很长的普通计算循环而没有 `await`，其他协程仍会被堵住。

### 2.2 gather 与 TaskGroup

`asyncio.gather()` 适合收集彼此独立的结果。Python 3.11 的 `asyncio.TaskGroup` 适合“共同命运”的任务：一个必要任务失败时，任务组取消其余未完成任务。

```python
import asyncio


async def load_part(part_id: int) -> str:
    """模拟加载一个分片。"""
    await asyncio.sleep(0.1)
    if part_id == 2:
        raise RuntimeError("分片 2 不可用")
    return f"分片 {part_id}"


async def load_all() -> None:
    """演示共同失败语义。"""
    try:
        async with asyncio.TaskGroup() as group:
            for part_id in range(3):
                group.create_task(load_part(part_id))
    except* RuntimeError as errors:
        for error in errors.exceptions:
            print(f"加载失败：{error}")


asyncio.run(load_all())
```

判断方法：任务失败不影响其他结果时，考虑 `gather(..., return_exceptions=True)` 并逐项检查；子任务共同构成不可分割操作时，优先 `TaskGroup`。

## 3. 并发控制：Semaphore 限制在飞请求数

直接创建一万个请求并不安全。HTTP 连接、文件描述符、内存、数据库连接池和 LLM 供应商配额都有限。

`asyncio.Semaphore` 是计数器：拿到名额才能进入临界区，离开时归还名额。

```python
import asyncio


async def call_model(
    request_id: int,
    semaphore: asyncio.Semaphore,
) -> str:
    """在固定并发上限内模拟调用模型。"""
    async with semaphore:
        print(f"请求 {request_id} 已进入下游")
        await asyncio.sleep(1)
        return f"结果 {request_id}"


async def main() -> None:
    """最多允许三个请求同时访问下游。"""
    semaphore = asyncio.Semaphore(3)
    tasks = [
        call_model(request_id, semaphore)
        for request_id in range(10)
    ]
    print(await asyncio.gather(*tasks))


asyncio.run(main())
```

上限是 **in-flight 请求数**，即正在访问下游的请求数。等待信号量的任务仍在内存中，所以信号量不能限制排队任务总数；背压会解决这一点。

并发上限没有固定答案，应结合上游并发配额、P95 延迟、实例数量、连接池大小、单请求内存和可接受排队时间。单机 `Semaphore` 只能管理当前进程；多实例服务还需要跨实例的限流或队列策略。

## 4. 超时：为等待设定终点

超时不是失败原因，而是**调用方不再愿意等待的边界**。没有超时的网络请求会让任务、连接和用户请求长期悬挂。

- **单次尝试超时**：某一次 HTTP 请求最多等多久。
- **总预算超时**：排队、重试和多次尝试加起来，整个业务操作最多等多久。

```python
import asyncio


async def slow_operation() -> str:
    """模拟耗时过长的操作。"""
    await asyncio.sleep(5)
    return "完成"


async def main() -> None:
    """在一秒后中止等待。"""
    try:
        async with asyncio.timeout(1.0):
            print(await slow_operation())
    except TimeoutError:
        print("操作超过一秒，已停止等待")


asyncio.run(main())
```

`asyncio.timeout()` 到期时会取消当前作用域内的等待，并在作用域外表现为 `TimeoutError`。因此超时和取消紧密相关。

使用 `httpx` 时，可以分别配置连接、读取、写入和连接池等待超时：

```python
import httpx


client = httpx.AsyncClient(
    timeout=httpx.Timeout(
        connect=3.0,
        read=30.0,
        write=10.0,
        pool=2.0,
    ),
)
```

对流式 LLM 输出，`read=30` 通常表示两段数据之间最长允许等待 30 秒，并不一定等于整个生成只能持续 30 秒。还需要总请求预算和最大输出 token。

超时值应来自产品目标：聊天首答、后台总结和流式生成的用户容忍时间不同。超时太短会误杀正常慢请求；太长会占住资源并让故障扩大。

## 5. 重试：只重试暂时可能恢复的错误

| 错误类型 | 通常重试？ | 原因 |
| --- | --- | --- |
| 连接重置、临时 DNS 失败 | 可以有限重试 | 可能是短暂网络抖动 |
| HTTP 429 | 可以，尊重 `Retry-After` | 上游要求稍后再试 |
| HTTP 500、502、503、504 | 可以有限重试 | 服务端可能短暂故障 |
| HTTP 400 | 不应重试 | 请求格式错误，重试不会改变它 |
| HTTP 401、403 | 不应重试 | 认证或权限问题，需要修复配置 |
| 用户取消 | 绝不重试 | 用户已明确不再需要结果 |

### 5.1 指数退避与抖动

所有客户端同时遇到 503，若又都在一秒后重试，会再次形成尖峰，这叫惊群效应。

```text
第 1 次重试前等待：base * 2^0
第 2 次重试前等待：base * 2^1
第 3 次重试前等待：base * 2^2
```

抖动（jitter）是在等待时间中加入随机性，让请求分散。

```python
import asyncio
import random


async def wait_before_retry(
    attempt: int,
    base_delay: float = 0.5,
    max_delay: float = 8.0,
) -> None:
    """按指数退避加随机抖动等待。"""
    ceiling = min(max_delay, base_delay * (2 ** attempt))
    await asyncio.sleep(random.uniform(0, ceiling))
```

### 5.2 幂等性决定能不能重试

**幂等**表示同一操作执行一次或多次，最终效果相同。读取数据通常幂等；扣款、创建订单、发送邮件通常不幂等。

当 LLM 请求会触发工具调用、写数据库或计费时，网络超时后不能盲目重发。应使用幂等键或服务端去重，确保客户端重试不会产生两次副作用。

### 5.3 一个最小、可审查的重试函数

```python
import asyncio
import random
from collections.abc import Awaitable, Callable
from typing import TypeVar


T = TypeVar("T")


async def retry_async(
    operation: Callable[[], Awaitable[T]],
    *,
    max_attempts: int = 3,
    base_delay: float = 0.5,
) -> T:
    """对明确的暂时性错误执行有限重试。"""
    if max_attempts < 1:
        raise ValueError("max_attempts 必须至少为 1")

    for attempt in range(max_attempts):
        try:
            return await operation()
        except (ConnectionError, TimeoutError):
            if attempt == max_attempts - 1:
                raise

            ceiling = base_delay * (2 ** attempt)
            await asyncio.sleep(random.uniform(0, ceiling))

    raise RuntimeError("不应执行到这里")
```

它故意不捕获所有 `Exception`，也不捕获 `asyncio.CancelledError`。参数错误、权限错误、代码缺陷和用户取消都不应被伪装成“再试一次”。

## 6. 取消：当结果不再有价值时停止

取消常见于用户点击“停止生成”、浏览器断开、上层超时、任务组中必要任务失败或服务关闭。

`asyncio` 通过向协程注入 `asyncio.CancelledError` 完成取消。它会在下一个 `await` 等可取消点出现。

```python
import asyncio


async def stream_tokens() -> None:
    """模拟流式输出，并在取消时执行清理。"""
    try:
        for token_index in range(100):
            await asyncio.sleep(0.2)
            print(f"token {token_index}")
    except asyncio.CancelledError:
        print("收到取消信号，停止读取上游流")
        raise
    finally:
        print("清理：关闭流、归还连接或记录状态")


async def main() -> None:
    """启动任务后主动取消它。"""
    task = asyncio.create_task(stream_tokens())
    await asyncio.sleep(0.7)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("调用方确认任务已取消")


asyncio.run(main())
```

三条规则：

1. 清理后必须重新抛出 `CancelledError`，否则调用方会误以为任务正常结束。
2. 不要用宽泛异常捕获包住整个协程；只捕获知道如何恢复的异常。
3. 取消不会立即杀死阻塞线程。若调用 `time.sleep()`、同步 HTTP 库或长时间 CPU 计算，事件循环无法及时响应取消。

错误写法是捕获 `CancelledError` 后只打印日志不抛出；正确结构是清理资源后使用 `raise` 继续传播取消。

## 7. 背压：让过载系统保持诚实

只使用 `Semaphore` 时，下面的写法仍可能创建海量等待任务：

```python
tasks = [call_model(item) for item in one_million_items]
results = await asyncio.gather(*tasks)
```

虽然后端同时只有有限请求，但一百万个协程对象、输入数据和结果引用仍可能耗尽内存。

`asyncio.Queue(maxsize=N)` 是有界等待区。队列满时，生产者停在 `await queue.put()`，消费者腾出空间后才继续，这就是背压。

```python
import asyncio


SENTINEL = object()


async def producer(
    queue: asyncio.Queue[int | object],
    items: list[int],
) -> None:
    """逐个生产任务；队列满时自动等待。"""
    for item in items:
        await queue.put(item)
        print(f"已入队：{item}")
    await queue.put(SENTINEL)


async def consumer(
    queue: asyncio.Queue[int | object],
    worker_id: int,
) -> None:
    """从队列中取任务并串行处理。"""
    while True:
        item = await queue.get()
        try:
            if item is SENTINEL:
                await queue.put(SENTINEL)
                return

            print(f"工作者 {worker_id} 处理：{item}")
            await asyncio.sleep(0.5)
        finally:
            queue.task_done()


async def main() -> None:
    """一个生产者与两个消费者协作处理有限队列。"""
    queue: asyncio.Queue[int | object] = asyncio.Queue(maxsize=3)

    async with asyncio.TaskGroup() as group:
        group.create_task(producer(queue, list(range(10))))
        group.create_task(consumer(queue, 1))
        group.create_task(consumer(queue, 2))


asyncio.run(main())
```

两层容量不同：`Queue(maxsize=3)` 控制**排队长度**，两个消费者控制**同时处理数量**。

队列满后的策略由产品决定：HTTP API 可返回 429 或 503 并附带 `Retry-After`；内部任务可等待有限时间后失败，或转入持久化消息队列。不能把无限等待当成背压。

还要区分：

- **并发限制**：同时最多多少进行中的请求。
- **速率限制**：单位时间最多多少请求或 token。
- **背压**：最多积压多少待处理任务。

在 LLM 服务中三者通常都需要，`Semaphore` 只能解决第一项。

## 8. 资源释放：所有结束路径都要关闭

资源包括 HTTP 连接、响应流、数据库连接、文件句柄、锁、临时文件和后台任务。资源泄漏在低流量时不明显，高并发时会变成连接池耗尽、文件描述符耗尽或内存上涨。

### 8.1 使用上下文管理器

```python
import httpx


async def fetch_json(url: str) -> dict:
    """请求 JSON，并自动关闭客户端与响应资源。"""
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()
```

`async with` 的价值不在写法好看，而在于无论成功、异常还是取消，退出作用域时都会执行清理。

上例适合演示；服务端通常应在应用启动时创建可复用的 `AsyncClient`，在关闭时调用 `aclose()`。每个请求都新建客户端会失去连接复用，也更易耗尽资源。

```python
import httpx


class ModelGateway:
    """封装模型 HTTP 客户端的生命周期。"""

    def __init__(self) -> None:
        """初始化尚未启动的网关。"""
        self._client: httpx.AsyncClient | None = None

    async def start(self) -> None:
        """创建可复用的异步 HTTP 客户端。"""
        self._client = httpx.AsyncClient(
            timeout=httpx.Timeout(connect=3.0, read=30.0, write=10.0, pool=2.0),
            limits=httpx.Limits(max_connections=20, max_keepalive_connections=10),
        )

    async def close(self) -> None:
        """关闭客户端并释放连接池。"""
        if self._client is not None:
            await self._client.aclose()
            self._client = None
```

对于自定义资源，使用 `finally`：

```python
resource = await acquire_resource()
try:
    await use_resource(resource)
finally:
    await release_resource(resource)
```

`finally` 会在正常返回、异常和超时导致的取消路径中运行。清理代码应短小可靠，且不应无限等待。

## 9. 组合实践：受控异步 HTTP 请求器

下面将并发上限、总超时、单次 HTTP 超时、有限重试、取消传播和资源复用组合在一起。这是教学骨架，不包含鉴权、日志、速率限制和跨实例协调。

先安装依赖：

```bash
python -m pip install httpx
```

```python
import asyncio
import random

import httpx


class TransientRequestError(Exception):
    """表示适合在有限次数内重试的临时请求错误。"""


class AsyncRequestRunner:
    """以受控并发调用外部 HTTP 服务。"""

    def __init__(
        self,
        *,
        max_concurrency: int = 10,
        max_attempts: int = 3,
        total_timeout_seconds: float = 20.0,
    ) -> None:
        """初始化请求治理参数。"""
        if max_concurrency < 1:
            raise ValueError("max_concurrency 必须至少为 1")
        if max_attempts < 1:
            raise ValueError("max_attempts 必须至少为 1")

        self._semaphore = asyncio.Semaphore(max_concurrency)
        self._max_attempts = max_attempts
        self._total_timeout_seconds = total_timeout_seconds
        self._client = httpx.AsyncClient(
            timeout=httpx.Timeout(
                connect=3.0,
                read=15.0,
                write=10.0,
                pool=2.0,
            ),
            limits=httpx.Limits(
                max_connections=max_concurrency,
                max_keepalive_connections=max_concurrency,
            ),
        )

    async def aclose(self) -> None:
        """关闭底层 HTTP 客户端并释放连接池。"""
        await self._client.aclose()

    async def post_json(self, url: str, payload: dict) -> dict:
        """在受控并发、总超时和有限重试下发送 JSON POST 请求。"""
        async with asyncio.timeout(self._total_timeout_seconds):
            async with self._semaphore:
                return await self._retry(url, payload)

    async def _retry(self, url: str, payload: dict) -> dict:
        """只对明确的短暂错误执行带抖动的重试。"""
        retryable_errors = (
            TransientRequestError,
            httpx.ConnectError,
            httpx.ReadTimeout,
            httpx.RemoteProtocolError,
        )

        for attempt in range(self._max_attempts):
            try:
                response = await self._client.post(url, json=payload)
                if response.status_code == 429 or 500 <= response.status_code <= 504:
                    raise TransientRequestError(
                        f"下游暂时不可用：{response.status_code}"
                    )
                response.raise_for_status()
                return response.json()
            except retryable_errors:
                if attempt == self._max_attempts - 1:
                    raise

                ceiling = min(4.0, 0.5 * (2 ** attempt))
                await asyncio.sleep(random.uniform(0, ceiling))

        raise RuntimeError("不应执行到这里")
```

调用方必须关闭请求器：

```python
async def main() -> None:
    """演示显式关闭请求器。"""
    runner = AsyncRequestRunner(max_concurrency=5)
    try:
        result = await runner.post_json(
            "https://example.com/v1/chat",
            {"message": "你好"},
        )
        print(result)
    finally:
        await runner.aclose()
```

关键取舍：

- `Semaphore` 放在重试流程外侧，一次逻辑请求重试时仍占用一个下游名额。它更稳定，但吞吐略低。
- 总超时覆盖信号量等待、HTTP 调用、退避等待和重试，避免长尾请求无限消耗资源。
- 只重试明确的短暂错误；400、401、403 直接失败。
- 不捕获 `CancelledError`，客户端取消能快速向下游和调用方传播。
- 没有假装实现熔断器、持久化队列或跨实例协调；这些属于下一层需求。

## 10. 常见错误与修正

| 错误写法 | 后果 | 修正 |
| --- | --- | --- |
| 对所有 `Exception` 无条件重试 | 参数、权限和代码 bug 被掩盖，故障放大 | 只重试明确的短暂异常与状态码 |
| 所有失败立即重试 | 造成惊群，压垮已故障的上游 | 指数退避、随机抖动、最多尝试次数 |
| 只设单次 HTTP 超时 | 多次重试后总耗时失控 | 增加整个业务操作的总时间预算 |
| `create_task` 后从不等待 | 异常丢失、关闭时泄漏后台工作 | 保存引用，使用 TaskGroup 或显式 await |
| 每个 API 请求新建 AsyncClient | 无法复用连接，性能差 | 在应用生命周期内复用客户端 |
| 取消时捕获后不 raise | 上层误判成功 | 清理后重新抛出 CancelledError |
| 只用信号量，不限制任务总数 | 队列无限长，内存上涨 | 使用有界队列或入口快速拒绝 |
| 在 `async def` 中调用 `time.sleep()` | 阻塞整个事件循环 | 使用 `await asyncio.sleep()` |
| 对非幂等写操作盲目重试 | 重复扣款、写入或通知 | 使用幂等键、服务端去重或人工处理 |

## 11. 如何验证

并发代码最容易“看起来能跑”，却在异常路径失效。测试至少覆盖成功、超时、重试、取消和资源释放。

```python
import asyncio
import pytest


@pytest.mark.asyncio
async def test_timeout_stops_slow_operation() -> None:
    """验证总超时会中止过慢操作。"""

    async def never_finishes() -> None:
        await asyncio.sleep(10)

    with pytest.raises(TimeoutError):
        async with asyncio.timeout(0.01):
            await never_finishes()
```

还应验证：

1. 用计数器记录最大同时执行数，断言不超过 Semaphore 配置。
2. 第一次失败、第二次成功时确实返回结果。
3. 临时错误耗尽次数后抛出，400、401、403 不重试。
4. 取消发生时不触发下一次重试，`finally` 中的清理已执行。
5. 总超时先到时，重试循环停止。
6. 队列已满时，入口按策略等待、拒绝或转存，不无限积压。

上线后至少记录请求数、成功率、异常类型、尝试次数、队列长度、拒绝数、P50/P95/P99 延迟、取消数、连接池等待时间和资源使用量。指标应帮助你回答：慢在哪里？重试是否放大故障？排队是否超过用户可接受时间？

## 12. 学习顺序与练习

### 第一步：协程与任务

- 用 `asyncio.sleep()` 写三个并发任务，比较串行和并发耗时。
- 分别使用 `gather` 与 `TaskGroup`，观察一个子任务失败后的差异。
- 故意在协程中使用 `time.sleep()`，理解阻塞事件循环的后果。

### 第二步：并发上限和超时

- 为十个模拟请求加 `Semaphore(3)`，记录同时执行的最大数量。
- 为模拟慢请求增加 `asyncio.timeout()`。
- 区分“获取信号量前超时”与“已开始 HTTP 请求后超时”。

### 第三步：重试和取消

- 编写前两次失败、第三次成功的假 API，验证指数退避。
- 编写一个 400 错误的假 API，验证不会重试。
- 启动流式任务后取消它，确认 `finally` 被执行且调用方收到 `CancelledError`。

### 第四步：背压和完整服务

- 使用 `Queue(maxsize=10)` 与多个消费者处理任务。
- 队列满时选择明确策略：等待、返回 429，或写入可靠消息队列。
- 把受控请求器接入一个小型 FastAPI 接口，为超时、429 和客户端取消补充测试。

## 13. 复习清单

学完后，你应该能回答：

- 为什么 I/O 密集型 LLM API 服务通常适合 asyncio，而 CPU 密集型任务不一定适合？
- 为什么“每个请求都 create_task”不等于具备高并发能力？
- 单次 HTTP 超时和总时间预算分别保护什么？
- 哪些错误可能适合重试，哪些绝不应重试？
- 为什么重试需要退避和随机抖动？
- 为什么取消后必须重新抛出 CancelledError？
- Semaphore 和 Queue(maxsize=...) 分别限制什么？
- 为什么 HTTP 客户端应复用，又为什么关闭时必须显式关闭？
- 当 LLM 调用可能触发写操作时，如何用幂等键避免重复副作用？

当你能把这些问题落到代码、测试和指标上，就已经从“会异步语法”迈向“能让模型服务稳定运行”。
