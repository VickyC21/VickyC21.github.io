---
layout: post
title: "Python Web 服务入门：用 FastAPI 和 Flask 设计 REST API、流式响应与 WebSocket"
date: 2026-08-17 14:00:00 +0800
categories: [Python基础]
tags: [Python, FastAPI, Flask, REST API, SSE, WebSocket, 后端开发]
---

# Python Web 服务入门：用 FastAPI 和 Flask 设计 REST API、流式响应与 WebSocket

大模型开发工程师不能只会调用模型，还要把模型能力做成浏览器、前端和其他服务都能稳定使用的 Web 服务。本文围绕技能清单中的 FastAPI 或 Flask、REST API、流式响应和 WebSocket，给出一条从零开始的学习路线与可运行示例。

示例使用内存字典保存任务。这样可以先专注 HTTP、协议和并发行为；但数据会在重启后丢失，多进程也不会共享，因此不能直接作为生产方案。

## 1. 学完后应能做到什么

- 解释 HTTP 的方法、路径、查询参数、请求头、请求体、状态码和响应体。
- 用 FastAPI 定义带输入校验、错误响应和返回模型的 REST API。
- 分辨普通 HTTP、流式 HTTP、SSE 和 WebSocket 的边界。
- 实现可结束、可报告错误、能响应客户端取消的流式接口。
- 实现 WebSocket 收发、断连和单进程广播。
- 知道 `async def` 何时有效，何时仍会阻塞服务。
- 为成功、非法输入和资源不存在写最小自动化测试。

本文不试图一次讲完数据库、OAuth、Docker 和消息队列。应先完成这个最小闭环，再逐项增加外部组件。

## 2. 四种通信方式

普通 HTTP 是一问一答：客户端发请求，服务器处理，最后返回完整响应。查询任务、创建会话、保存设置等通常适合它。

| 方式 | 数据方向 | 连接特点 | 常见用途 |
| --- | --- | --- | --- |
| 普通 HTTP | 客户端请求，服务端响应 | 一问一答 | CRUD、查询、配置 |
| 流式 HTTP | 客户端请求，服务端分段响应 | 仍是 HTTP，响应体分多次到达 | LLM 输出、大文件导出 |
| SSE | 服务端持续推送 | HTTP 长连接，服务端单向 | token、进度、通知 |
| WebSocket | 双方均可主动发送 | HTTP 握手后升级为双向连接 | 聊天、协作、实时控制 |

不要因为 WebSocket 听起来更实时就让所有接口都用它。REST 的认证、缓存、监控和失败语义更直接。只有客户端和服务端都需要在连接存在期间随时发送消息时，WebSocket 的复杂度才值得支付。

以这个请求为例：

```text
GET /api/tasks/42?include_notes=true
Authorization: Bearer <token>
```

- `42` 是路径参数，通常定位一个资源。
- `include_notes=true` 是查询参数，常用于筛选、排序、分页或可选行为。
- `Authorization` 是请求头，承载认证等元信息。
- `POST`、`PUT`、`PATCH` 常把 JSON 放在请求体中。

密钥和令牌不要放进 URL 查询参数，URL 很可能进入日志、代理记录和浏览器历史。

## 3. FastAPI 与 Flask 如何选择

两者都能写出可靠服务。新建大模型应用 API 时，建议优先 FastAPI；维护既有 Flask 项目时，应理解它的同步模型，而不是为了跟风整体重写。

| 维度 | FastAPI | Flask |
| --- | --- | --- |
| 风格 | 类型标注和 Pydantic 驱动 | 极简、自由度高 |
| 参数校验 | 自动生成校验与错误响应 | 默认需要自行处理或加扩展 |
| API 文档 | 自动生成 OpenAPI 和 `/docs` | 通常需要额外工具 |
| 异步 | 原生支持 ASGI 和 `async def` | 以同步 WSGI 为核心 |
| WebSocket | 原生支持 | 常需要扩展或 ASGI 方案 |
| 适合 | 新 API、流式 LLM 服务 | 既有项目、小型同步服务 |

