---
layout: post
title: "Python 并发编程入门：asyncio、协程、线程与进程的适用边界"
date: 2026-08-17 10:00:00 +0800
categories: [Python基础]
tags: [Python, asyncio, 并发编程, 协程, 线程, 进程]
---

# Python 并发编程入门：asyncio、协程、线程与进程的适用边界

大模型应用服务常常要同时处理流式模型响应、数据库或向量检索、外部工具调用、超时与客户端取消。要把这类服务写稳，不能只会把代码“并发起来”，还要知道每种并发模型在等待什么资源、会占用什么资源，以及失败时怎样收尾。

本文以 Python 3.11+ 为基线，面向初学者讲清 `asyncio`、协程、线程与进程的关系、边界和工程实践。所有示例仅依赖标准库；可复制为独立 `.py` 文件运行。

## 1. 学完后应具备的能力

- 能区分并发（concurrency）与并行（parallelism）。
- 能解释进程、线程、协程各自拥有和共享什么资源。
- 面对 I/O 密集、CPU 密集、阻塞 SDK、批处理等场景，能选对实现模型。
- 能用 `asyncio` 管理任务、超时、取消、并发上限与资源释放。
- 能识别 GIL 的真实影响，而不是简单地认为“Python 线程没有用”。
- 能为 LLM 服务设计不阻塞事件循环的调用链。

## 2. 先建立正确模型

### 2.1 并发与并行不是同一个概念

**并发**指一段时间内交错推进多个任务。单核 CPU 也能并发：任务 A 等网络时，程序去推进任务 B。

**并行**指同一时刻在多个 CPU 核上执行多个计算。多进程最常用于 Python 的 CPU 并行。

一个直观比喻：

- 并发像一位服务员在多张桌子之间切换。等菜时服务员可以继续服务别桌。
- 并行像多位服务员同时服务不同桌。
- 关键不是桌子数量，而是服务员有没有在“等待”。如果服务员一直在切菜（持续占用 CPU），频繁切换不会让切菜本身更快。

### 2.2 I/O 密集与 CPU 密集

选型首先看时间主要花在哪里。

| 任务类型 | 主要等待对象 | 常见例子 | 优先方案 |
| --- | --- | --- | --- |
| I/O 密集 | 网络、磁盘、数据库、定时器 | 调用模型 API、HTTP、Redis、文件读写 | `asyncio`；阻塞库则用线程 |
| CPU 密集 | CPU 运算 | 图像处理、压缩、特征计算、大量 JSON/文本计算 | 进程池，或使用会释放 GIL 的原生库 |
| 混合任务 | I/O 与少量计算 | RAG 请求、模型工具编排 | `asyncio` 作主流程，局部转线程/进程 |

注意：不是“任务很多就上异步”。若任务没有等待点，异步只会让代码更复杂。

### 2.3 四个概念的层级关系

- **进程（process）**：操作系统分配资源的隔离单位。不同进程默认不共享 Python 内存；一个进程崩溃通常不会直接弄坏另一个进程的内存。
- **线程（thread）**：进程内的执行单元。线程共享同一进程的堆、全局变量和打开的文件描述符，因此通信方便，也需要同步保护。
- **协程（coroutine）**：由 Python 代码主动让出执行权的轻量任务。协程通常运行在一个线程的事件循环中。
- **事件循环（event loop）**：`asyncio` 的调度器。它在协程遇到 `await`、愿意暂停时，转而推进其他就绪任务。

协程不是线程，也不是“自动在后台运行”。一个普通的 `asyncio` 程序默认仍只有一个进程、一个线程；它靠协作式切换提高 I/O 等待期间的吞吐。

## 3. GIL：线程边界的关键背景

CPython（最常用的 Python 实现）有全局解释器锁，简称 GIL。通常同一时刻只能有一个线程执行 Python 字节码。因此：

- 多线程**不能可靠地加速纯 Python 的 CPU 密集循环**。
- 多线程对**阻塞 I/O 仍非常有用**：网络或文件 I/O 等待时，解释器可以让其他线程继续执行。
- NumPy、PyTorch、压缩库等底层原生代码可能在计算时释放 GIL；这时线程是否有效，要以该库文档和性能测量为准。
- GIL 不等于“线程安全”。`x += 1` 这类读改写操作仍可能产生竞争条件，访问共享可变状态时仍需要锁或更好的消息传递设计。

