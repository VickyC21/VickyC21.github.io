---
layout: post
title: "Python 服务代码工程化讲义：可读、可维护、可测试"
date: 2026-08-17 10:00:00 +0800
categories: [Python基础]
tags: [Python, FastAPI, pytest, 软件工程, 服务开发]
---

# Python 服务代码工程化讲义：可读、可维护、可测试

> 面向刚开始做大模型应用或后端服务的 Python 初学者。目标是把“能跑的一段脚本”组织为职责清楚、便于修改、能自动验证的小型服务。

大模型工程师不只是调用模型 API。模型请求、Prompt 组装、权限校验、数据库读写、重试、日志、评测和异常处理，最终都会落在服务代码里。若把一切堆进单个脚本，需求变化后往往会快速失控：改一处会影响多处，报错难定位，也很难写测试。

本文用“订单支付服务”贯穿示例。它不是大模型项目，但把“订单”替换成“会话”，把“仓储”替换成“模型客户端”，同样的原则仍然成立。

---

## 1. 三个判断标准

### 可读

刚接手项目的人应能快速找到：请求从哪里进入、业务规则在哪里、数据从哪里读取和保存、失败时返回什么。可读性主要来自准确的名称、短而单一的函数、清楚的调用路径，而不是文件数量或复杂模式。

### 可维护

例如新增规则“已取消订单不能支付”，理想状态是只修改业务规则附近的代码，并找到相邻测试。推荐依赖方向：

~~~text
HTTP 请求 -> 路由层 -> 服务层 -> 仓储层 -> 数据库
                  |
                  -> 领域模型 / 业务规则
~~~

路由层可以调用服务层，服务层可以调用仓储层；领域模型不应导入 FastAPI，仓储层不应导入路由。

### 可测试

不启动 Web 服务、不连接真实数据库、不调用付费模型 API，也能验证大部分业务规则。关键是隔离副作用：数据库、网络、文件、当前时间和随机数都放在边缘；核心规则尽量成为普通的输入到输出。

### 不要过度工程化

小项目不需要一开始拆成几十个模块。出现下列信号再拆分：一个函数同时处理 HTTP、校验、业务判断、SQL、外部 API 和日志；同一规则在多个接口复制；想测试一段逻辑却必须启动数据库；函数名需要用“和”连接多个动作。

---

## 2. 单脚本为什么会失控

下面的代码能运行，但会越来越难维护：

~~~python
@app.post("/orders/{order_id}/pay")
def pay_order(order_id: int):
    connection = sqlite3.connect("app.db")
    row = connection.execute(
        "SELECT status FROM orders WHERE id = ?", (order_id,)
    ).fetchone()
    if row is None:
        raise HTTPException(404, "订单不存在")
    if row[0] == "cancelled":
        raise HTTPException(409, "已取消订单不能支付")
    connection.execute(
        "UPDATE orders SET status = 'paid' WHERE id = ?", (order_id,)
    )
    connection.commit()
~~~

一个函数同时承担 HTTP 处理、SQL、连接管理、业务规则和错误翻译。后果是：无法脱离数据库测试“取消订单不可支付”；其他调用方式无法复用；资源释放容易遗漏；改接口协议会影响核心业务。

---

## 3. 从够用的目录结构开始

~~~text
app/
├── main.py                 # 应用装配与入口
├── api/orders.py           # HTTP 路由
├── domain/models.py        # 业务数据和状态
├── domain/exceptions.py    # 可预期业务异常
├── services/order_service.py
├── repositories/order_repository.py
├── schemas/orders.py       # API 输入和输出模型
└── core/config.py          # 配置

tests/
├── unit/test_order_service.py
└── integration/test_orders_api.py
~~~

API 层只理解 HTTP；服务层只理解业务；仓储层只理解存取；领域模型表达稳定的业务数据；测试从调用者角度证明行为。目录名称可以变，职责边界不能混乱。

---

## 4. 用模型、类型和异常表达业务

~~~python
# app/domain/models.py
"""订单领域的核心数据结构与状态定义。"""