框架不会替你解决认证、授权、超时、重试、限流和测试。框架的价值是使接口契约更清晰、更容易验证。

## 4. 运行第一个 FastAPI 服务

建议使用 Python 3.11+：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install "fastapi[standard]" pytest httpx
```

创建 `main.py`：

```python
# 演示 REST API、流式响应和 WebSocket 的最小 FastAPI 服务。
from fastapi import FastAPI

app = FastAPI(title="任务服务练习")


@app.get("/health")
async def health() -> dict[str, str]:
    # 返回服务健康状态，供本地检查和部署平台探测。
    return {"status": "ok"}
```

启动开发服务器：

```powershell
uvicorn main:app --reload
```

访问 `http://127.0.0.1:8000/health` 应返回 JSON；`/docs` 是可交互的 API 文档；`/openapi.json` 是机器可读的接口定义。`main:app` 表示从 `main.py` 导入名为 `app` 的对象。`--reload` 只用于本地开发。

## 5. REST API：用资源和状态码表达意图

REST 将任务、会话、文档、模型配置等业务对象视为资源。HTTP 方法表达动作，路径表达资源：

| 目的 | 方法 | 路径 | 成功状态码 |
| --- | --- | --- | --- |
| 查询任务列表 | `GET` | `/api/tasks` | `200` |
| 查询一个任务 | `GET` | `/api/tasks/{task_id}` | `200` |
| 创建任务 | `POST` | `/api/tasks` | `201` |
| 部分更新任务 | `PATCH` | `/api/tasks/{task_id}` | `200` |
| 删除任务 | `DELETE` | `/api/tasks/{task_id}` | `204` |

尽量避免 `/getTasks`、`/createTask`、`/deleteTask?id=1`。这些写法重复了 HTTP 方法的语义，长期协作会让接口风格失控。

常用状态码：

- `200 OK`：成功并返回数据。
- `201 Created`：成功创建资源。
- `204 No Content`：成功但没有响应体，常用于删除。
- `401 Unauthorized`：身份凭证缺失或无效。
- `403 Forbidden`：身份已确认，但没有权限。
- `404 Not Found`：资源不存在。
- `409 Conflict`：与当前状态冲突，例如唯一键重复。
- `422 Unprocessable Content`：字段校验失败，FastAPI 默认使用。
- `429 Too Many Requests`：超过限流配额。
- `500 Internal Server Error`：未预期服务端错误，不能暴露堆栈。

## 6. 用 Pydantic 定义接口契约

把以下内容加入 `main.py`。它实现内存任务的创建、查询、更新和删除：

