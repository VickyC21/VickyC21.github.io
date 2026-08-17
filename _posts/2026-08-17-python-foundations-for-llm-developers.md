---
layout: post
title: "大模型开发工程师 Python 基础讲义：从数据类型到类型标注"
date: 2026-08-17 01:00:00 +0800
categories: [Python基础]
tags: [Python, LLM, 入门, 类型标注, 异常处理]
---

# 大模型开发工程师 Python 基础讲义：从数据类型到类型标注

本文面向 Python 零基础学习者，覆盖大模型开发工程师必备的六项能力：数据类型、函数、类、异常处理、上下文管理器和类型标注。目标不是背语法，而是能把一个模型请求拆成清晰的数据、步骤和失败路径。

建议每个示例都亲自运行，改一个值并观察输出和报错。Python 能力来自输入、输出和反馈，而不是只看懂代码。

> 本文以 Python 3.11+ 为例，代码只使用标准库。将示例保存为 `.py` 文件后，用 `python 文件名.py` 运行。

## 1. 学习目标和路线

完成后，你应该能够：

1. 用合适的数据类型表示用户问题、消息列表、模型配置和响应；
2. 把校验、提示词构造、成本计算拆为可复用函数；
3. 用类组织“数据 + 行为”，例如聊天消息与生成配置；
4. 处理用户输入、文件读取和上游响应的失败；
5. 用 `with` 可靠释放文件、连接和锁；
6. 用类型标注把函数的输入、输出和数据结构说清楚。

| 阶段 | 内容 | 完成标志 |
| --- | --- | --- |
| 1 | 数据类型 | 能读写嵌套 `dict` 与 `list` |
| 2 | 函数 | 能将校验和业务逻辑拆开 |
| 3 | 类 | 能定义有属性和方法的对象 |
| 4 | 异常 | 能区分可恢复错误和程序缺陷 |
| 5 | 上下文管理器 | 能安全读写 UTF-8 文件 |
| 6 | 类型标注 | 能标注函数、容器和可选值 |
| 7 | 综合练习 | 能完成模拟的 LLM 请求处理器 |

## 2. 变量和基础类型

变量是给一个值取名字。等号 `=` 的含义是“把右侧值绑定给左侧名称”，不是数学中的相等。

```python
model_name = "gpt-4.1-mini"
max_tokens = 512
temperature = 0.2
stream = False

print(model_name)
print(max_tokens * 2)
```

使用小写字母和下划线命名，例如 `request_id`、`model_response`。避免 `a`、`data1` 这类不表达用途的名称。

用 `type()` 可查看类型：

```python
print(type("hello"))  # str
print(type(42))       # int
print(type(3.14))     # float
print(type(True))     # bool
print(type(None))     # NoneType
```

外部输入经常是字符串。遇到报错时，先打印值和类型：

```python
token_count = "128"
print(token_count, type(token_count))

token_count = int(token_count)
print(token_count + 1)
```

## 3. 数据类型：让数据形状清晰可靠

### 3.1 字符串 `str`

字符串用于用户问题、提示词、模型回答和日志。

```python
prompt = "请用三句话解释 RAG"
system_prompt = "你是一名严谨的技术助手。"

full_prompt = system_prompt + "\n用户问题：" + prompt
print(full_prompt)
```

拼接变量优先使用 f-string：

```python
model = "gpt-4.1-mini"
latency_ms = 238
print(f"模型 {model} 的响应耗时为 {latency_ms} ms。")
```

常用方法：

```python
question = "  什么是上下文窗口？  "

print(question.strip())              # 去除两端空白
print(question.lower())              # 转小写
print(question.replace("窗口", "长度"))
print("、".join(["RAG", "Agent", "评测"]))
print(question.startswith("什么"))
```

字符串不可变。调用 `replace()` 不会原地修改：

```python
text = "hello"
text.replace("h", "H")
print(text)  # hello

text = text.replace("h", "H")
print(text)  # Hello
```

### 3.2 数字 `int`、`float` 和布尔值 `bool`

整数适合 token 数、状态码和重试次数；浮点数适合温度、耗时和价格；布尔值表达真或假。

```python
input_tokens = 120
output_tokens = 80
price_per_million = 2.0

total_tokens = input_tokens + output_tokens
cost = total_tokens / 1_000_000 * price_per_million
is_short_request = total_tokens < 512

print(total_tokens, cost, is_short_request)
```

除法差异：