from dataclasses import dataclass
from enum import StrEnum


class OrderStatus(StrEnum):
    """订单在支付流程中的状态。"""

    PENDING = "pending"
    PAID = "paid"
    CANCELLED = "cancelled"


@dataclass
class Order:
    """表示一个订单。

    Args:
        id: 订单唯一标识。
        amount: 订单金额，单位为分。
        status: 当前订单状态。
    """

    id: int
    amount: int
    status: OrderStatus
~~~

金额用分的整数表示，避免浮点精度问题；状态使用枚举，避免随手写错字符串。函数参数、返回值、容器元素和外部边界应优先补类型。不要因为数据“通常是 JSON”就把它写成 Any，特别是模型输出。

服务层先用业务语言表达失败，不要直接依赖 HTTP：

~~~python
# app/domain/exceptions.py
"""订单领域中可预期的业务异常。"""


class OrderNotFoundError(Exception):
    """指定订单不存在时抛出。"""


class OrderAlreadyPaidError(Exception):
    """重复支付时抛出。"""


class OrderNotPayableError(Exception):
    """订单状态不允许支付时抛出。"""
~~~

这样服务既可被 HTTP 接口使用，也可被命令行、消息消费者或定时任务使用。最外层再决定是返回 404、409、重试还是告警。

---

## 5. 仓储层与服务层

服务只需要“按 ID 找订单”和“保存订单”，无需知道底层是 PostgreSQL、Redis、SQLite 还是字典：

~~~python
# app/repositories/order_repository.py
"""订单数据访问边界及内存实现。"""

from typing import Protocol

from app.domain.models import Order


class OrderRepository(Protocol):
    """订单存储需要提供的最小能力。"""

    def get_by_id(self, order_id: int) -> Order | None:
        """按 ID 查询，不存在时返回 None。"""

    def save(self, order: Order) -> None:
        """保存订单最新状态。"""


class InMemoryOrderRepository:
    """用于单元测试和本地演示的内存仓储。"""

    def __init__(self, initial_orders: list[Order] | None = None) -> None:
        self._orders = {order.id: order for order in initial_orders or []}

    def get_by_id(self, order_id: int) -> Order | None:
        return self._orders.get(order_id)

    def save(self, order: Order) -> None:
        self._orders[order.id] = order
~~~

服务层把规则写成可直接阅读的业务流程：

~~~python
# app/services/order_service.py
"""订单支付用例的业务流程实现。"""

from app.domain.exceptions import (
    OrderAlreadyPaidError,
    OrderNotFoundError,
    OrderNotPayableError,
)
from app.domain.models import Order, OrderStatus
from app.repositories.order_repository import OrderRepository


class OrderService:
    """处理订单相关业务用例。"""

    def __init__(self, repository: OrderRepository) -> None:
        self._repository = repository

    def pay(self, order_id: int) -> Order:
        """将待支付订单更新为已支付状态。"""
        order = self._repository.get_by_id(order_id)
        if order is None:
            raise OrderNotFoundError()
        if order.status is OrderStatus.PAID:
            raise OrderAlreadyPaidError()
        if order.status is not OrderStatus.PENDING:
            raise OrderNotPayableError("订单当前不可支付")

        order.status = OrderStatus.PAID
        self._repository.save(order)
        return order
~~~

现在流程清楚：找订单，判断状态，更新，保存。没有 SQL、没有 HTTP、没有框架对象，因此易读且易测。

---

## 6. API 层与依赖注入

~~~python
# app/api/orders.py
"""订单 HTTP 路由及异常映射。"""

from fastapi import APIRouter, Depends, HTTPException

from app.domain.exceptions import (
    OrderAlreadyPaidError,
    OrderNotFoundError,
    OrderNotPayableError,
)
from app.services.order_service import OrderService

router = APIRouter(prefix="/orders", tags=["orders"])


def get_order_service() -> OrderService:
    """提供已装配依赖的订单服务。"""
    raise NotImplementedError