不要用“线程没有用”或“线程一定并行”作判断。先确认工作负载是 I/O、纯 Python CPU，还是会释放 GIL 的原生计算。

## 4. 协程与 asyncio：高并发 I/O 的首选

### 4.1 `async def`、协程对象与 `await`

`async def` 定义的是协程函数；调用它得到的是**协程对象**，此时函数体还没有运行。只有在 `await`、`asyncio.create_task()` 或事件循环入口中，协程才会被调度执行。

```python
import asyncio


async def say_hello(name: str) -> str:
    await asyncio.sleep(1)
    return f"你好，{name}"


async def main() -> None:
    coroutine = say_hello("Vicky")  # 仅创建对象，尚未运行
    message = await coroutine          # 交给事件循环执行并等待结果
    print(message)


asyncio.run(main())
```

`asyncio.run()` 适合放在脚本的最外层，负责创建并关闭事件循环。在已经由框架托管事件循环的环境（例如某些 Web 框架、Notebook）中，不要嵌套调用它；应使用框架规定的入口方式或直接 `await`。

### 4.2 顺序等待与并发等待

下面两段代码都使用异步函数，但只有第二段真正并发推进。

```python
import asyncio
import time


async def fetch(name: str) -> str:
    await asyncio.sleep(1)
    return f"{name} 完成"


async def sequential() -> None:
    started_at = time.perf_counter()
    first = await fetch("任务 A")
    second = await fetch("任务 B")
    print(first, second, round(time.perf_counter() - started_at, 2))


async def concurrent() -> None:
    started_at = time.perf_counter()
    first, second = await asyncio.gather(
        fetch("任务 A"),
        fetch("任务 B"),
    )
    print(first, second, round(time.perf_counter() - started_at, 2))


asyncio.run(sequential())  # 约 2 秒
asyncio.run(concurrent())  # 约 1 秒
```

`asyncio.gather()` 会并发等待一组可等待对象。它适合“所有子任务都应完成”这一简单场景；更复杂的子任务生命周期建议使用后文的 `TaskGroup`。

### 4.3 `create_task`：让任务开始独立推进

当你创建一个任务时，事件循环可以在你等待其他事情期间推进它。任务的异常必须被观察（`await task`、读取 `task.result()`，或被 `TaskGroup` 管理），否则可能只留下“Task exception was never retrieved”警告。

```python
import asyncio


async def query_model() -> str:
    await asyncio.sleep(1)
    return "模型结果"


async def query_database() -> str:
    await asyncio.sleep(0.5)
    return "检索结果"


async def main() -> None:
    model_task = asyncio.create_task(query_model())
    database_task = asyncio.create_task(query_database())

    model_result = await model_task
    database_result = await database_task
    print(model_result, database_result)


asyncio.run(main())
```

实际服务中应避免“创建后忘记管理”的游离任务（fire-and-forget）。请求已经结束、资源已经关闭时，游离任务仍可能继续写日志、访问数据库或抛出异常。

### 4.4 `TaskGroup`：结构化并发

Python 3.11 的 `asyncio.TaskGroup` 将一组子任务的生命周期绑在同一个作用域：退出 `async with` 前会等待它们；其中一个任务失败时，其余任务会被取消。这比手动创建一堆任务更适合请求内的并行查询。

```python
import asyncio


async def fetch_source(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"来自 {name} 的结果"


async def main() -> None:
    async with asyncio.TaskGroup() as group:
        policy_task = group.create_task(fetch_source("知识库", 0.4))
        profile_task = group.create_task(fetch_source("用户服务", 0.2))

    print(policy_task.result())
    print(profile_task.result())


asyncio.run(main())
```

当子任务可以独立失败、需要部分成功结果时，不应生搬硬套 `TaskGroup` 的“一个失败则取消全部”语义。需要先定义业务规则：失败能否降级、是否应返回部分结果、哪些失败必须终止请求。

### 4.5 最常见错误：在事件循环里做阻塞操作

下面的 `time.sleep()` 会阻塞整个事件循环。虽然函数声明为 `async`，其他协程仍无法运行。

```python
import asyncio
import time


async def wrong() -> None:
    time.sleep(2)  # 错误：阻塞事件循环


async def correct() -> None:
    await asyncio.sleep(2)  # 正确：把等待机会让给其他协程
```