```python
print(7 / 2)   # 3.5，普通除法
print(7 // 2)  # 3，向下取整
print(7 % 2)   # 1，余数
```

条件分支使用 `if / elif / else`，缩进是 Python 语法的一部分：

```python
temperature = 1.2

if temperature < 0:
    print("温度不能为负数")
elif temperature > 1:
    print("温度较高，回答可能更发散")
else:
    print("温度处于常用范围")
```

不要混淆赋值与比较：

```python
status = "success"
is_success = status == "success"
print(is_success)
```

### 3.3 列表 `list`

列表保存有顺序的一组值，适合聊天记录、检索文档和候选模型。

```python
messages = [
    {"role": "system", "content": "你是技术助手。"},
    {"role": "user", "content": "什么是向量检索？"},
]

print(messages[0])
print(messages[-1])
print(len(messages))
```

下标从 `0` 开始，`-1` 表示最后一个元素。

```python
models = ["gpt-4.1-mini", "qwen", "llama"]
models.append("deepseek")
models.remove("qwen")
first_model = models.pop(0)

print(first_model)
print(models)
```

遍历元素：

```python
documents = ["文档 A", "文档 B", "文档 C"]

for document in documents:
    print(f"正在处理：{document}")

for index, document in enumerate(documents, start=1):
    print(f"{index}. {document}")
```

不要在遍历同一列表时删除元素。筛选时创建新列表：

```python
scores = [0.91, 0.15, 0.78, 0.22]
qualified_scores = []

for score in scores:
    if score >= 0.5:
        qualified_scores.append(score)

print(qualified_scores)

# 熟悉后可写成列表推导式：
qualified_scores = [score for score in scores if score >= 0.5]
```

### 3.4 字典 `dict`

字典通过“键: 值”保存关联信息，JSON 请求和响应通常对应 Python 字典。

```python
request = {
    "request_id": "req_001",
    "model": "gpt-4.1-mini",
    "temperature": 0.2,
    "stream": False,
}

print(request["model"])
request["max_tokens"] = 512
request["temperature"] = 0.5
```

键一定存在时使用 `[]`；外部响应可能缺字段时使用 `get()`：

```python
response = {"answer": "RAG 是检索增强生成。"}

print(response.get("answer"))
print(response.get("citations"))       # None
print(response.get("citations", []))   # []
```

遍历字典：

```python
usage = {"input_tokens": 120, "output_tokens": 80}

for name, count in usage.items():
    print(f"{name}: {count}")
```

嵌套响应示例：

```python
api_response = {
    "choices": [
        {
            "message": {
                "role": "assistant",
                "content": "可以使用混合检索提升召回率。",
            }
        }
    ],
    "usage": {"input_tokens": 100, "output_tokens": 30},
}

answer = api_response["choices"][0]["message"]["content"]
print(answer)
```

深层 `[]` 读取只适合已经验证过的数据。上游可能缺键、空列表或类型错误，生产代码必须先校验或处理异常。

### 3.5 元组 `tuple`、集合 `set` 和 `None`

元组是固定顺序的一组值，通常不修改：

```python
DEFAULT_TIMEOUTS = (3, 10)
connect_timeout, read_timeout = DEFAULT_TIMEOUTS
```

集合自动去重，适合角色白名单和文档 ID 去重：

```python
allowed_roles = {"system", "user", "assistant"}
print("user" in allowed_roles)

document_ids = {"doc_1", "doc_2", "doc_1"}
print(document_ids)
```

集合没有稳定顺序，不能按下标访问。

`None` 表示“没有值”，不等于空字符串、零或空列表：

```python
cached_answer = None

if cached_answer is None:
    print("缓存未命中")
```

判断 `None` 使用 `is None`，不要写 `== None`。

### 3.6 可变与不可变

字符串、整数、浮点数和元组通常不可变；列表和字典可变。函数内部修改可变参数会影响调用方：

```python
def add_system_message(messages):
    messages.append({"role": "system", "content": "保持严谨。"})


history = []
add_system_message(history)
print(history)
```

如果不希望修改原数据，复制后处理：

```python
def with_system_message(messages):
    copied_messages = messages.copy()
    copied_messages.append({"role": "system", "content": "保持严谨。"})
    return copied_messages
```

## 4. 函数：把一件事做清楚、做一次、重复使用

函数将单一任务封装成可复用单元，能减少重复、缩小测试范围，并让主流程更容易读。

