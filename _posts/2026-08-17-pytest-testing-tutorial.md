---
layout: post
title: "从零掌握 pytest：Python 单元测试与集成测试讲义"
date: 2026-08-17 10:00:00 +0800
categories: [Python基础]
tags: [Python, pytest, 单元测试, 集成测试, 软件工程]
---

# 从零掌握 pytest：Python 单元测试与集成测试讲义

> 学习目标：你将能用 pytest 编写 Python 单元测试与集成测试，并系统覆盖正常、空值、非法值和边界场景。

测试不是“代码写完后再验收一次”，而是把程序应遵守的规则写成可以反复执行的断言。对于大模型开发工程师，模型输出解析、工具参数校验、API 服务、数据库写入、超时与重试都可能在异常输入下出错；测试能把这些风险提前变成可定位的问题。

## 1. 测试的基本结构

一条测试通常遵循 AAA 模式：

1. Arrange：准备输入、依赖和初始状态。
2. Act：调用待测函数或接口。
3. Assert：确认结果是否符合预期。

~~~python
# price.py
def calculate_discount(price: float, rate: float) -> float:
    """根据折扣率计算折后价格。"""
    return price * rate
~~~

~~~python
# test_price.py
from price import calculate_discount


def test_calculate_discount_returns_discounted_price():
    """正常价格和折扣率应得到正确结果。"""
    result = calculate_discount(100, 0.8)

    assert result == 80
~~~

assert 的意思是“这个条件必须为真”。若返回值不是 80，pytest 会标记失败，并展示预期值和实际值的差异。

## 2. 单元测试和集成测试的区别

### 2.1 单元测试

单元测试验证一个小而独立的单元，通常是函数、类方法或业务规则。它应当快速、稳定，尽量不访问真实网络、生产数据库或收费的模型 API。

例如，金额计算、输入校验、提示词拼接、模型 JSON 解析，通常适合写单元测试。

单元测试回答的问题是：

- 给定输入后，函数是否返回正确结果？
- 空值、错误类型和越界值是否按约定被拒绝？
- 关键业务分支是否被覆盖？
- 下游依赖失败时是否按约定处理？

### 2.2 集成测试

集成测试验证多个真实组件连接后能否协同工作。例如：

- FastAPI 是否接收 JSON、完成校验并返回正确响应；
- 服务层是否能写入测试数据库；
- 路由、依赖注入、序列化与错误处理是否一起生效；
- 大模型输出是否经过解析、校验后被保存。

| 场景 | 优先类型 | 原因 |
| --- | --- | --- |
| 计算金额、校验字段、拼接提示词 | 单元测试 | 快、定位准确、没有外部依赖 |
| API 请求到响应 | 集成测试 | 验证路由、校验与序列化 |
| SQL 查询、事务、唯一约束 | 集成测试 | 必须验证真实数据库行为 |
| 第三方 LLM API | Mock 单元测试为主 | 避免网络、费用、配额和随机性 |
| LLM 服务真实连通性 | 少量可选集成测试 | 只验证关键连通性 |
| 用户从前端到后端的完整流程 | 端到端测试 | 验证范围最大，数量应最少 |

不要让每个单元测试都连接数据库或网络。这样测试会慢、脆弱，失败原因也难定位。

## 3. 安装、发现规则和运行命令

Windows PowerShell 示例：

~~~powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -U pytest
~~~

常用命令：

~~~powershell
pytest                    # 运行全部测试
pytest -v                 # 显示每个测试的名称
pytest -vv                # 显示更详细的失败信息
pytest tests/test_api.py  # 只运行一个文件
pytest -k username        # 只运行名称包含 username 的测试
pytest -x                 # 第一个失败后停止
pytest --pdb              # 失败后进入调试器
~~~

pytest 默认会发现：

- 文件名以 test_ 开头或以 _test.py 结尾；
- 函数名以 test_ 开头；
- 测试类名以 Test 开头，类内方法以 test_ 开头。

