---
layout: post
title: "Python 问题定位讲义：日志、断点与性能分析"
date: 2026-08-17 01:00:00 +0800
categories: [Python基础]
tags: [Python, 调试, 日志, 性能分析, LLM]
---

# Python 问题定位讲义：日志、断点与性能分析

本讲义面向刚进入大模型开发领域的 Python 初学者。目标不是背下命令，而是建立一条可靠的排障路径：**先确认现象，再用证据缩小范围，最后验证修复是否真的解决问题。**

大模型服务的“问题”不只意味着报错。请求一直转圈、回答为空、偶发超时、内存持续升高、同一 Prompt 结果不一致，都需要定位。三类工具回答不同的问题：

| 工具 | 核心问题 | 优点 | 局限 |
| --- | --- | --- | --- |
| 日志 | 发生了什么、何时发生、影响了谁 | 保留现场，适合线上和异步场景 | 看不到所有局部变量 |
| 断点 | 这一行执行时变量为何变成这样 | 可逐行观察变量和调用栈 | 不能在生产环境暂停服务 |
| 性能分析 | 时间、CPU、内存到底花在哪里 | 用数据找热点 | 不直接判断业务正确性 |

请记住：**先复现，后定位；先测量，后优化；一次只验证一个假设。**

## 1. 先给问题分类

收到“服务不对劲”时，先分类，工具选择会清晰很多。

1. **确定性报错**：每次运行都抛出同一个异常，例如 KeyError、TypeError、连接被拒绝。先看异常栈和日志。
2. **结果错误**：程序没有崩，但输出不符合预期，例如检索为空、模型 JSON 缺字段。先记录输入和关键中间结果，必要时打断点。
3. **偶发错误**：只在高并发、特定数据或某些时段出现。优先补充请求 ID、重试次数、外部依赖耗时等日志。
4. **变慢或超时**：用户感觉“卡”，或 P95/P99 延迟升高。先测端到端和分段耗时，再做性能剖析。
5. **资源异常**：CPU 满载、内存上涨、GPU OOM、文件描述符耗尽。先记录资源指标和请求特征，再分析。

### 1.1 最小复现

“有时不对”还不可验证。你要把它缩小成最短、可重复运行的代码。

~~~~python
def normalize_query(query: str) -> str:
    """清理用户的检索问题。"""
    return query.strip().lower()


# 用能触发问题的最小输入复现。
normalize_query(None)
~~~~

运行会稳定得到：

~~~~text
AttributeError: 'NoneType' object has no attribute 'strip'
~~~~

现在才能提出可验证修复：query 为空时抛出清晰的 ValueError，或保证调用方只传字符串。不要一开始就在很多地方加 try/except，这通常只会隐藏根因。

### 1.2 排障记录模板

- 现象：什么时间、什么输入、什么环境发生了什么。
- 影响：一个请求、一类用户还是所有服务。
- 复现：能否稳定复现；不能时的共同条件。
- 证据：异常栈、请求 ID、日志片段、性能报告、版本号。
- 假设：最可能的 1 到 3 个原因。
- 验证：怎样证伪或证实每个假设。
- 结论：根因、修复、回归测试和残余风险。

## 2. 日志：为未来的自己保留现场

print() 适合临时学习，但没有等级、时间、模块、统一格式和可靠输出。服务程序使用标准库 logging。

### 2.1 日志等级

- DEBUG：开发排查细节，例如清洗后的查询词、分块数、缓存命中。
- INFO：正常但值得记录的过程，例如请求开始、任务完成。
- WARNING：还能继续运行，但有异常迹象，例如重试后成功。
- ERROR：当前操作失败，例如模型接口返回 500、解析失败。
- CRITICAL：系统级严重故障，例如关键配置丢失。

不要把每件事都记成 INFO，也不要把可预期分支记成 ERROR。等级不准确会让告警失去意义。

### 2.2 可直接运行的基础配置

~~~~python
import logging


def configure_logging() -> None:
    """配置应用基础日志格式。

    仅在应用启动入口调用一次。业务模块只获取 logger，
    不要重复调用 basicConfig。
    """
    logging.basicConfig(
        level=logging.INFO,
        format=(
            "%(asctime)s | %(levelname)-8s | %(name)s | "
            "%(filename)s:%(lineno)d | %(message)s"
        ),
    )