```python
def build_user_message(question):
    """将用户问题包装成消息字典。"""
    return {"role": "user", "content": question}


message = build_user_message("什么是 KV Cache？")
print(message)
```

`def` 后是函数名，括号中是参数，`return` 把结果交回调用方。没有 `return` 的函数默认返回 `None`。

### 4.1 参数、关键字参数和默认值

```python
def create_request(model, prompt):
    return {"model": model, "prompt": prompt}


create_request("gpt-4.1-mini", "解释注意力机制")

create_request(
    prompt="解释注意力机制",
    model="gpt-4.1-mini",
)
```

关键字参数不依赖顺序，多个同类参数时可读性更强。

```python
def create_generation_config(temperature=0.2, max_tokens=512):
    return {
        "temperature": temperature,
        "max_tokens": max_tokens,
    }


print(create_generation_config())
print(create_generation_config(temperature=0.8))
```

不要将可变对象作为默认值。下面的 `tags` 会被所有调用共享：

```python
# 不推荐
def add_tag_bad(tag, tags=[]):
    tags.append(tag)
    return tags
```

正确写法以 `None` 为哨兵：

```python
def add_tag(tag, tags=None):
    if tags is None:
        tags = []
    tags.append(tag)
    return tags
```

### 4.2 输入校验和单一职责

函数应尽早校验不可信输入：

```python
def normalize_question(question):
    """清理并校验用户问题。"""
    if not isinstance(question, str):
        raise TypeError("question 必须是字符串")

    normalized = question.strip()
    if not normalized:
        raise ValueError("question 不能为空")

    if len(normalized) > 4_000:
        raise ValueError("question 过长，请控制在 4000 个字符以内")

    return normalized
```

不要让一个函数同时校验参数、调用模型、打印和写文件。拆成 `normalize_question()`、`build_messages()`、`generate()`、`save_answer()`，失败点更明确，也更容易测试。

函数内部变量通常只在函数内部存在。避免依赖随处可改的全局变量，把依赖作为参数传入。

相同输入总返回相同输出，且不读写外部状态的函数称为纯函数：

```python
def estimate_cost(total_tokens, price_per_million):
    return total_tokens / 1_000_000 * price_per_million
```

打印、写文件和 HTTP 请求都会产生副作用。副作用不是坏事，但应尽量位于程序边界，核心计算保持简单。

### 4.3 `*args`、`**kwargs` 与测试

```python
def log_event(event_name, **fields):
    print(event_name)
    for key, value in fields.items():
        print(f"  {key}={value}")


log_event("model_request", model="gpt-4.1-mini", request_id="req_001")
```

它们适合转发参数或记录灵活字段，不应成为逃避清晰业务接口的工具。

尚未学习 pytest 时，可以先用断言验证函数：

```python
assert normalize_question("  什么是 RAG？ ") == "什么是 RAG？"

try:
    normalize_question("")
except ValueError:
    print("空字符串被正确拒绝")
else:
    raise AssertionError("预期应该抛出 ValueError")
```

对函数至少覆盖：正常输入、空值、类型错误和边界值。

## 5. 类：组织“数据 + 行为”

当数据和围绕它的操作天然属于同一个对象时，使用类。例如一条聊天消息有角色、内容，以及转换为 API 字典的行为。

```python
class ChatMessage:
    """表示一条聊天消息。"""

    def __init__(self, role, content):
        self.role = role
        self.content = content

    def to_dict(self):
        return {
            "role": self.role,
            "content": self.content,
        }


message = ChatMessage("user", "什么是函数调用？")
print(message.role)
print(message.to_dict())
```

- `class ChatMessage:` 定义类型；
- `__init__` 在创建实例时自动调用；
- `self` 是当前对象；
- `self.role` 是实例属性；
- `to_dict()` 是实例方法。

### 5.1 把约束放进对象

让错误数据无法创建，比让它在程序中流动更可靠：

```python
class ChatMessage:
    """表示经过基础校验的聊天消息。"""

    ALLOWED_ROLES = {"system", "user", "assistant"}

    def __init__(self, role, content):
        if role not in self.ALLOWED_ROLES:
            raise ValueError(f"不支持的角色：{role}")

        if not isinstance(content, str) or not content.strip():
            raise ValueError("content 必须是非空字符串")

        self.role = role
        self.content = content.strip()

    def to_dict(self):
        return {"role": self.role, "content": self.content}

    def __repr__(self):
        return f"ChatMessage(role={self.role!r}, content={self.content!r})"
```