推荐目录结构：

~~~text
your-project/
├── src/
│   └── app/
│       ├── __init__.py
│       └── validators.py
├── tests/
│   ├── conftest.py
│   ├── test_validators.py
│   └── integration/
│       └── test_api.py
└── pyproject.toml
~~~

小项目可以把测试放在业务代码旁边，但随着项目增长，统一使用 tests 目录更易维护。

## 4. 四类必须覆盖的测试数据

只写一条“成功用例”不叫充分测试。先把输入划分为四类：

| 类别 | 含义 | 示例 |
| --- | --- | --- |
| 正常值 | 合法且常见的输入 | 正确用户名、合法金额、正确日期 |
| 空值 | 缺失或没有内容 | None、空字符串、空列表、缺失字段 |
| 非法值 | 类型或格式不符合约定 | 负数、错误枚举、损坏 JSON、字符串代替整数 |
| 边界值 | 正好处于规则边缘 | 最小值、最大值、上限加一、下限减一 |

实用口诀：每增加一条输入规则，就分别问正常、空、非法、边界时会怎样。

## 5. 完整示例：用户名校验

需求如下：

- 必须是字符串；
- 去除首尾空格后，长度必须为 3 到 20；
- 仅允许英文、数字和下划线；
- 不允许空内容。

业务代码：

~~~python
# src/app/validators.py
import re


def validate_username(username: str) -> str:
    """校验并返回清理后的用户名。

    Args:
        username: 用户提交的原始用户名。

    Returns:
        去除首尾空白后的合法用户名。

    Raises:
        TypeError: username 不是字符串时抛出。
        ValueError: 用户名为空、长度越界或包含非法字符时抛出。
    """
    if not isinstance(username, str):
        raise TypeError("用户名必须是字符串")

    normalized = username.strip()
    if not normalized:
        raise ValueError("用户名不能为空")
    if not 3 <= len(normalized) <= 20:
        raise ValueError("用户名长度必须在 3 到 20 个字符之间")
    if not re.fullmatch(r"[A-Za-z0-9_]+", normalized):
        raise ValueError("用户名只能包含字母、数字和下划线")

    return normalized
~~~

测试代码：

~~~python
# tests/test_validators.py
import pytest

from app.validators import validate_username


def test_validate_username_returns_normalized_valid_username():
    """正常用户名应通过，并去除首尾空白。"""
    assert validate_username("  vicky_21  ") == "vicky_21"


@pytest.mark.parametrize(
    ("username", "message"),
    [
        (None, "用户名必须是字符串"),
        (123, "用户名必须是字符串"),
        ([], "用户名必须是字符串"),
    ],
)
def test_validate_username_rejects_non_string(username, message):
    """空值和错误类型不应被当作用户名接受。"""
    with pytest.raises(TypeError, match=message):
        validate_username(username)


@pytest.mark.parametrize(
    "username",
    [
        "",
        "   ",
        "ab",
        "a" * 21,
        "alice!",
        "alice space",
        "张三",
    ],
)
def test_validate_username_rejects_invalid_value(username):
    """空内容、长度越界和非法字符均应被拒绝。"""
    with pytest.raises(ValueError):
        validate_username(username)


@pytest.mark.parametrize(
    "username",
    [
        "abc",       # 最小合法长度
        "a" * 20,    # 最大合法长度
    ],
)
def test_validate_username_accepts_boundary_values(username):
    """长度边界上的合法值应通过。"""
    assert validate_username(username) == username
~~~

这一组测试的关键是：每条规则既有通过用例，也有拒绝用例；存在上下限时，要测下限、上限和越界值。

## 6. 断言与异常测试

不要只测试“没有报错”：

~~~python
def test_create_user():
    create_user("alice")
~~~

上面的测试没有确认用户是否真的创建成功。应该断言可观察结果：