logger = logging.getLogger(__name__)


def divide(total: int, count: int) -> float:
    """计算平均值，失败时记录完整异常栈。

    Args:
        total: 总量。
        count: 份数。

    Returns:
        每份平均值。

    Raises:
        ZeroDivisionError: count 为 0 时抛出。
    """
    logger.info("开始计算平均值：total=%s, count=%s", total, count)
    try:
        result = total / count
    except ZeroDivisionError:
        logger.exception("计算平均值失败")
        raise

    logger.debug("平均值计算完成：result=%s", result)
    return result


if __name__ == "__main__":
    configure_logging()
    divide(10, 0)
~~~~

logger.exception() 只能在 except 中使用，它会附带 traceback。相比只写“失败”，异常栈能指出失败的文件、行号和调用链。

推荐使用延迟格式化：

~~~~python
logger.debug("检索到 %s 个候选文档", len(documents))
~~~~

不要在高频路径写：

~~~~python
logger.debug(f"检索到 {len(documents)} 个候选文档")
~~~~

前一种在 DEBUG 被关闭时不必先构造消息。

### 2.3 LLM 请求日志示例

每次大模型请求都应可追踪。以下例子模拟了 request_id 和耗时记录。

~~~~python
import logging
import time
import uuid


logger = logging.getLogger(__name__)


def answer_question(question: str, model_name: str) -> str:
    """模拟一次可观测的模型问答请求。

    Args:
        question: 用户问题。
        model_name: 模型名称。

    Returns:
        模拟回答。
    """
    request_id = uuid.uuid4().hex
    started_at = time.perf_counter()
    logger.info(
        "LLM 请求开始 request_id=%s model=%s question_length=%s",
        request_id,
        model_name,
        len(question),
    )

    try:
        # 真实项目中，这里调用模型 API 或本地推理服务。
        response = f"关于“{question}”的模拟回答"
        elapsed_ms = (time.perf_counter() - started_at) * 1000
        logger.info(
            "LLM 请求成功 request_id=%s model=%s elapsed_ms=%.1f",
            request_id,
            model_name,
            elapsed_ms,
        )
        return response
    except Exception:
        elapsed_ms = (time.perf_counter() - started_at) * 1000
        logger.exception(
            "LLM 请求失败 request_id=%s model=%s elapsed_ms=%.1f",
            request_id,
            model_name,
            elapsed_ms,
        )
        raise
~~~~

线上 Prompt 可能含个人信息、业务机密或 API Key，因此不要默认记录完整问题和回答。优先记录长度、哈希、模型版本、耗时和错误类别；需要样本时先脱敏。

推荐字段：

- request_id 或 trace_id：把 API、检索、模型、工具调用串成同一请求。
- model、model_version、prompt_version：确认使用了哪套模型和提示词。
- retrieved_count、rerank_count：判断检索链路是否异常。
- input_tokens、output_tokens、cost：定位异常用量与成本。
- elapsed_ms、ttft_ms：区分总延迟与首 token 延迟。
- error_type、retry_count：判断是否为暂时性外部故障。

### 2.4 文件日志滚动

长期运行的服务不能让日志文件无限变大。

~~~~python
import logging
from logging.handlers import RotatingFileHandler


def configure_file_logging() -> None:
    """配置滚动文件日志。"""
    handler = RotatingFileHandler(
        "app.log",
        maxBytes=10 * 1024 * 1024,
        backupCount=5,
        encoding="utf-8",
    )
    handler.setFormatter(
        logging.Formatter(
            "%(asctime)s | %(levelname)s | %(name)s | %(message)s"
        )
    )
    root_logger = logging.getLogger()
    root_logger.setLevel(logging.INFO)
    root_logger.addHandler(handler)
~~~~

容器服务常输出到标准输出，由平台收集。无论去向如何，都要保持格式稳定、字段可查询、敏感信息不泄露、保留周期明确。

### 2.5 日志常见误区