同类问题还包括：在 `async def` 中直接使用同步 HTTP 客户端、同步数据库驱动、长时间的 CPU 循环、`subprocess.run()` 和大文件同步读写。遇到它们时，分别改用异步库、`asyncio.to_thread()` 或进程池。

## 5. asyncio 的工程要点：超时、取消、限流与资源释放

### 5.1 超时是边界，不是重试的替代品

每个外部依赖都应有超时。没有超时的请求可能永远占着连接、任务槽位与内存。Python 3.11+ 可用 `asyncio.timeout()`：

```python
import asyncio


async def remote_call() -> str:
    await asyncio.sleep(10)
    return "不会及时返回"


async def main() -> None:
    try:
        async with asyncio.timeout(2):
            result = await remote_call()
            print(result)
    except TimeoutError:
        print("远端调用超时，执行降级或返回可识别错误")


asyncio.run(main())
```

超时后要明确业务动作：返回错误、读取缓存、切换备用模型，还是进入异步队列。不要无限重试；重试必须同时考虑幂等性、总截止时间和下游承压能力。

### 5.2 取消是正常控制流，要清理资源后继续抛出

取消会在协程下一个可取消点（通常是 `await`）注入 `asyncio.CancelledError`。清理代码应放在 `finally` 中；通常不要吞掉取消异常，否则上层会误以为任务仍在运行。

```python
import asyncio


async def stream_tokens() -> None:
    try:
        while True:
            await asyncio.sleep(0.2)
            print("生成一个 token")
    finally:
        print("关闭流式连接，释放请求资源")


async def main() -> None:
    task = asyncio.create_task(stream_tokens())
    await asyncio.sleep(0.7)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        print("调用方确认任务已取消")


asyncio.run(main())
```

在 Web 服务中，客户端断开连接往往意味着请求任务应被取消。模型 SDK、HTTP 客户端和数据库客户端是否真正支持取消，需要查其文档并做集成测试。

### 5.3 用信号量限制并发，避免把下游压垮

一次性对一万个 URL 使用 `gather()` 可能创建一万个任务与连接。并发上限应基于连接池、目标服务配额、内存和压测结果确定。

```python
import asyncio


async def call_api(item: int, limiter: asyncio.Semaphore) -> str:
    async with limiter:
        await asyncio.sleep(0.2)
        return f"item={item}"


async def main() -> None:
    limiter = asyncio.Semaphore(5)
    results = await asyncio.gather(
        *(call_api(item, limiter) for item in range(20)),
    )
    print(results)


asyncio.run(main())
```

`Semaphore` 限制的是“同时进入临界区的数量”，不是请求速率。若还要限制每秒请求数，需要令牌桶、漏桶或网关限流策略。

### 5.4 用有界队列实现背压

背压的本质是：生产速度超过消费速度时，系统不能无限积压任务。`asyncio.Queue(maxsize=N)` 满了后，`put()` 会等待，迫使上游减速。

```python
import asyncio


async def producer(queue: asyncio.Queue[int]) -> None:
    for item in range(10):
        await queue.put(item)
        print(f"已入队: {item}")
    for _ in range(2):
        await queue.put(-1)  # 两个消费者各收到一个停止标记


async def worker(name: str, queue: asyncio.Queue[int]) -> None:
    while True:
        item = await queue.get()
        try:
            if item == -1:
                return
            await asyncio.sleep(0.3)
            print(f"{name} 已处理: {item}")
        finally:
            queue.task_done()


async def main() -> None:
    queue: asyncio.Queue[int] = asyncio.Queue(maxsize=3)
    async with asyncio.TaskGroup() as group:
        group.create_task(producer(queue))
        group.create_task(worker("worker-1", queue))
        group.create_task(worker("worker-2", queue))


asyncio.run(main())
```

真实队列还要设计拒绝策略：队列满时是等待、快速失败、丢弃低优先级任务，还是转移到持久化消息队列。这个选择是业务契约，不是 Python API 能替你决定的。

### 5.5 资源释放必须成对出现

连接、文件、锁和流式响应都必须在异常与取消时释放。优先使用上下文管理器：`async with` 对应异步资源，`with` 对应同步资源。以具体客户端文档为准，例如 HTTP 客户端通常应该复用一个长期连接池，而不是每个请求新建连接。