```python
from datetime import datetime, timezone
from typing import Annotated
from uuid import uuid4

from fastapi import HTTPException, Query, status
from pydantic import BaseModel, Field


class TaskCreate(BaseModel):
    # 创建任务时客户端允许提交的字段。
    title: str = Field(min_length=1, max_length=100, description="任务标题")
    description: str | None = Field(default=None, max_length=1000)


class TaskUpdate(BaseModel):
    # 部分更新任务时允许提交的字段。
    title: str | None = Field(default=None, min_length=1, max_length=100)
    description: str | None = Field(default=None, max_length=1000)
    completed: bool | None = None


class Task(BaseModel):
    # 服务对外返回的完整任务资源。
    id: str
    title: str
    description: str | None
    completed: bool
    created_at: datetime


TASKS: dict[str, Task] = {}


@app.post("/api/tasks", response_model=Task, status_code=status.HTTP_201_CREATED)
async def create_task(payload: TaskCreate) -> Task:
    # 创建任务并补全由服务端负责的字段。
    task = Task(
        id=str(uuid4()),
        title=payload.title,
        description=payload.description,
        completed=False,
        created_at=datetime.now(timezone.utc),
    )
    TASKS[task.id] = task
    return task


@app.get("/api/tasks", response_model=list[Task])
async def list_tasks(
    completed: bool | None = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
) -> list[Task]:
    # 按完成状态筛选任务，并限制返回数量。
    tasks = list(TASKS.values())
    if completed is not None:
        tasks = [task for task in tasks if task.completed == completed]
    return tasks[:limit]


@app.get("/api/tasks/{task_id}", response_model=Task)
async def get_task(task_id: str) -> Task:
    # 按 ID 返回任务；不存在时返回 404。
    task = TASKS.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="任务不存在")
    return task


@app.patch("/api/tasks/{task_id}", response_model=Task)
async def update_task(task_id: str, payload: TaskUpdate) -> Task:
    # 仅更新本次请求中显式给出的字段。
    task = TASKS.get(task_id)
    if task is None:
        raise HTTPException(status_code=404, detail="任务不存在")
    updated_task = task.model_copy(update=payload.model_dump(exclude_unset=True))
    TASKS[task_id] = updated_task
    return updated_task


@app.delete("/api/tasks/{task_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_task(task_id: str) -> None:
    # 删除任务；重复删除返回 404，避免静默掩盖调用错误。
    if task_id not in TASKS:
        raise HTTPException(status_code=404, detail="任务不存在")
    del TASKS[task_id]
```

关键点：

- `TaskCreate` 与 `Task` 分开，客户端不应伪造 `id`、创建时间和初始状态。
- `TaskUpdate` 的字段可选；`exclude_unset=True` 才能区分未传字段和主动传入空值。
- `response_model` 不只是文档，也会校验和过滤响应，避免内部字段泄露。
- `TASKS` 仅用于练习，不支持持久化、事务和跨进程共享。

验证：

```powershell
curl.exe -X POST http://127.0.0.1:8000/api/tasks `
  -H "Content-Type: application/json" `
  -d "{\"title\":\"学习 FastAPI\",\"description\":\"完成 REST 练习\"}"

curl.exe "http://127.0.0.1:8000/api/tasks?completed=false&limit=10"
curl.exe "http://127.0.0.1:8000/api/tasks/not-exists"
```

应验证：创建返回 `201`；`limit=0` 返回 `422`；不存在 ID 返回 `404`；删除返回 `204` 且没有响应体。客户端收到 `204` 后不能调用 `response.json()`。

## 7. 认证、CORS 与错误处理

结构校验回答字段格式是否正确；业务校验回答当前用户能否操作资源、状态能否改变。两者不能混为一谈。

FastAPI 的 `Depends` 可复用认证等前置逻辑。下面只演示入口，生产环境还应验证签名、过期时间、账号状态、角色和对象级权限：

```python
from typing import Annotated

from fastapi import Depends
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer

bearer_scheme = HTTPBearer(auto_error=False)


async def get_current_user(
    credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer_scheme)],
) -> str:
    # 演示令牌校验；生产环境不得硬编码令牌。
    if credentials is None or credentials.credentials != "demo-token":
        raise HTTPException(status_code=401, detail="缺少或无效的访问令牌")
    return "demo-user"


@app.get("/api/me")
async def get_me(current_user: Annotated[str, Depends(get_current_user)]) -> dict[str, str]:
    # 返回当前用户，验证认证依赖的成功与失败路径。
    return {"user": current_user}
```

CORS 是浏览器是否允许网页读取跨域响应的规则，不是认证。可信前端跨域调用时可配置：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH", "DELETE"],
    allow_headers=["Authorization", "Content-Type", "X-Request-ID"],
)
```

不要在携带 Cookie 或身份凭证的生产服务上使用 `allow_origins=["*"]`。CORS 不能阻止脚本或其他服务器直接调用 API，因此不能替代认证和授权。

错误响应应让调用方知道发生了什么、能否重试、如何定位。日志中记录请求 ID、路径、状态码、耗时和异常上下文；响应不要泄露堆栈、密钥或用户隐私。不要捕获所有异常后仍返回 `200` 和 `{"success": false}`，这会破坏监控和重试语义。

## 8. 异步：`async def` 不会自动加速

异步适合等待网络、文件、数据库等 I/O。协程在 `await` 时让出执行权，使事件循环能处理其他请求；它不等于 CPU 并行。

```python
import asyncio