1. 记录 API Key、Cookie、完整敏感 Prompt 或原始用户数据。
2. 捕获后静默吞掉异常，例如 except Exception: return None。
3. 错误日志没有异常栈。
4. 循环内输出大量日志，拖慢服务且淹没关键信息。
5. 没有 request_id，导致并发日志无法关联。
6. 只看最后一条 ERROR；应沿同一 request_id 向前找第一个异常信号。

## 3. 断点：暂停程序，观察真实状态

日志记录过程，断点让你在某一刻检查变量和调用栈。初学者常用 IDE 图形断点与内置 breakpoint()。

### 3.1 VS Code 图形断点

1. 安装 Python 扩展，并选择项目正确的虚拟环境解释器。
2. 点击行号左侧设置红点。
3. 从“运行和调试”启动当前文件，或按 F5。
4. 程序暂停后检查“变量”“监视”“调用堆栈”。
5. 使用“单步跳过”“单步进入”“继续”。

优先看：输入类型是否正确、字典键是否存在、列表是否为空、分支条件为何为真或假、调用当前函数的是谁，以及异常前最后一次状态变化的位置。

### 3.2 条件断点、命中计数和日志断点

循环有一万次时，普通断点不实用：

- **条件断点**：只在 score < 0 或 document_id == "bad-17" 时停下。
- **命中计数**：第 100 次循环才暂停，观察累积状态。
- **日志断点**：输出变量但不暂停，适合不希望改变程序时序的场景。

条件表达式应无副作用，不要在其中调用会修改状态的函数。

### 3.3 使用 breakpoint() 与 pdb

~~~~python
def parse_score(raw_score: str) -> float:
    """把模型返回的评分文本转换为浮点数。

    Args:
        raw_score: 原始评分文本。

    Returns:
        清理后的分数。
    """
    cleaned_score = raw_score.strip()
    breakpoint()
    return float(cleaned_score)
~~~~

运行到这里会进入 pdb。常用命令：

| 命令 | 作用 |
| --- | --- |
| p variable | 打印变量 |
| pp variable | 易读地打印复杂对象 |
| l | 显示附近源码 |
| n | 下一行，不进入函数 |
| s | 下一行，进入函数 |
| c | 继续到下一个断点 |
| w | 查看调用栈 |
| u / d | 在调用栈中上移 / 下移 |
| q | 退出调试器并终止程序 |

若模型返回 "N/A"：

~~~~text
(Pdb) p raw_score
'N/A'
(Pdb) p cleaned_score
'N/A'
(Pdb) n
ValueError: could not convert string to float: 'N/A'
~~~~

结论不是 float 有问题，而是上游输出违反了协议。修复应是定义结构化输出、校验字段并为拒答建立明确状态，而不是忽略 ValueError。

### 3.4 调用栈和边界校验

~~~~python
def build_prompt(user_profile: dict) -> str:
    """使用用户资料构建提示词。"""
    return f"你好，{user_profile['nickname']}！"


def handle_request(payload: dict) -> str:
    """从请求载荷读取资料。"""
    return build_prompt(payload["profile"])
~~~~

在 build_prompt 中断住后，使用 w 看调用栈。如果 profile 没有 nickname，就要检查无效数据是从哪里进入的。外部输入边界（HTTP 请求、消息队列、文件解析）应尽早校验，底层函数也保留必要的防御性检查。

断点的边界：

- 不要把 breakpoint() 留在提交代码中。
- 不要在线上服务中暂停进程。
- 异步、多线程和多进程中，断点可能改变调度顺序，竞态问题可能暂时消失。
- 断点只证明这一次运行的状态，修复仍要靠复现或测试验证。

## 4. 性能分析：用数据找到真正的慢点

优化前先定义指标：

- **墙钟时间**：用户实际等待多久。
- **CPU 时间**：进程真正占用 CPU 的时间。
- **吞吐量**：单位时间完成多少请求或 token。
- **P50/P95/P99 延迟**：典型、较慢、极端慢请求。
- **内存占用和增长**：内存是否持续无法回落。

API 与 LLM 服务慢，常是网络、数据库、向量检索、模型推理、串行工具调用或重试，不一定是 Python CPU 慢。因此先分段计时。

### 4.1 time.perf_counter() 分段计时

~~~~python
import time