~~~python
def test_create_user_returns_expected_user():
    user = create_user("alice")

    assert user.name == "alice"
    assert user.is_active is True
    assert user.id is not None
~~~

常用断言：

~~~python
assert result == expected
assert "token" in response
assert items
assert not items
assert value is None
assert 0 <= score <= 1
assert response["status"] == "ok"
~~~

浮点数应使用近似比较：

~~~python
import pytest


def test_tax_calculation():
    assert calculate_tax(19.9, 0.13) == pytest.approx(2.587)
~~~

异常测试要尽可能具体：

~~~python
import pytest


def parse_temperature(value: str) -> float:
    """把温度文本解析为摄氏温度。"""
    number = float(value)
    if not -273.15 <= number <= 1000:
        raise ValueError("温度超出允许范围")
    return number


def test_parse_temperature_rejects_non_numeric_text():
    with pytest.raises(ValueError):
        parse_temperature("warm")


def test_parse_temperature_rejects_below_absolute_zero():
    with pytest.raises(ValueError, match="温度超出允许范围"):
        parse_temperature("-274")
~~~

通常，TypeError 表示类型错误，例如函数需要字符串却收到列表；ValueError 表示类型可以接受，但值本身不合法，例如空字符串或数值越界。

## 7. 参数化：把多组同类用例写清楚

当同一规则要测试多组输入时，使用 pytest.mark.parametrize。每一行都会被 pytest 当作独立用例，因此报告能准确指出哪一组失败。

~~~python
import pytest


def is_valid_page_size(page_size: int) -> bool:
    """判断分页大小是否为 1 到 100 的整数。"""
    return (
        isinstance(page_size, int)
        and not isinstance(page_size, bool)
        and 1 <= page_size <= 100
    )


@pytest.mark.parametrize(
    ("page_size", "expected"),
    [
        (1, True),
        (50, True),
        (100, True),
        (0, False),
        (101, False),
        (-1, False),
        (None, False),
        ("10", False),
        (True, False),
    ],
)
def test_is_valid_page_size(page_size, expected):
    assert is_valid_page_size(page_size) is expected
~~~

这里专门测试 True，因为 Python 中 bool 是 int 的子类。若只检查 isinstance(page_size, int)，True 会意外通过。这是边界测试经常发现的真实问题。

给参数增加 id 后，失败报告更好读：

~~~python
@pytest.mark.parametrize(
    ("text", "expected"),
    [
        pytest.param("hello", "HELLO", id="普通英文"),
        pytest.param("", "", id="空字符串"),
    ],
)
def test_to_uppercase(text, expected):
    assert text.upper() == expected
~~~

## 8. fixture：复用准备工作，并保持测试隔离

fixture 用于准备测试数据、对象或环境。默认 function 作用域表示每个测试都有一份新资源，这是最安全的默认值。

~~~python
# tests/conftest.py
import pytest

from app.service import UserService


@pytest.fixture
def user_service():
    """为每个测试创建新的内存用户服务。"""
    return UserService()
~~~

~~~python
# tests/test_service.py
def test_create_user_increases_count(user_service):
    user_service.create("alice")

    assert user_service.count() == 1


def test_new_service_starts_empty(user_service):
    assert user_service.count() == 0
~~~

常见作用域：

| scope | 生命周期 | 使用建议 |
| --- | --- | --- |
| function | 每个测试函数 | 默认选择，隔离性最好 |
| class | 每个测试类 | 一组紧密相关的测试 |
| module | 每个测试文件 | 可共享但成本较高的资源 |
| session | 全部测试 | 极少数昂贵、不可变资源 |

不要轻易把可变对象设为 session 作用域，否则前一个测试残留的数据可能污染后一个测试。

### 用 tmp_path 测试文件

~~~python
import json


def save_config(path, config: dict) -> None:
    """把配置写入 JSON 文件。"""
    path.write_text(json.dumps(config), encoding="utf-8")


def load_config(path) -> dict:
    """从 JSON 文件读取配置。"""
    return json.loads(path.read_text(encoding="utf-8"))