@app.get("/demo/non-blocking")
async def non_blocking() -> dict[str, str]:
    # 异步等待时让出事件循环。
    await asyncio.sleep(1)
    return {"message": "等待完成"}
```

反例：

```python
import time


@app.get("/demo/blocking")
async def blocking() -> dict[str, str]:
    # 同步 sleep 会阻塞事件循环，影响其他异步请求。
    time.sleep(1)
    return {"message": "等待完成"}
```

异步路由应调用异步 I/O 库并使用 `await`。若模型 SDK、HTTP 客户端或数据库驱动是同步阻塞的，应使用同步路由、线程池或异步客户端；大量 CPU 计算和本地重推理通常应交给独立工作进程或任务队列。

## 9. 流式响应：让用户尽早看到生成内容

完整模型回答可能需要数秒。流式响应将首段内容尽早交给客户端，因此除总耗时外，还要关心 TTFT，也就是首 token 时间。

### 9.1 `StreamingResponse` 分块返回文本

```python
import asyncio
from collections.abc import AsyncIterator

from fastapi.responses import StreamingResponse


async def generate_text(prompt: str) -> AsyncIterator[str]:
    # 模拟逐段生成；实际项目替换为模型 SDK 的异步流式迭代器。
    answer = f"你问的是：{prompt}。这是一个流式返回示例。"
    for character in answer:
        await asyncio.sleep(0.05)
        yield character


@app.get("/api/generate")
async def generate(prompt: str) -> StreamingResponse:
    # 将模拟文本作为分块 HTTP 响应返回。
    return StreamingResponse(generate_text(prompt), media_type="text/plain; charset=utf-8")
```

使用终端验证：

```powershell
curl.exe -N "http://127.0.0.1:8000/api/generate?prompt=什么是流式响应"
```

`-N` 尽量关闭 `curl` 缓冲。若服务端持续 `yield`、浏览器仍最后才显示，应检查客户端缓冲、响应类型、反向代理缓冲和真实生成节奏。代码正确不等于端到端真的在流。

### 9.2 SSE：带事件名的单向流

SSE 是基于 HTTP 的文本事件约定，每个事件由空行结束：

```text
event: token
data: {"text":"你"}

event: done
data: {}

```

服务端实现：

```python
import json


async def sse_events(prompt: str) -> AsyncIterator[str]:
    # 将生成片段包装为 SSE 事件，并发送完成或失败信号。
    try:
        async for chunk in generate_text(prompt):
            payload = json.dumps({"text": chunk}, ensure_ascii=False)
            yield f"event: token\ndata: {payload}\n\n"
        yield "event: done\ndata: {}\n\n"
    except asyncio.CancelledError:
        # 客户端断开后应取消上游模型读取，避免继续消耗 token。
        raise
    except Exception:
        # 生产环境应记录异常与稳定错误码，不能泄露内部堆栈。
        yield "event: error\ndata: {\"message\": \"生成失败\"}\n\n"