def search_documents(query: str) -> list[str]:
    """模拟检索。"""
    time.sleep(0.08)
    return [f"{query} 的文档 1", f"{query} 的文档 2"]


def generate_answer(documents: list[str]) -> str:
    """模拟生成。"""
    time.sleep(0.35)
    return "模拟回答"


def answer(query: str) -> str:
    """分别记录检索、生成和总耗时。"""
    total_started_at = time.perf_counter()

    retrieval_started_at = time.perf_counter()
    documents = search_documents(query)
    retrieval_ms = (time.perf_counter() - retrieval_started_at) * 1000

    generation_started_at = time.perf_counter()
    result = generate_answer(documents)
    generation_ms = (time.perf_counter() - generation_started_at) * 1000

    total_ms = (time.perf_counter() - total_started_at) * 1000
    print(
        f"retrieval={retrieval_ms:.1f}ms, "
        f"generation={generation_ms:.1f}ms, total={total_ms:.1f}ms"
    )
    return result
~~~~

如果总耗时 450ms、生成占 350ms，优化字符串拼接没有价值。应优先考虑模型、输出 token 上限、流式首 token、缓存或模型服务并发。

### 4.2 timeit 比较纯 Python 片段

~~~~python
from timeit import timeit


list_time = timeit(
    "sum([number * number for number in range(10_000)])",
    number=1_000,
)
generator_time = timeit(
    "sum(number * number for number in range(10_000))",
    number=1_000,
)

print(f"列表推导式：{list_time:.4f}s")
print(f"生成器表达式：{generator_time:.4f}s")
~~~~

timeit 适合短小、确定的本地代码，不适合网络 API、数据库或模型调用；外部依赖的波动会掩盖结论。

### 4.3 cProfile 找 CPU 热点

~~~~python
def fibonacci(number: int) -> int:
    """用故意低效的递归算法计算斐波那契数。"""
    if number < 2:
        return number
    return fibonacci(number - 1) + fibonacci(number - 2)


if __name__ == "__main__":
    print(fibonacci(30))
~~~~

运行：

~~~~powershell
python -m cProfile -s cumulative slow_example.py
~~~~

重点关注：

- ncalls：调用次数。
- tottime：函数自身耗时，不含子函数。
- cumtime：函数和子调用总耗时。
- percall：单次平均耗时。

递归 fibonacci 会有海量重复调用。改成迭代能降低时间复杂度：

~~~~python
def fibonacci_fast(number: int) -> int:
    """使用迭代计算斐波那契数。

    Args:
        number: 非负整数序号。

    Returns:
        第 number 个斐波那契数。

    Raises:
        ValueError: number 小于 0 时抛出。
    """
    if number < 0:
        raise ValueError("number 必须是非负整数")

    previous, current = 0, 1
    for _ in range(number):
        previous, current = current, previous + current
    return previous
~~~~

也可以在代码中收集报告：

~~~~python
import cProfile
import pstats


profiler = cProfile.Profile()
profiler.enable()
fibonacci_fast(10_000)
profiler.disable()

statistics = pstats.Stats(profiler).sort_stats("cumulative")
statistics.print_stats(20)
~~~~

cProfile 告诉你哪个函数慢。确认热点确实在某函数内部后，再考虑 line_profiler 找哪一行慢；不要一开始就安装很多工具。

### 4.4 tracemalloc 分析内存增长

~~~~python
import tracemalloc


cache: list[str] = []


def process_request(index: int) -> None:
    """故意永久缓存请求数据，模拟内存增长。"""
    cache.append("x" * 100_000 + str(index))


tracemalloc.start()
before_snapshot = tracemalloc.take_snapshot()

for index in range(100):
    process_request(index)

after_snapshot = tracemalloc.take_snapshot()
for statistic in after_snapshot.compare_to(before_snapshot, "lineno")[:5]:
    print(statistic)
~~~~

输出会指向 cache.append 所在行。不要简单“定时清空缓存”；先确认缓存是否需要、是否需要容量上限或 TTL、是否有并发安全要求。tracemalloc 主要观察 Python 分配，对 NumPy、PyTorch、原生扩展或 GPU 内存未必完整，应再看对应运行时指标。

### 4.5 生产性能定位顺序