@router.post("/{order_id}/pay")
def pay_order(
    order_id: int,
    service: OrderService = Depends(get_order_service),
) -> dict[str, object]:
    """处理支付订单请求。"""
    try:
        order = service.pay(order_id)
    except OrderNotFoundError as error:
        raise HTTPException(404, "订单不存在") from error
    except (OrderAlreadyPaidError, OrderNotPayableError) as error:
        raise HTTPException(409, str(error) or "订单当前不可支付") from error

    return {"id": order.id, "amount": order.amount, "status": order.status}
~~~

路由函数只做 HTTP 翻译。Depends 表示路由不自己创建服务，而是声明所需依赖；测试时可替换为内存仓储，生产时注入真实数据库。应用入口也应只负责拼装路由和配置。

---

## 7. 配置、日志和错误分类

模型名称、超时、数据库地址和密钥会随开发、测试、生产环境变化。代码定义配置和校验，环境变量提供值，业务代码只使用已校验配置。密钥绝不能提交仓库，也不要记录到日志。

~~~python
from pydantic_settings import BaseSettings


class Settings(BaseSettings):
    """服务运行配置。"""

    database_url: str
    request_timeout_seconds: float = 10.0
    llm_model: str = "gpt-4.1-mini"
~~~

服务使用 logging，而不是长期依赖 print。记录关联 ID、关键动作、耗时、异常堆栈和不敏感上下文。logger.exception 适合在 except 块中使用，它会记录堆栈。

| 类型 | 例子 | 处理方式 |
| --- | --- | --- |
| 输入问题 | 字段缺失、格式错误 | 返回 400 或 422 |
| 业务冲突 | 已支付、无权限 | 返回 403、404、409 |
| 基础设施故障 | 数据库断开、模型 API 超时 | 记录日志，返回通用 5xx |

不要无差别捕获 Exception 后返回“失败”。这会吞掉堆栈，调用方也无法区分请求写错和服务坏了。只捕获你知道如何处理的异常。

---

## 8. 测试：从手动点击到自动验证

测试投入可按金字塔理解：大量快速稳定的单元测试、少量 API 集成测试、极少量端到端测试。

单元测试不启动 FastAPI，也不连接数据库：

~~~python
# tests/unit/test_order_service.py
"""订单支付服务的单元测试。"""

import pytest

from app.domain.exceptions import OrderAlreadyPaidError, OrderNotPayableError
from app.domain.models import Order, OrderStatus
from app.repositories.order_repository import InMemoryOrderRepository
from app.services.order_service import OrderService


def make_service(order: Order) -> OrderService:
    """为测试构造服务。"""
    return OrderService(InMemoryOrderRepository([order]))


def test_待支付订单可以支付() -> None:
    service = make_service(Order(1, 1000, OrderStatus.PENDING))

    result = service.pay(1)

    assert result.status is OrderStatus.PAID


def test_已支付订单不能重复支付() -> None:
    service = make_service(Order(1, 1000, OrderStatus.PAID))

    with pytest.raises(OrderAlreadyPaidError):
        service.pay(1)


def test_已取消订单不能支付() -> None:
    service = make_service(Order(1, 1000, OrderStatus.CANCELLED))

    with pytest.raises(OrderNotPayableError):
        service.pay(1)
~~~

测试名称应描述行为，而不是内部实现。优先验证“取消订单不能支付”，而非“save 被调用一次”。

每个业务动作至少考虑：正常值、空值或不存在、非法值、边界值、状态转换、外部失败、权限与幂等性。LLM 服务还要覆盖非法 JSON、字段缺失、Prompt 注入、模型超时、流式中断、重试是否重复执行工具、日志是否泄漏敏感数据。

测试替身的含义：Fake 是可工作的简化实现，例如内存仓储；Stub 返回预设数据，例如固定模型响应；Mock 验证交互，例如是否发送一次消息。初学阶段优先使用 Fake，只有“是否发生调用”本身就是业务要求时才重点验证 Mock。

---

## 9. 异步、超时、重试和幂等性

async 适合模型 API、异步数据库等等待型 I/O；它不会让 CPU 密集计算自动变快。