`ALLOWED_ROLES` 是所有实例共享的类属性；`self.role` 属于各自实例。 `__repr__` 让对象更方便调试。

### 5.2 类方法、继承和组合

类方法可提供另一种构造方式：

```python
class ChatMessage:
    def __init__(self, role, content):
        self.role = role
        self.content = content

    @classmethod
    def from_dict(cls, data):
        return cls(role=data["role"], content=data["content"])
```

继承描述“是一种”关系：

```python
class BaseModelClient:
    def generate(self, prompt):
        raise NotImplementedError


class DemoModelClient(BaseModelClient):
    def generate(self, prompt):
        return f"演示回答：{prompt}"
```

不要为了“面向对象”把所有函数塞进类。没有长期状态、只做一次转换的逻辑通常用函数更清楚。多个能力协作时，优先考虑组合，避免不必要的深层继承。

## 6. 异常处理：让失败可预期、可定位、可恢复

外部输入、文件、网络和模型输出都可能失败。异常处理不是隐藏错误，而是在知道如何恢复或补充上下文的位置做出正确决策。

| 异常 | 常见原因 |
| --- | --- |
| `ValueError` | 值不合法，例如 `int("abc")` |
| `TypeError` | 对不兼容类型做操作 |
| `KeyError` | 字典键不存在 |
| `IndexError` | 列表下标越界 |
| `FileNotFoundError` | 文件不存在 |
| `TimeoutError` | 操作超时 |

### 6.1 `try / except / else / finally`

```python
def parse_max_tokens(value):
    try:
        max_tokens = int(value)
    except ValueError:
        raise ValueError("max_tokens 必须是整数") from None
    else:
        if max_tokens <= 0:
            raise ValueError("max_tokens 必须大于 0")
        return max_tokens
```

- `try`：放置可能失败的最小代码块；
- `except`：处理预期且有恢复方案的异常；
- `else`：没有异常时执行；
- `finally`：无论成功失败都会执行，常用于资源清理。

不要写裸 `except`，它会吞掉拼写错误、逻辑错误等真正需要修复的问题：

```python
# 不推荐
try:
    result = call_model()
except:
    result = "发生错误"
```

捕获具体类型：

```python
try:
    result = parse_max_tokens("abc")
except ValueError as error:
    print(f"参数错误：{error}")
```

### 6.2 自定义异常和异常链

当调用方需要区分业务失败时，定义自定义异常：

```python
class ModelResponseError(Exception):
    """表示模型响应不符合预期结构。"""


def extract_answer(response):
    answer = response.get("answer")
    if not isinstance(answer, str) or not answer.strip():
        raise ModelResponseError("模型响应缺少非空 answer 字段")
    return answer.strip()
```

转译底层异常时保留真正原因：

```python
import json


def load_json_config(text):
    try:
        return json.loads(text)
    except json.JSONDecodeError as error:
        raise ValueError("配置不是合法 JSON") from error
```

处理用户可修正参数、文件不存在、临时网络失败等可预期错误；变量拼错、逻辑矛盾等程序缺陷应记录并尽早暴露，不能用默认值伪装成功。

## 7. 上下文管理器：确保资源始终被释放

文件、连接、事务和锁都需要释放。若中途异常而不清理，长时间运行的服务可能泄漏文件句柄、连接或锁。

`with` 把资源获取和清理绑定在一起：

```python
with open("prompt.txt", encoding="utf-8") as file:
    prompt = file.read()

print(prompt)
```

离开 `with` 块时，无论是否异常，文件都会关闭。写文件时显式声明 UTF-8：

```python
answer = "RAG 通过检索外部知识增强模型回答。"

with open("answer.txt", "w", encoding="utf-8") as file:
    file.write(answer)
```

### 7.1 自定义上下文管理器

上下文管理器实现 `__enter__()` 和 `__exit__()`：

```python
from time import perf_counter


class Timer:
    """统计 with 代码块执行时间。"""

    def __enter__(self):
        self.start_time = perf_counter()
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        elapsed = perf_counter() - self.start_time
        print(f"代码块耗时：{elapsed:.3f} 秒")
        return False


with Timer():
    total = sum(range(1_000_000))
```

`__exit__()` 返回 `False` 表示不吞掉异常，通常应保持这个行为。

简单场景可使用 `contextlib.contextmanager`：