## 6. 线程：兼容阻塞式 I/O 的实用工具

### 6.1 什么时候使用线程

使用线程的典型理由不是“我要提升 CPU 速度”，而是：现有依赖只有同步接口，调用它会阻塞事件循环或主线程。例如旧版 SDK、同步数据库驱动、阻塞文件 API、某些企业内部客户端。

在异步函数中，优先用 `asyncio.to_thread()` 把短时阻塞调用转到默认线程池：

```python
import asyncio
import time


def blocking_legacy_sdk(prompt: str) -> str:
    time.sleep(1)
    return f"旧 SDK 的回复: {prompt}"


async def main() -> None:
    first = asyncio.to_thread(blocking_legacy_sdk, "问题 A")
    second = asyncio.to_thread(blocking_legacy_sdk, "问题 B")
    results = await asyncio.gather(first, second)
    print(results)


asyncio.run(main())
```

`to_thread()` 不会让纯 Python CPU 密集函数绕过 GIL；它的价值是防止阻塞函数冻住事件循环。

### 6.2 显式线程池与异常传播

需要控制线程数或在同步程序中并发执行 I/O 时，使用 `ThreadPoolExecutor`。通过 `future.result()` 或 `as_completed()` 获取结果时，工作线程异常会重新抛出，不能忽略。

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import time


def download(name: str) -> str:
    time.sleep(0.5)
    return f"下载完成: {name}"


with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(download, name) for name in ["a", "b", "c"]]
    for future in as_completed(futures):
        print(future.result())
```

线程数不是越多越好。过多线程会带来上下文切换、内存开销、连接数膨胀和下游限流。应以外部连接池上限和压测数据决定。

### 6.3 线程共享内存，必须防范竞争条件

多个线程读写同一可变对象时，结果依赖运行时序，称为竞争条件。用 `threading.Lock` 保护最小必要临界区：

```python
from concurrent.futures import ThreadPoolExecutor
from threading import Lock

counter = 0
counter_lock = Lock()


def increment() -> None:
    global counter
    for _ in range(10_000):
        with counter_lock:
            counter += 1


with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [executor.submit(increment) for _ in range(4)]
    for future in futures:
        future.result()

print(counter)  # 稳定为 40000
```

锁保护的是正确性，不是性能优化。长时间持锁会让并发退化为串行。更好的默认设计是减少共享可变状态，在线程间传递不可变数据或消息。

## 7. 进程：CPU 密集任务的并行方案

### 7.1 什么时候使用进程

纯 Python 的大量计算会长期占有 GIL。此时将独立工作单元分发到多个进程，才能利用多个 CPU 核。`ProcessPoolExecutor` 是标准库中最直接的选择。

```python
from concurrent.futures import ProcessPoolExecutor
import math


def count_primes(limit: int) -> int:
    count = 0
    for number in range(2, limit):
        is_prime = True
        for divisor in range(2, int(math.sqrt(number)) + 1):
            if number % divisor == 0:
                is_prime = False
                break
        if is_prime:
            count += 1
    return count


if __name__ == "__main__":
    limits = [40_000, 42_000, 44_000, 46_000]
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(count_primes, limits))
    print(results)