def test_save_and_load_config(tmp_path):
    config_path = tmp_path / "config.json"
    expected = {"model": "gpt-test", "timeout": 30}

    save_config(config_path, expected)

    assert config_path.exists()
    assert load_config(config_path) == expected
~~~

tmp_path 会提供独立临时目录，因此不会污染项目目录或误删真实文件。

## 9. Mock 与 monkeypatch：隔离外部依赖

单元测试不应真的调用网络、生产数据库或收费模型。可以用 Mock 替代外部客户端：

~~~python
# src/app/summarizer.py
def summarize(client, text: str) -> str:
    """调用模型客户端，把文本压缩为摘要。"""
    if not text or not text.strip():
        raise ValueError("待摘要文本不能为空")

    response = client.generate(
        prompt=f"请用一句话总结：{text}",
        temperature=0,
    )
    return response.text
~~~

~~~python
from unittest.mock import Mock

import pytest

from app.summarizer import summarize


def test_summarize_sends_expected_prompt_and_returns_model_text():
    client = Mock()
    client.generate.return_value.text = "这是摘要。"

    result = summarize(client, "pytest 可以自动运行测试。")

    assert result == "这是摘要。"
    client.generate.assert_called_once_with(
        prompt="请用一句话总结：pytest 可以自动运行测试。",
        temperature=0,
    )


def test_summarize_rejects_empty_text_without_calling_model():
    client = Mock()

    with pytest.raises(ValueError, match="不能为空"):
        summarize(client, "   ")

    client.generate.assert_not_called()
~~~

Mock 需要验证本模块的职责：

- 是否向下游传递了正确参数；
- 是否正确处理下游返回；
- 无效输入是否提前失败；
- 下游异常是否按约定转换或传播。

monkeypatch 适合隔离环境变量：

~~~python
import os


def get_model_name() -> str:
    """读取模型名；未配置时返回本地默认值。"""
    return os.getenv("MODEL_NAME", "local-dev-model")


def test_get_model_name_uses_environment_value(monkeypatch):
    monkeypatch.setenv("MODEL_NAME", "production-model")

    assert get_model_name() == "production-model"


def test_get_model_name_uses_default_when_missing(monkeypatch):
    monkeypatch.delenv("MODEL_NAME", raising=False)

    assert get_model_name() == "local-dev-model"
~~~

不要直接修改 os.environ 后忘记恢复，否则测试会相互污染。

## 10. 集成测试示例：FastAPI

安装依赖：

~~~powershell
python -m pip install fastapi httpx
~~~

应用代码：

~~~python
# src/app/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field


app = FastAPI()


class SumRequest(BaseModel):
    """加法请求体。"""

    left: int = Field(ge=0, le=1_000_000)
    right: int = Field(ge=0, le=1_000_000)


@app.post("/sum")
def sum_numbers(payload: SumRequest) -> dict[str, int]:
    """返回两个非负整数之和。"""
    total = payload.left + payload.right
    if total > 1_000_000:
        raise HTTPException(status_code=422, detail="结果不能超过 1000000")

    return {"total": total}
~~~

测试代码：

~~~python
# tests/integration/test_api.py
from fastapi.testclient import TestClient

from app.main import app


client = TestClient(app)


def test_sum_returns_total_for_normal_input():
    response = client.post("/sum", json={"left": 2, "right": 3})

    assert response.status_code == 200
    assert response.json() == {"total": 5}


def test_sum_accepts_boundary_input():
    response = client.post("/sum", json={"left": 500_000, "right": 500_000})

    assert response.status_code == 200
    assert response.json() == {"total": 1_000_000}


def test_sum_rejects_empty_body():
    response = client.post("/sum", json={})

    assert response.status_code == 422


def test_sum_rejects_negative_value():
    response = client.post("/sum", json={"left": -1, "right": 3})

    assert response.status_code == 422