```python
from contextlib import contextmanager
from time import perf_counter


@contextmanager
def timer(label):
    start_time = perf_counter()
    try:
        yield
    finally:
        elapsed = perf_counter() - start_time
        print(f"{label} 耗时：{elapsed:.3f} 秒")


with timer("生成提示词"):
    prompt = "请总结：" + "Python 基础。"
```

在 LLM 工程中，上下文管理器常用于提示词和评测文件、数据库事务、HTTP 会话、锁、计时及需要释放的计算资源。

## 8. 类型标注：在运行前说清接口

Python 是动态类型语言。类型标注默认不在运行时强制检查，但能帮助 IDE、静态检查工具和代码阅读者理解约定。

```python
model_name: str = "gpt-4.1-mini"
max_tokens: int = 512
temperature: float = 0.2
stream: bool = False


def estimate_cost(total_tokens: int, price_per_million: float) -> float:
    return total_tokens / 1_000_000 * price_per_million
```

### 8.1 容器与可选值

```python
models: list[str] = ["gpt-4.1-mini", "qwen"]
usage: dict[str, int] = {"input_tokens": 120, "output_tokens": 80}
roles: set[str] = {"system", "user", "assistant"}
timeouts: tuple[float, float] = (3.0, 30.0)


def find_cached_answer(cache: dict[str, str], key: str) -> str | None:
    return cache.get(key)
```

返回值含 `None` 时，调用方必须先判断：

```python
answer = find_cached_answer({}, "question:rag")

if answer is not None:
    print(answer.upper())
else:
    print("缓存未命中")
```

### 8.2 `TypedDict`、数据类和 `Protocol`

复杂 JSON 可用 `TypedDict` 描述：

```python
from typing import NotRequired, TypedDict


class MessageDict(TypedDict):
    role: str
    content: str


class GenerationConfigDict(TypedDict):
    model: str
    temperature: float
    max_tokens: NotRequired[int]
```

主要保存数据的对象可使用数据类：

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class GenerationRequest:
    model: str
    prompt: str
    temperature: float = 0.2
    max_tokens: int = 512
```

`frozen=True` 让对象创建后不可修改，适合不希望请求配置被中途改动的场景。

只关心对象“能生成文本”，不关心其父类时可使用 `Protocol`：

```python
from typing import Protocol


class TextGenerator(Protocol):
    def generate(self, prompt: str) -> str:
        """根据提示词生成文本。"""


def ask_model(client: TextGenerator, question: str) -> str:
    return client.generate(question)
```

任何拥有匹配 `generate()` 方法的对象都可以传入，便于用假客户端测试。

类型标注实用原则：

1. 先标注公开函数、跨模块函数和复杂数据结构；
2. 不要为了标注写难读的“类型体操”；
3. `Any` 表示放弃检查，应限制范围；
4. 外部 JSON、文件和模型输出仍需运行时校验；
5. 类型标注和运行时校验互补，不能互相替代。

## 9. 综合示例：模拟 LLM 请求处理器

下面组合六项能力。它不调用真实模型，重点是数据流和失败路径。

```python
from dataclasses import dataclass
from time import perf_counter
from typing import TypedDict


class MessageDict(TypedDict):
    role: str
    content: str


class RequestValidationError(ValueError):
    """表示生成请求不满足业务约束。"""


@dataclass(frozen=True)
class GenerationConfig:
    model: str
    temperature: float = 0.2
    max_tokens: int = 512


class Timer:
    def __enter__(self):
        self.start_time = perf_counter()
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        elapsed = perf_counter() - self.start_time
        print(f"本次处理耗时：{elapsed:.3f} 秒")
        return False


def validate_question(question: str) -> str:
    normalized = question.strip()
    if not normalized:
        raise RequestValidationError("问题不能为空")
    if len(normalized) > 4_000:
        raise RequestValidationError("问题超过 4000 个字符")
    return normalized


def validate_config(config: GenerationConfig) -> GenerationConfig:
    if not 0 <= config.temperature <= 2:
        raise RequestValidationError("temperature 必须在 0 到 2 之间")
    if config.max_tokens <= 0:
        raise RequestValidationError("max_tokens 必须大于 0")
    return config


def build_messages(question: str) -> list[MessageDict]:
    return [
        {"role": "system", "content": "你是一名严谨的 Python 学习助手。"},
        {"role": "user", "content": question},
    ]


def fake_generate(messages: list[MessageDict], config: GenerationConfig) -> str:
    return f"[{config.model}] 演示回答：{messages[-1]['content']}"