```

Windows 默认使用 `spawn` 启动子进程，会重新导入主模块。因此创建进程池的代码必须放在 `if __name__ == "__main__":` 保护内。否则可能反复创建子进程或直接报错。

### 7.2 进程的代价与限制

进程隔离带来并行能力，也带来成本：

- 创建进程和传输数据比创建协程、线程昂贵得多。
- 传给进程池的函数、参数和返回值通常需要可序列化（pickle）。嵌套函数、lambda、打开的连接对象常常不能直接传。
- 大对象在进程间复制会消耗时间和内存；需要评估数据量，必要时考虑共享内存、文件或专业计算框架。
- 数据库连接、HTTP 客户端等资源通常不能从父进程直接“带入”子进程复用；应在子进程按库规范初始化。
- 在 Web 请求处理路径中临时创建进程池代价很高。应复用受控的执行器，或把重任务交给独立 worker/队列系统。

## 8. 选型决策表：从工作负载倒推工具

| 你遇到的场景 | 推荐方案 | 原因 | 不推荐的做法 |
| --- | --- | --- | --- |
| 同时调用多个支持异步的 HTTP/模型 SDK | `asyncio` + 异步客户端 | 等待网络时可切换到其他请求 | 一个请求一个线程或串行等待 |
| 异步服务中必须调用同步 SDK | `asyncio.to_thread()` | 隔离阻塞，不冻住事件循环 | 直接在 `async def` 中调用同步函数 |
| 批量下载或读写文件，只有同步库 | 线程池 | I/O 等待时线程可切换 | 进程池或无限制建线程 |
| 纯 Python 图像/文本/数学批处理 | 进程池 | 可利用多核，绕过 GIL | 仅靠线程加速 Python 循环 |
| PyTorch/NumPy 等原生计算 | 查库文档并基准测试 | 可能已释放 GIL 或自带并行 | 先入为主地套进程池 |
| 长耗时、可重试的离线任务 | 任务队列 + 独立 worker | 请求线程不应被长期占用 | 在 API 请求中等待数分钟 |
| 单次请求内并行检索、模型调用 | `TaskGroup`/`gather` + 超时 | 生命周期属于同一请求 | 创建无主的后台任务 |

## 9. 面向 LLM 服务的组合方式

典型 RAG/Agent 请求不是只用一种模型，而是混合流程：

1. API 入口运行在异步事件循环中。
2. 并行发起异步的权限查询、向量检索、模型调用或工具调用。
3. 某个只有同步接口的企业 SDK 用 `asyncio.to_thread()` 包装。
4. 真正 CPU 重的文档解析、重排或批量计算交给受控进程池或异步任务队列。
5. 所有外部调用都具备超时、取消、并发上限、重试边界和观测日志。

示意代码如下。这里使用占位函数展示边界；生产中需要换成实际客户端，并复用其连接池。

```python
import asyncio
import time


def blocking_permission_check(user_id: str) -> bool:
    time.sleep(0.1)
    return user_id == "user-1"


async def search_knowledge_base(question: str) -> list[str]:
    await asyncio.sleep(0.2)
    return [f"与 {question} 相关的证据"]


async def call_llm(question: str, evidence: list[str]) -> str:
    await asyncio.sleep(0.4)
    return f"基于 {evidence[0]} 的回答"


async def answer(user_id: str, question: str) -> str:
    async with asyncio.timeout(3):
        has_permission, evidence = await asyncio.gather(
            asyncio.to_thread(blocking_permission_check, user_id),
            search_knowledge_base(question),
        )
        if not has_permission:
            raise PermissionError("无权访问该知识库")
        return await call_llm(question, evidence)


async def main() -> None:
    print(await answer("user-1", "如何处理超时？"))


asyncio.run(main())
```

这里的关键不是 `gather()` 本身，而是边界清晰：权限检查是阻塞 I/O，所以进入线程；检索和模型调用是异步 I/O，所以留在事件循环；若 `call_llm()` 实际是同步 SDK，也必须替换为 `to_thread()` 或支持异步的 SDK。

## 10. 重试、幂等与截止时间

重试会放大流量。下游已经超载时，大量立即重试可能造成“重试风暴”。可靠策略至少包括：

- 只重试暂时性失败，例如连接超时、可恢复的 5xx；参数校验失败、权限失败通常不该重试。
- 给整个请求设定**总截止时间**，不要让每一层都各自重试到超时。
- 使用指数退避（每次等待更久）并加随机抖动（jitter），避免大量请求同一时刻重试。
- 确认操作是否幂等。写入、扣费、发送邮件、创建工单等操作，在重试前应有幂等键或去重机制。
- 记录尝试次数、失败类型、下游名称和最终结果，便于判断问题在客户端、网络还是服务端。

示例中的重试函数只展示基本控制流；真实 HTTP 客户端应结合状态码和异常类型精细分类。

```python
import asyncio
import random


async def retry(operation, attempts: int = 3) -> str:
    for attempt in range(1, attempts + 1):
        try:
            return await operation()
        except TimeoutError:
            if attempt == attempts:
                raise
            delay = 0.2 * (2 ** (attempt - 1)) + random.uniform(0, 0.1)
            await asyncio.sleep(delay)
    raise RuntimeError("不会执行到这里")