@app.get("/api/generate/events")
async def generate_events(prompt: str) -> StreamingResponse:
    # 返回 SSE 格式的生成事件流。
    return StreamingResponse(
        sse_events(prompt),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

浏览器消费：

```html
<script>
  const prompt = encodeURIComponent("解释 SSE");
  const stream = new EventSource(`/api/generate/events?prompt=${prompt}`);

  stream.addEventListener("token", (event) => {
    const { text } = JSON.parse(event.data);
    document.body.append(text);
  });
  stream.addEventListener("done", () => stream.close());
  stream.addEventListener("error", () => stream.close());
</script>
```

原生 `EventSource` 只能发 `GET`，且不方便加自定义 `Authorization` 头。若请求需要复杂 JSON 或 Bearer Token，使用 `fetch` 读取 `ReadableStream`，或设计合适的 Cookie 认证；不要把长期密钥放在查询参数。

客户端点击停止时使用 `AbortController.abort()`。服务端还必须向上游模型、数据库连接和并发信号量传播取消，用户关闭页面不等于模型自动停止。

对 LLM 流建议定义 `meta`、`token`、`tool_call`、`usage`、`error`、`done`。HTTP 响应已经开始后不能把 `200` 改成 `500`，所以流协议内必须有 `error` 事件，客户端收到 `done` 前都不能视为成功。

## 10. WebSocket：双方均可主动发送

单次提问、连续收答案时，SSE 或 `fetch` 流通常更简单。聊天、协作编辑、实时人工接管和客户端需要随时发停止命令时，才使用 WebSocket。

最小回显服务：

```python
from fastapi import WebSocket, WebSocketDisconnect


@app.websocket("/ws/echo")
async def echo_socket(websocket: WebSocket) -> None:
    # 接收客户端文本并回显，演示接受连接和断连处理。
    await websocket.accept()
    try:
        while True:
            message = await websocket.receive_text()
            await websocket.send_json({"type": "echo", "text": message})
    except WebSocketDisconnect:
        return
```

浏览器测试：

```html
<script>
  const socket = new WebSocket("ws://127.0.0.1:8000/ws/echo");
  socket.addEventListener("open", () => socket.send("你好，WebSocket"));
  socket.addEventListener("message", (event) => console.log(JSON.parse(event.data)));
  socket.addEventListener("close", () => console.log("连接已关闭"));
  socket.addEventListener("error", () => console.error("连接异常"));
</script>
```

HTTPS 页面必须使用 `wss://`，不能使用 `ws://`。最小单进程广播器：

```python
class ConnectionManager:
    # 管理当前进程中的连接并提供广播能力。
    def __init__(self) -> None:
        self.connections: set[WebSocket] = set()

    async def connect(self, websocket: WebSocket) -> None:
        await websocket.accept()
        self.connections.add(websocket)

    def disconnect(self, websocket: WebSocket) -> None:
        self.connections.discard(websocket)

    async def broadcast(self, message: dict[str, str]) -> None:
        disconnected: list[WebSocket] = []
        for connection in self.connections:
            try:
                await connection.send_json(message)
            except Exception:
                disconnected.append(connection)
        for connection in disconnected:
            self.disconnect(connection)


manager = ConnectionManager()


@app.websocket("/ws/chat")
async def chat_socket(websocket: WebSocket) -> None:
    await manager.connect(websocket)
    try:
        while True:
            text = await websocket.receive_text()
            await manager.broadcast({"type": "message", "text": text})
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

该管理器只适合单进程。多 worker 或多机器时，每个进程都有独立的连接集合；跨实例广播需要 Redis Pub/Sub、消息队列或专用实时服务。

生产 WebSocket 还应处理：短期身份票据或安全 Cookie、对象级授权、`type` 和 `request_id` 消息协议、心跳、慢客户端背压、消息大小限制、限流、断线重连和连接数监控。浏览器 WebSocket API 不能像 `fetch` 一样自由加入 `Authorization` 头，因此不要用 URL 传长期令牌。

## 11. Flask：同步服务的基本写法

Flask 示例：

```python
# 演示 Flask 的最小同步 REST 服务。
from flask import Flask, jsonify, request

app = Flask(__name__)


@app.get("/health")
def health():
    return jsonify(status="ok")


@app.post("/api/tasks")
def create_task():
    payload = request.get_json(silent=True)
    if not isinstance(payload, dict) or not payload.get("title"):
        return jsonify(error="title 为必填字段"), 400
    return jsonify(id="demo-id", title=payload["title"]), 201
```

启动：

```powershell
pip install flask
flask --app app run --debug
```

Flask 也可用生成器做基础流式响应：

```python
import time
from flask import Response, stream_with_context


@app.get("/api/generate")
def generate():
    @stream_with_context
    def stream():
        for chunk in ["你", "好", "，", "流", "式", "响", "应"]:
            time.sleep(0.1)
            yield chunk

    return Response(stream(), content_type="text/plain; charset=utf-8")
```

同步等待会占用一个工作线程。Flask 的核心是 WSGI 请求响应模型，WebSocket 常需 Flask-Sock、Flask-SocketIO 或 ASGI 方案。新项目若以流式 LLM 和 WebSocket 为重点，FastAPI 的路径更短；既有 Flask 项目则不必无故整体重写。

## 12. 用 pytest 验证 API 契约

创建 `test_main.py`：

```python
from fastapi.testclient import TestClient

from main import app

client = TestClient(app)


def test_create_and_get_task() -> None:
    created = client.post("/api/tasks", json={"title": "写测试"})
    assert created.status_code == 201

    task_id = created.json()["id"]
    fetched = client.get(f"/api/tasks/{task_id}")
    assert fetched.status_code == 200
    assert fetched.json()["title"] == "写测试"


def test_blank_title_is_rejected() -> None:
    response = client.post("/api/tasks", json={"title": ""})
    assert response.status_code == 422


def test_missing_task_returns_404() -> None:
    response = client.get("/api/tasks/not-exists")
    assert response.status_code == 404
```

运行 `pytest -q`。内存状态会在测试间共享；练习变大后，应在每个测试前清空状态，或改用可注入的存储替身。流式接口还要测片段顺序、`done`、`error` 和取消后的资源释放；WebSocket 还要测连接、非法消息、断连和广播范围。

## 13. 上线前检查清单

- HTTPS：认证和用户数据必须经 TLS；WebSocket 使用 `wss://`。
- 超时：入站请求、模型、数据库和外部工具都要有明确超时。
- 重试与幂等：只重试明确的瞬时失败；写入操作使用幂等键，不能盲目重试。
- 限流与并发：按用户、租户和全局限制。LLM 流长期占用连接，应单独估算容量。
- 安全：限制请求体、提示词、文件和 WebSocket 消息大小；工具调用必须做权限和参数校验。
- 可观测性：记录请求 ID、路径、状态码、耗时、模型耗时、TTFT、token 用量和错误码。
- 部署：配置健康检查、优雅停机、代理长连接超时和流式缓冲。
- 版本：破坏性变更使用清晰 API 版本，不能悄悄修改客户端依赖的字段语义。

## 14. 七日练习路线

1. 启动 FastAPI，完成 `/health`，用 `/docs` 和 `curl` 验证。
2. 实现任务 CRUD，逐一验证 `201`、`204`、`404`、`422`。
3. 加入 Pydantic、分页和最小 Bearer Token 依赖。
4. 实现 `StreamingResponse`，确认终端逐段输出。
5. 改为 SSE，补充 `token`、`error`、`done`，并在浏览器消费。
6. 实现 WebSocket 回显和广播，主动关闭页面验证断连。
7. 补充 pytest、日志、超时和取消，记录一次亲自复现并定位的失败。

完成后再将模拟生成器替换为真实模型 SDK。应把 SDK 的片段和异常转换为稳定事件，不能将供应商私有对象直接暴露给前端。

## 15. 最终自检

你应能回答：为什么此处选择 REST、SSE 或 WebSocket；流式响应如何表达中途失败；`async def` 为什么仍可能阻塞；多进程部署后内存广播为何失效；如何证明接口处理了非法输入、断连和上游失败。

能够用协议选择、输入输出契约、失败路径、资源限制和测试方法回答这些问题，才说明你真正具备了构建大模型应用服务端的可靠起点。