def test_sum_rejects_type_error():
    response = client.post("/sum", json={"left": "two", "right": 3})

    assert response.status_code == 422


def test_sum_rejects_result_above_limit():
    response = client.post("/sum", json={"left": 1_000_000, "right": 1})

    assert response.status_code == 422
    assert response.json()["detail"] == "结果不能超过 1000000"
~~~

这组测试覆盖正常值、空值、非法值和边界值。集成测试应同时断言状态码和关键响应内容，避免“状态码正确但错误原因错误”仍被当作成功。

## 11. 集成测试示例：SQLite 数据库

测试数据库必须和真实数据隔离。SQLite 内存数据库很适合学习：

~~~python
# src/app/repository.py
import sqlite3


def create_connection() -> sqlite3.Connection:
    """创建内存 SQLite 连接并初始化用户表。"""
    connection = sqlite3.connect(":memory:")
    connection.execute(
        """
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            email TEXT NOT NULL UNIQUE
        )
        """
    )
    return connection


def create_user(connection: sqlite3.Connection, email: str) -> int:
    """写入用户并返回新用户 ID。"""
    cursor = connection.execute(
        "INSERT INTO users (email) VALUES (?)",
        (email,),
    )
    connection.commit()
    return cursor.lastrowid


def find_user_by_email(connection: sqlite3.Connection, email: str):
    """按邮箱查询单个用户。"""
    return connection.execute(
        "SELECT id, email FROM users WHERE email = ?",
        (email,),
    ).fetchone()
~~~

~~~python
# tests/integration/test_repository.py
import sqlite3

import pytest

from app.repository import create_connection, create_user, find_user_by_email


@pytest.fixture
def connection():
    """为每个测试提供独立内存数据库，并在结束后关闭。"""
    db = create_connection()
    yield db
    db.close()


def test_create_user_can_be_queried(connection):
    user_id = create_user(connection, "alice@example.com")

    row = find_user_by_email(connection, "alice@example.com")

    assert row == (user_id, "alice@example.com")


def test_find_user_returns_none_when_user_does_not_exist(connection):
    assert find_user_by_email(connection, "missing@example.com") is None


def test_create_user_rejects_duplicate_email(connection):
    create_user(connection, "alice@example.com")

    with pytest.raises(sqlite3.IntegrityError):
        create_user(connection, "alice@example.com")
~~~

fixture 中 yield 之前负责准备，之后负责清理。真实项目还要关注迁移、事务回滚、时区、并发和唯一约束。

## 12. LLM 应用如何测试

LLM 输出不完全确定，真实调用也会受模型版本、网络、费用和配额影响。因此应分层测试：

| 要验证的内容 | 推荐方式 |
| --- | --- |
| Prompt 是否包含必要上下文 | 单元测试与 Mock 客户端 |
| JSON 是否可解析、字段是否合规 | 单元测试与固定样本 |
| 工具调用参数、权限、额度 | 单元测试 |
| API 路由、鉴权、错误响应 | 集成测试 |
| LLM 服务真实连通性 | 少量可选集成测试 |
| 回答质量和拒答能力 | 有标准答案的离线评测集 |

先把模型返回内容视为不可信字符串：

~~~python
import json

import pytest


def parse_classification(text: str) -> dict:
    """解析模型分类结果，并限制允许的标签。"""
    payload = json.loads(text)
    label = payload.get("label")
    if label not in {"positive", "negative", "neutral"}:
        raise ValueError("模型返回了不支持的标签")
    return payload


def test_parse_classification_accepts_valid_result():
    result = parse_classification('{"label": "positive", "confidence": 0.92}')

    assert result["label"] == "positive"
    assert result["confidence"] == 0.92


@pytest.mark.parametrize(
    "text",
    [
        "",
        "not-json",
        "{}",
        '{"label": "unknown"}',
        '{"label": null}',
    ],
)
def test_parse_classification_rejects_invalid_result(text):
    with pytest.raises((json.JSONDecodeError, ValueError)):
        parse_classification(text)