```

为了保持示例聚焦，`operation` 没有完整类型标注。实际项目应为可调用对象、返回值与可重试异常建立明确的类型和测试。

## 11. 初学者高频误区清单

1. **以为 `async def` 会自动并发。** 只有在多个任务被调度，并在等待点让出控制权时才有并发。
2. **在异步函数中使用 `time.sleep()` 或同步 HTTP。** 这会阻塞整个事件循环；换成 `asyncio.sleep()`、异步客户端或 `to_thread()`。
3. **把 CPU 密集计算塞进协程。** 没有 `await` 的大循环会长时间霸占事件循环；转进程池或调整算法。
4. **不设超时。** 网络调用、锁、队列和流式生成都可能无限等待。
5. **无限制 `gather()`。** 任务、连接和内存会同步膨胀；用信号量或有界队列限流。
6. **吞掉 `CancelledError`。** 上游无法真正取消，资源泄漏与幽灵任务会出现。
7. **线程中随意修改全局状态。** 共享内存需要锁，但更好的设计往往是避免共享可变状态。
8. **为小任务频繁创建进程。** 启动与序列化开销可能超过计算收益；先基准测试。
9. **只测平均耗时。** 并发服务还要观察 P95/P99 延迟、超时率、排队长度、线程/进程池饱和度和取消是否生效。
10. **将并发当作性能保证。** 下游限流、数据库连接池、CPU、内存、网络和锁都会成为瓶颈；性能结论必须来自压测。

## 12. 建议的四周学习路径

### 第 1 周：理解执行模型

- 写一个顺序版与 `asyncio.gather()` 版的定时器程序，比较耗时。
- 修改示例，将 `asyncio.sleep()` 换成 `time.sleep()`，观察事件循环为何卡住。
- 用自己的话解释“协作式调度”和“抢占式线程调度”的区别。

### 第 2 周：掌握 asyncio 的可靠性

- 完成带 `TaskGroup` 的并行查询示例。
- 为远端调用增加总超时、取消和 `finally` 清理。
- 使用 `Semaphore` 和有界 `Queue` 写一个限并发下载模拟器。

### 第 3 周：掌握线程与进程边界

- 将一个阻塞函数分别直接调用和用 `to_thread()` 调用，观察异步请求是否被阻塞。
- 使用 `ThreadPoolExecutor` 批量处理 I/O 模拟任务，并让其中一个任务抛异常，确认异常能被收集。
- 使用 `ProcessPoolExecutor` 跑 CPU 计算，比较串行、线程、进程的耗时。测量时增加足够工作量，并记录机器核数。

### 第 4 周：做一个小型 LLM 服务练习

- 用任意 Web 框架实现一个请求入口，内部并行“权限校验”和“检索”。
- 给每个外部调用加超时、并发上限和结构化日志。
- 模拟客户端取消，验证后台任务、连接和队列槽位能够释放。
- 写测试覆盖成功、超时、权限失败、下游异常、取消、队列满这六类场景。

## 13. 自检题

完成学习后，尝试不看答案回答：

1. 为什么两个 `await` 连续写通常是串行，而 `gather()` 可以并发？
2. 为什么一个 `async def` 中的纯 Python 大循环仍会让其他协程“卡住”？
3. 一个同步模型 SDK 放进 FastAPI 的异步路由会有什么后果？怎样最小化改造？
4. 为什么线程能帮助阻塞 I/O，却通常不能加速纯 Python CPU 计算？
5. 进程池为什么要求函数和参数可序列化？Windows 为什么需要 `__main__` 保护？
6. 并发上限、超时、取消、背压各自防止哪一种故障？
7. 一个创建工单的工具调用超时后，为什么不能盲目重试？

如果这些问题能结合实际服务链路回答，而不只是背定义，你就已经具备了大模型应用开发所需的并发基础。

## 14. 推荐查阅的官方资料

- Python 官方文档：`asyncio`、`concurrent.futures`、`threading`、`multiprocessing`。
- Python 官方文档：异常、上下文管理器、队列和日志模块。
- 所使用 HTTP、数据库、模型 SDK 的并发与取消文档。尤其要确认客户端是否异步、连接池如何关闭、超时分哪几层、取消是否会中断底层请求。

学习时请始终保留一个习惯：先识别等待点和资源边界，再决定并发模型；写完后用超时、取消、限流、日志和压测验证，而不是只看一次正常输出。