def handle_generation(question: str, config: GenerationConfig) -> str:
    clean_question = validate_question(question)
    checked_config = validate_config(config)
    messages = build_messages(clean_question)

    with Timer():
        return fake_generate(messages, checked_config)


def main() -> None:
    config = GenerationConfig(model="demo-model", max_tokens=256)

    try:
        answer = handle_generation("  什么是类型标注？  ", config)
    except RequestValidationError as error:
        print(f"请求被拒绝：{error}")
    else:
        print(answer)


if __name__ == "__main__":
    main()
```

观察这个示例：

- `str`、`list`、`dict` 描述数据形状；
- 数据类保存不可变配置；
- 函数分别承担校验、构造和生成；
- 自定义异常表达用户可理解的请求错误；
- `with Timer()` 保证异常时也记录耗时；
- 类型标注使输入输出和嵌套结构可见。

尝试将温度改成 `3.0`、将问题改为空字符串，观察错误在何处被拒绝；再尝试让 `fake_generate()` 返回空字符串，思考应在哪一层新增响应校验。

## 10. 初学者高频错误

### 把字符串当数字

```python
max_tokens = "256"
# print(max_tokens + 1)  # TypeError

max_tokens = int(max_tokens)
print(max_tokens + 1)
```

### 用 `dict["key"]` 读取不可信响应

```python
response = {}
# answer = response["answer"]  # KeyError

answer = response.get("answer")
if answer is None:
    print("响应缺少 answer")
```

### 捕获所有异常后继续运行

```python
# 不推荐：可能掩盖程序错误。
try:
    answer = 1 / 0
except Exception:
    answer = "默认回答"
```

只处理有恢复方案的异常。未知异常应记录并暴露，便于定位根因。

### 忘记关闭文件或连接

优先使用 `with`。不要依赖程序结束时“应该会自动关闭”，Web 服务可能运行数周。

### 类型标注代替运行时校验

类型标注只是约定。HTTP JSON、文件内容和模型输出依然是不可信输入，仍需要 `isinstance()`、范围校验、Schema 或 Pydantic。

### 使用 `is` 比较普通值

``is`` 判断是否为同一个对象，``==`` 比较值。普通值使用 `==`；只有判断 `None` 时使用 `is None`。

## 11. 建议练习

1. **消息统计器**：输入消息列表，输出总消息数、每种 `role` 的数量、内容总字符数和空内容消息序号。
2. **模型参数校验器**：拒绝空模型名、温度不在 `0` 到 `2`、最大 token 不是正整数。
3. **提示词模板加载器**：读取 UTF-8 模板，用用户问题替换 `{question}`；处理文件不存在和缺少占位符。
4. **聊天消息类**：只允许三种角色、拒绝空内容、实现 `to_dict()` 和 `__repr__()`。
5. **带类型标注的假模型客户端**：实现 `DemoClient.generate(prompt: str) -> str`，再用 `Protocol` 编写调用函数。

## 12. 学习完成自检

在学习 asyncio、HTTP 服务、pytest、数据库或 LLM SDK 前，请确认你能够：

- [ ] 使用 `str`、`int`、`float`、`bool`、`list`、`dict`、`set`、`tuple` 和 `None`；
- [ ] 从嵌套字典和列表安全读取数据；
- [ ] 写出带输入校验、默认参数和返回值的函数；
- [ ] 解释可变默认参数为何危险；
- [ ] 定义包含属性、方法和基础校验的类；
- [ ] 捕获具体异常而不是裸 `except`；
- [ ] 使用 `with` 安全读写 UTF-8 文件；
- [ ] 为函数、容器和可选值写基本类型标注；
- [ ] 解释为何类型标注不能替代运行时校验。

## 13. 下一步

建议依次学习：

1. `pytest`：自动覆盖正常、空值、非法值和边界场景；
2. `logging`：记录请求 ID、耗时、错误与模型用量；
3. `json`、`pathlib`、`datetime`：掌握常用标准库；
4. `asyncio`：理解协程、超时、取消和并发请求；
5. HTTP 客户端与 FastAPI：将函数组织成真实 LLM API；
6. Pydantic：为 API 请求和模型输出建立运行时 Schema。

Python 基础的终点不是记住所有语法，而是能把需求拆成清晰的数据结构、函数边界、失败路径和资源生命周期。后续做 RAG、Agent、流式接口和模型评测时，这些能力会不断复用。