~~~python
async def fetch_context_and_answer(question, retriever, model_client):
    """检索上下文后异步调用模型。"""
    documents = await retriever.search(question)
    prompt = build_prompt(question, documents)
    return await model_client.generate(prompt)
~~~

常见误区：在 async def 中调用同步且耗时的客户端，仍会阻塞事件循环；在 async def 中做大文件解析或大量计算，仍要考虑进程池、任务队列或独立 worker；没有 await 时，普通 def 往往更简单。

对每个外部调用明确：超时上限、哪些错误可重试、重试次数与退避、客户端断开后的取消行为，以及重试是否安全。创建订单、扣费、调用写入型工具必须先设计幂等键；没有幂等保护的重试可能造成重复扣费。

---

## 10. LLM 服务的额外边界

模型输出和检索文本都不能被默认信任。模型输出应在边界处校验：

~~~python
from pydantic import BaseModel, ValidationError


class ExtractionResult(BaseModel):
    """模型应返回的结构化抽取结果。"""

    name: str
    category: str
    confidence: float


def parse_model_output(raw_json: str) -> ExtractionResult:
    """校验模型输出，并在不合法时明确失败。"""
    try:
        return ExtractionResult.model_validate_json(raw_json)
    except ValidationError as error:
        raise ValueError("模型返回的结构不合法") from error
~~~

模型建议调用工具，不代表它有权限执行。工具层必须自行做参数 Schema 校验、用户权限和资源归属校验、超时与幂等控制、审计日志；高风险操作必须人工审批。模型负责建议，业务系统负责授权和执行。

---

## 11. 从脚本走向服务的重构路线

不要一次性“重构优雅”，按可验证的小步推进：

1. 给最重要成功场景和失败场景各补一条测试。
2. 抽出纯函数，例如参数规范化、Prompt 组装、状态判断。
3. 把 HTTP、SQL、模型调用集中到客户端或仓储。
4. 建立服务函数或服务类，用业务名承载完整用例。
5. 让路由或脚本只处理输入输出。
6. 区分业务异常、验证错误、基础设施错误。
7. 补集成测试、日志和必要指标。

每完成一步，运行对应测试。重构目标是“行为不变但结构更清楚”，不是一次性追求理想架构。

---

## 12. 四周学习计划与自查

第 1 周：学习函数、类、模块、包、异常、dataclass、Enum、类型标注；把百行脚本拆为输入解析、业务函数、输出；为纯函数写五条 pytest 测试。

第 2 周：用 FastAPI 写一个 CRUD 或任务提交接口；让路由只处理 HTTP，服务层只处理业务；使用 Pydantic 校验请求和响应。

第 3 周：区分单元与集成测试；使用 Fake 替换数据库或模型客户端；加入环境变量配置、关键日志和超时、空值、非法值、边界测试。

第 4 周：封装模型客户端；校验结构化模型输出；为 Prompt 组装写测试；使用假客户端测试超时、非法 JSON、缺字段；记录请求 ID、模型名、耗时、token 用量和失败原因。

每次提交前检查：

- [ ] 路由是否只处理 HTTP 输入、输出和依赖注入？
- [ ] 业务规则是否在服务或领域函数中？
- [ ] 数据库、模型 API 是否集中在边缘模块？
- [ ] 正常、空值、非法值、边界、失败路径是否被考虑？
- [ ] 不启动外部服务能否测试核心规则？
- [ ] 是否避免把密钥和敏感数据写入代码或日志？
- [ ] 是否实际运行了相关测试？

## 结语

可读、可维护、可测试不是一次重构获得的属性，而是每个小改动持续做出的选择：一个函数只承担一个主要责任；业务规则远离 HTTP 和数据库细节；不可靠外部输入在边界处校验；测试描述用户真正关心的行为；抽象只解决已经出现的复杂度。

当你能按这种方式完成一个小服务，即使暂时不使用复杂框架，也已经具备大模型工程中最基础、最可迁移的服务开发能力。下一步可把订单示例替换为文档问答、结构化信息抽取或工具调用工作流，继续练习同一套边界与验证方法。