~~~

实际项目应使用 Pydantic 或 JSON Schema，并继续覆盖字段缺失、类型错误、额外字段与业务规则冲突。

## 13. 为集成测试加标记

~~~python
import pytest


@pytest.mark.integration
def test_api_works():
    ...
~~~

在 pyproject.toml 注册标记：

~~~toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
markers = [
    "integration: 需要多个组件协作的集成测试",
]
~~~

运行方式：

~~~powershell
pytest -m integration
pytest -m "not integration"
~~~

日常开发可频繁运行快速单元测试；提交前或 CI 中运行完整测试集。

## 14. 常见误区和定位顺序

### 只测成功路径

问题：只有登录成功测试，没有空密码、错误密码、冻结账户。  
改进：每条输入规则都覆盖正常、空、非法和边界值。

### 测试彼此依赖

问题：第二个测试假设第一个测试已写入数据。  
改进：每个测试自己准备数据；使用 function scope fixture；不依赖全局状态或执行顺序。

### 单元测试访问真实服务

问题：网络、密钥、限流和费用让测试不稳定。  
改进：单元测试 Mock 外部边界；少量真实连通性测试单独标记。

### 断言过宽或过脆

问题：断言 status_code 小于 500 太宽；精确比较随机 ID 又太脆。  
改进：断言业务契约，例如正确状态码、错误码、关键字段和值域。

失败后的正确顺序：

1. 确认测试输入和预期是否正确；
2. 查看实际值、异常类型或响应体；
3. 判断是实现缺陷、测试缺陷还是需求未定义；
4. 只修复已定位的最小问题；
5. 先重跑失败测试，再跑相关测试集。

## 15. 7 天练习计划

### 第 1 天：断言

为“两数相加”“判断字符串为空”“摄氏转华氏”写正常与边界测试。

### 第 2 天：异常和参数化

实现 validate_age(age)：只接受 0 到 150 的整数，且 bool 不算整数。用参数化覆盖 None、字符串、负数、0、150、151 和 True。

### 第 3 天：fixture 和文件

实现 JSON 配置读写模块。用 tmp_path 覆盖正常读写、文件不存在、非法 JSON 和空文件。

### 第 4 天：Mock

实现天气查询服务，Mock 成功响应、超时异常和错误状态码；断言请求参数和错误处理。

### 第 5 天：FastAPI

实现 POST /tasks。覆盖合法请求、缺失字段、错误类型、长度边界和重复任务名。

### 第 6 天：数据库

使用 SQLite 实现任务创建与查询。确保每个测试独立，覆盖不存在记录、重复唯一键和事务失败。

### 第 7 天：日常工作流

每次修改先运行相关测试，再运行 pytest -q。挑一个真实缺陷，写出复现测试，修复后确认测试能够防止回归。

## 16. 提交前自查清单

- [ ] 新增或修改的规则是否有测试？
- [ ] 是否覆盖正常值、空值、非法值和边界值？
- [ ] 测试是否能独立、重复运行？
- [ ] 单元测试是否隔离网络、生产数据库和真实密钥？
- [ ] 异常是否断言了合理类型或 HTTP 状态码？
- [ ] 是否断言了真正重要的结果，而非仅仅“不报错”？
- [ ] 是否先运行失败测试，再运行相关测试集？
- [ ] 未覆盖或无法验证的风险是否被如实记录？

## 17. 核心结论

1. 测试把需求转成可执行断言，不是为了增加代码量。
2. 单元测试验证独立规则；集成测试验证组件协作。
3. 正常、空、非法、边界四类用例应成为固定检查框架。
4. fixture 提供隔离，Mock 与 monkeypatch 隔离外部依赖。
5. LLM 系统要重点测试输入校验、结构化输出、工具参数、权限和 API 契约。
6. 测试失败先定位证据，再做最小修正，最后重新验证。