1. 明确慢的是端到端、TTFT、某接口还是 CPU。
2. 确认影响范围：所有请求、某模型、长文本还是高并发。
3. 日志记录分段耗时、输入长度、检索文档数和输出 token 数。
4. CPU 高时用 cProfile；内存持续增长时用 tracemalloc。
5. 外部调用慢时检查重试、连接池、超时、查询计划、队列和网络指标。
6. 只优化测量证实的热点，并以同一负载重新测量。

## 5. 三种工具如何配合

假设知识库问答服务部分请求报错、且高峰期变慢。

### 5.1 日志先确定报错边界

~~~~python
import logging


logger = logging.getLogger(__name__)


def extract_answer(payload: dict) -> str:
    """读取模型输出中的 answer 字段。"""
    try:
        answer = payload["answer"]
        if not isinstance(answer, str) or not answer.strip():
            raise ValueError("answer 必须是非空字符串")
        return answer
    except (KeyError, ValueError, TypeError):
        logger.exception(
            "模型输出不合法 keys=%s payload_type=%s",
            list(payload) if isinstance(payload, dict) else None,
            type(payload).__name__,
        )
        raise
~~~~

若日志显示 keys=['reasoning']，问题是模型没有返回 answer，而不是前端渲染失败。保留安全脱敏样本，检查 Prompt、JSON Schema、模型版本和解析重试。

### 5.2 断点核对真实数据

在 answer = payload["answer"] 前停住。若 API 实际返回：

~~~~python
{"reasoning": "无法从上下文找到答案"}
~~~~

可验证结论是：协议没有覆盖拒答路径。修复可以要求始终返回 {"answer": "...", "status": "answered|not_found"}，并用 Pydantic 或等价校验器校验，而不是假设每个请求都有答案。

### 5.3 性能报告决定优化方向

若日志显示检索只要 80ms、模型生成 4 秒，且 cProfile 显示 Python CPU 很低，Python 业务代码不是热点。优先检查输出 token 是否过多、模型服务是否排队、超时是否触发重复调用、是否能流式返回、是否可缓存检索结果或回答。

## 6. 练习路线

每个练习保留“故障代码、定位过程、修复代码、验证结果”。

1. 写 JSON 读取程序，分别记录文件不存在、JSON 格式错误和字段缺失。
2. 写字符串转整数列表函数，混入空字符串和 "N/A"，用条件断点只停在非法项。
3. 模拟“检索 + 模型调用 + 后处理”，分别记录耗时。
4. 对递归 fibonacci 运行 cProfile，改为迭代后再次比较。
5. 用 tracemalloc 定位不断增长的全局缓存，加入容量上限后比较快照。
6. 为假的 LLM 请求加入 request_id、超时日志、输出校验和分段耗时，构造正常、非法 JSON、慢响应三类输入。

## 7. 上线前检查清单

- [ ] 失败路径记录了足够上下文和异常栈。
- [ ] 日志没有 API Key、密码、Cookie、完整敏感 Prompt 或未经脱敏的个人数据。
- [ ] 可以通过请求 ID 串起一次请求。
- [ ] 常见异常有最小复现或自动化测试。
- [ ] 性能优化前后使用同一场景和指标测量。
- [ ] 已采集或计划采集 P95/P99、错误率和内存增长。
- [ ] 没有把 breakpoint()、临时调试日志或大规模 print() 留在生产路径。
- [ ] 修复后验证了正常、空值、非法值和边界输入。

## 8. 核心结论

1. 日志记录发生过什么，尤其适合线上、异步和偶发问题。
2. 断点观察这一刻的真实变量与调用栈，适合本地复现。
3. 性能分析证明时间和内存花在哪里，不能凭直觉优化。
4. 异常不是根因；沿调用链、请求 ID 和分段耗时寻找第一个异常信号。
5. 调试的终点不是“这次不报错”，而是能复现、能修复、能用测试或指标证明问题已解决。

掌握这三类工具后，再学习 OpenTelemetry、Prometheus、Sentry、PyTorch Profiler 或分布式追踪会更顺畅。它们不是替代基础能力，而是把同一套“记录、观察、测量、验证”扩展到更复杂的生产系统。

