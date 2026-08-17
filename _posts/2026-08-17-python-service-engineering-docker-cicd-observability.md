---
layout: post
title: "Python 服务工程化入门：Docker、CI/CD、分环境配置、日志、指标与告警"
date: 2026-08-17 14:00:00 +0800
categories: [Python基础]
tags: [Python, Docker, CI/CD, FastAPI, 日志, Prometheus, 告警, 工程化]
---

# Python 服务工程化入门：Docker、CI/CD、分环境配置、日志、指标与告警

作为大模型开发工程师，你交付的通常不是一段“能调用模型”的代码，而是一个能被别人稳定使用的服务。它需要能够在开发机、测试环境和生产环境以一致方式运行；出现问题时能定位；性能或错误恶化时能被及时发现；改动通过检查后能可控地上线并在必要时回滚。

本文以一个很小的 Python API 服务为主线，带你理解以下能力如何连接成闭环：

```text
代码 -> 自动化测试 -> Docker 镜像 -> 部署环境 -> 日志/指标 -> 告警 -> 排障与改进
```

示例使用 Python 3.11+、FastAPI、Docker、Docker Compose、GitHub Actions 和 Prometheus 的常见写法。重点是建立正确的工程心智模型；工具可以替换，原则不应丢失。

> 学习边界：文中的 Compose、密钥和部署示例适合本地学习或小型内部服务。真正的生产系统还应补充访问控制、镜像扫描、密钥托管、高可用、备份、变更审批和演练等治理措施。

## 1. 先建立全景：一次请求经历了什么

假设你做了一个“文本摘要”接口：客户端把文章发给服务，服务返回摘要。即使先不接真实大模型，它已经需要处理输入、异常、日志和性能观测。

```text
客户端
  |
  | HTTP POST /summarize
  v
反向代理 / 负载均衡（生产环境常见）
  |
  v
Python API 服务
  |-- 读取运行环境配置
  |-- 校验请求
  |-- 调用模型 API 或本地推理服务
  |-- 记录结构化日志
  |-- 更新延迟、请求量、错误数等指标
  v
响应客户端

CI：每次提交自动检查代码和测试
CD：通过门禁后构建镜像、部署新版本
监控：采集指标，满足规则时向人发出告警
```

这里的每个部件分别解决一种问题：

| 能力 | 要解决的问题 | 一个常见失败信号 |
| --- | --- | --- |
| Docker | “在我电脑上能跑，到服务器却不行” | 依赖、Python 版本或启动命令不一致 |
| 分环境配置 | 开发、测试、生产的地址和开关不同 | 测试环境误连生产数据库 |
| CI | 人工检查不稳定，坏代码进入主分支 | 合并后才发现语法或测试失败 |
| CD | 发布步骤依赖手工操作且不可追溯 | 不知道线上运行的是哪次提交 |
| 日志 | 单次请求为什么失败 | 只有“500 错误”，没有上下文 |
| 指标 | 系统整体是否变差 | 用户先发现“最近变慢了” |
| 告警 | 重要问题要及时触达负责人 | 指标红了但没有人注意 |

不要把它们当成七个孤立名词。它们共同构成服务的反馈回路：**部署后用日志和指标观察，异常时告警，找到原因后修复，再通过 CI/CD 发布。**

## 2. 学习项目：一个可观测的摘要 API

先约定一个最小项目目录。你可以把它作为后续练习的骨架：

```text
summary-service/
├── app/
│   ├── __init__.py
│   ├── config.py
│   └── main.py
├── tests/
│   └── test_main.py
├── .dockerignore
├── .env.example
├── Dockerfile
├── compose.yaml
├── requirements.txt
└── .github/workflows/ci.yml
```

其中 `app/main.py` 负责 API 和运行时观测，`app/config.py` 负责读取配置。测试、镜像构建和部署配置都围绕这个服务组织。

### 2.1 安装依赖

`requirements.txt`：

```text
fastapi==0.115.12
uvicorn[standard]==0.34.2
pydantic-settings==2.8.1
prometheus-client==0.21.1
pytest==8.3.5
httpx==0.28.1
```

在本地创建虚拟环境并安装依赖：

```bash
python -m venv .venv
# Windows PowerShell
.\.venv\Scripts\Activate.ps1
# macOS / Linux
source .venv/bin/activate

python -m pip install -r requirements.txt
```

### 2.2 用环境变量表达配置

配置不是“写在代码里的常量”。同一份代码在不同环境需要不同的端口、模型地址、日志等级和功能开关。把这些可变值放入环境变量，代码只定义**名称、类型、默认值和校验规则**。

`app/config.py`：

```python
"""集中定义服务的运行时配置，并从环境变量加载。"""

from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """服务运行时配置。

    所有值均可由同名环境变量覆盖。开发环境可从本地 .env 文件读取；
    生产环境应由部署平台或专用密钥管理服务注入，不应把密钥提交到仓库。
    """

    app_env: str = "development"
    app_version: str = "dev"
    log_level: str = "INFO"
    model_base_url: str = "http://localhost:8001"
    request_timeout_seconds: float = 20.0
    enable_demo_summary: bool = True
    api_key: str | None = None

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",
    )


@lru_cache
def get_settings() -> Settings:
    """创建并缓存配置对象，避免在每个请求中重复读取环境变量。"""

    return Settings()
```

初学时最容易混淆三件事：

- `.env` 是**本地开发便利文件**，不是安全边界；它通常不提交到 Git。
- 环境变量是**进程启动时收到的配置输入**；Docker、CI 和云平台都能提供它。
- 配置对象是**Python 代码中经过类型校验后的访问入口**；业务代码不要到处调用 `os.getenv()`。

提供一个可以提交的模板 `.env.example`：

```dotenv
APP_ENV=development
APP_VERSION=dev
LOG_LEVEL=INFO
MODEL_BASE_URL=http://localhost:8001
REQUEST_TIMEOUT_SECONDS=20
ENABLE_DEMO_SUMMARY=true
# API_KEY=仅在本地复制为 .env 后填写，绝不提交真实值
```

把 `.env` 加入 `.gitignore`。尤其不要把模型提供商 API Key、数据库密码、Webhook 密钥写进 Git 历史；即使删掉文件，已经推送的历史也可能仍含泄露内容。

### 2.3 最小 API、健康检查与指标入口

`app/main.py`：

```python
"""提供摘要接口、健康检查、结构化日志与 Prometheus 指标。"""

import json
import logging
import time
import uuid
from contextlib import asynccontextmanager

from fastapi import FastAPI, HTTPException, Request
from pydantic import BaseModel, Field
from prometheus_client import Counter, Histogram, make_asgi_app

from app.config import get_settings

REQUEST_COUNT = Counter(
    "http_requests_total",
    "HTTP 请求总数",
    ["method", "path", "status"],
)
REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP 请求处理耗时（秒）",
    ["method", "path"],
)


class JsonFormatter(logging.Formatter):
    """将关键日志字段编码为一行 JSON，方便日志系统检索。"""

    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "timestamp": self.formatTime(record, "%Y-%m-%dT%H:%M:%S%z"),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "request_id": getattr(record, "request_id", None),
            "path": getattr(record, "path", None),
            "status_code": getattr(record, "status_code", None),
            "duration_ms": getattr(record, "duration_ms", None),
        }
        return json.dumps(payload, ensure_ascii=False)


def configure_logging() -> None:
    """把应用日志输出到标准输出，交由容器平台统一采集。"""

    settings = get_settings()
    handler = logging.StreamHandler()
    handler.setFormatter(JsonFormatter())
    logging.basicConfig(level=settings.log_level, handlers=[handler], force=True)


class SummarizeRequest(BaseModel):
    """摘要请求的输入结构。"""

    text: str = Field(min_length=1, max_length=10_000)


class SummarizeResponse(BaseModel):
    """摘要请求的输出结构。"""

    summary: str


@asynccontextmanager
async def lifespan(_: FastAPI):
    """在服务开始接收流量前完成一次性初始化。"""

    configure_logging()
    yield


app = FastAPI(title="Summary Service", lifespan=lifespan)
app.mount("/metrics", make_asgi_app())
logger = logging.getLogger(__name__)


@app.middleware("http")
async def observe_request(request: Request, call_next):
    """为每个请求生成关联 ID、记录耗时，并更新基础 HTTP 指标。"""

    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    start_time = time.perf_counter()
    status_code = 500

    try:
        response = await call_next(request)
        status_code = response.status_code
        response.headers["X-Request-ID"] = request_id
        return response
    finally:
        duration_seconds = time.perf_counter() - start_time
        path = request.url.path
        REQUEST_COUNT.labels(request.method, path, str(status_code)).inc()
        REQUEST_DURATION.labels(request.method, path).observe(duration_seconds)
        logger.info(
            "request_completed",
            extra={
                "request_id": request_id,
                "path": path,
                "status_code": status_code,
                "duration_ms": round(duration_seconds * 1000, 2),
            },
        )


@app.get("/healthz")
async def healthz() -> dict[str, str]:
    """报告进程是否存活。

    它只回答“Web 进程是否能响应”，不代表数据库、模型服务等依赖一定可用。
    """

    return {"status": "ok"}


@app.get("/readyz")
async def readyz() -> dict[str, str]:
    """报告实例是否准备接收流量。

    真正接入模型或数据库后，应在此处做带超时的轻量依赖检查，
    并把失败转换为 503，使流量调度器暂时不把请求发到此实例。
    """

    return {"status": "ready"}


@app.post("/summarize", response_model=SummarizeResponse)
async def summarize(payload: SummarizeRequest) -> SummarizeResponse:
    """返回演示用摘要，后续可替换为受超时和重试保护的模型调用。"""

    settings = get_settings()
    if not settings.enable_demo_summary:
        raise HTTPException(status_code=503, detail="摘要能力暂不可用")

    normalized_text = " ".join(payload.text.split())
    summary = normalized_text[:100]
    if len(normalized_text) > 100:
        summary += "..."
    return SummarizeResponse(summary=summary)
```

启动服务：

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

然后分别访问：

```bash
curl http://127.0.0.1:8000/healthz
curl http://127.0.0.1:8000/readyz
curl http://127.0.0.1:8000/metrics
curl -X POST http://127.0.0.1:8000/summarize \
  -H "Content-Type: application/json" \
  -d '{"text":"这是一段需要被摘要的较长文本。"}'
```

你会看到：API 响应中有 `X-Request-ID`，终端输出一行 JSON 日志，`/metrics` 中出现 `http_requests_total` 与 `http_request_duration_seconds`。这正是后续排障所需的最小证据链。

## 3. Docker：把“运行环境”也纳入交付物

### 3.1 容器与镜像到底是什么

可以先记住一个实用类比：

- **Dockerfile**：制作说明书，定义从什么基础环境开始、安装什么、如何启动。
- **镜像（image）**：按说明书制作好的、不可变的运行包。
- **容器（container）**：某个镜像启动后的运行实例。
- **镜像仓库（registry）**：存放并分发镜像的地方，例如 GitHub Container Registry、Docker Hub 或企业仓库。

容器并不是虚拟机。它通常与宿主机共享内核，但拥有相对隔离的进程、网络和文件系统视图。它解决的核心问题是**环境一致性和可重复交付**，不是天然安全、天然高可用或天然不出故障。

### 3.2 编写 Dockerfile

项目根目录的 `Dockerfile`：

```dockerfile
# 固定主版本，避免每次构建意外切换到不兼容的 Python 大版本。
FROM python:3.11-slim

# 让日志及时输出，避免 Python 产生无用的 .pyc 文件。
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# 先复制依赖清单，利用 Docker 的构建缓存。
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# 最后复制业务代码；代码改变时只重建这一层及后续层。
COPY app ./app

# 容器内部监听的端口。EXPOSE 是说明，不会自动把端口暴露到宿主机。
EXPOSE 8000

# 一个容器通常只承担一个主要进程。
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

配套的 `.dockerignore`：

```dockerignore
.git
.github
.venv
__pycache__/
.pytest_cache/
.env
*.pyc
*.pyo
*.pyd
tests/
```

`.dockerignore` 决定哪些文件不会被发送给 Docker 构建上下文。它能让构建更快，也能降低把本地虚拟环境、测试缓存和 `.env` 中密钥意外打进镜像的风险。它不是万能防泄露手段，真正的密钥仍不应出现在构建上下文中。

构建和运行：

```bash
# -t 是镜像标签。开发时可使用本地名称。
docker build -t summary-service:dev .

# -p 左侧是主机端口，右侧是容器端口。
docker run --rm -p 8000:8000 \
  -e APP_ENV=development \
  -e APP_VERSION=local \
  summary-service:dev
```

验证：

```bash
curl http://127.0.0.1:8000/healthz
# 查看容器日志；<container_id> 从 docker ps 获取。
docker logs <container_id>
```

### 3.3 初学者常见 Docker 误区

| 误区 | 为什么有问题 | 正确做法 |
| --- | --- | --- |
| 把 `.env` 用 `COPY` 打进镜像 | 密钥进入镜像层，可能被推送到仓库 | 在运行/部署时注入密钥 |
| 代码变了就重装全部依赖 | 构建慢，缓存失效 | 先复制依赖清单，再复制源码 |
| 用 `latest` 当生产版本 | 无法准确知道实际部署内容，也难回滚 | 使用 Git SHA、语义版本或构建号标签 |
| 把日志写入容器内文件 | 容器重建后文件易丢失，采集困难 | 业务日志写标准输出/错误输出 |
| 一个容器启动多个守护进程 | 退出、重启、日志与健康管理变复杂 | 一个容器一个主要职责 |
| 以 root 运行所有服务 | 权限过大，攻击面更高 | 生产镜像创建非 root 用户运行 |

生产镜像还建议增加非 root 用户、依赖锁定与安全扫描。学习阶段先确保你能解释“每一行 Dockerfile 的目的”和“镜像标签为何可追溯”。

## 4. Docker Compose：一次启动多个相关服务

当 API、Prometheus、数据库、Redis 等要一起运行时，逐个输入 `docker run` 很容易漏参数。Docker Compose 用一个 YAML 文件描述这些**本地协作服务**。

`compose.yaml`：

```yaml
services:
  api:
    build:
      context: .
    image: summary-service:${APP_VERSION:-dev}
    environment:
      APP_ENV: ${APP_ENV:-development}
      APP_VERSION: ${APP_VERSION:-dev}
      LOG_LEVEL: ${LOG_LEVEL:-INFO}
      ENABLE_DEMO_SUMMARY: ${ENABLE_DEMO_SUMMARY:-true}
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/healthz')"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 10s

  prometheus:
    image: prom/prometheus:v2.55.1
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    depends_on:
      api:
        condition: service_healthy
```

对应的 `prometheus.yml`：

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: summary-service
    metrics_path: /metrics
    static_configs:
      - targets: ["api:8000"]
```

运行：

```bash
docker compose up --build
```

然后访问 `http://127.0.0.1:9090`，在查询框输入：

```promql
http_requests_total
```

再连续调用几次 `/summarize`，你会看到时间序列增长。停止服务使用：

```bash
docker compose down
```

`down` 会移除 Compose 创建的容器和网络；如果额外加 `-v`，还会删除关联卷。对于保存数据库数据的实验环境，使用 `-v` 前要明确它会清除数据。

## 5. 分环境配置：同一份代码，不同的运行参数

### 5.1 为什么至少要区分环境

典型环境及职责：

| 环境 | 主要用途 | 可接受的风险 | 不应做的事 |
| --- | --- | --- | --- |
| development（开发） | 本地快速调试 | 使用模拟服务、调试日志 | 连接生产数据或使用生产密钥 |
| test / CI（测试） | 自动化验证 | 每次创建临时数据 | 依赖人工保留状态 |
| staging（预发布） | 接近生产的验收 | 小范围真实流程验证 | 误当生产稳定性保证 |
| production（生产） | 服务真实用户 | 变更受控、可回滚 | 用开发配置或调试密钥运行 |

环境不是文件夹名，而是一组明确的运行策略。`APP_ENV=production` 本身不会让系统更安全；它只有在代码、部署平台和运维规则都据此采取不同策略时才有意义。

### 5.2 该配置什么，不该配置什么

适合通过环境变量配置的内容：

- 服务端口、日志级别、运行环境名；
- 数据库和外部服务地址；
- 超时、限流阈值、功能开关；
- 镜像版本、模型名称、Prompt 版本；
- 密钥的引用或密钥本身（由受控平台注入）。

不适合当作环境变量的内容：

- 每一条业务规则；
- 大段 Prompt 或大份 JSON；
- 会频繁变化且需要审计的业务数据；
- 需要多人协作编辑的复杂配置。

后几类数据应进入版本化配置文件、数据库或专用配置服务，并配套审批和审计。不要把“所有东西都塞进环境变量”误解为工程化。

### 5.3 配置优先级与密钥处理

建议定义清楚优先级，例如：

```text
部署平台显式环境变量 > 本地 .env 文件 > 代码默认值
```

这样本地可以快速启动，CI 或生产可以覆盖关键值。密钥处理至少遵守以下规则：

1. 仓库只提交 `.env.example`，不提交 `.env`。
2. CI 通过平台的 Secrets 功能注入密钥，不把值打印到日志。
3. 生产优先使用云厂商或企业的 Secret Manager / Vault 一类密钥服务。
4. 密钥必须可轮换；泄露后立即废弃旧值、生成新值并检查访问记录。
5. 日志和错误响应中不得输出 `Authorization`、Cookie、API Key、用户隐私文本或完整模型 Prompt。

## 6. CI：每次改动都要经过可重复的质量门禁

### 6.1 CI 与 CD 的区别

- **持续集成（CI，Continuous Integration）**：代码提交或发起合并请求后，自动安装依赖、运行检查和测试，证明这次变更至少满足既定质量门槛。
- **持续交付/部署（CD）**：将已经通过门禁的版本构建、发布并部署。Continuous Delivery 往往指“随时可发布，生产发布可人工批准”；Continuous Deployment 指“通过门禁后自动进入生产”。

初学阶段最重要的不是记住英文全称，而是能回答：**这次上线的代码是否经过测试？使用哪个版本？失败时如何停止或回滚？**

### 6.2 先写一个接口测试

`tests/test_main.py`：

```python
"""验证摘要 API 的关键输入输出与健康检查。"""

from fastapi.testclient import TestClient

from app.main import app

client = TestClient(app)


def test_healthz_returns_ok() -> None:
    """存活检查应返回成功状态。"""

    response = client.get("/healthz")

    assert response.status_code == 200
    assert response.json() == {"status": "ok"}


def test_summarize_returns_prefix() -> None:
    """有效文本应返回确定的演示摘要。"""

    response = client.post("/summarize", json={"text": "  Python   服务工程化  "})

    assert response.status_code == 200
    assert response.json() == {"summary": "Python 服务工程化"}


def test_summarize_rejects_empty_text() -> None:
    """空文本不应进入业务处理。"""

    response = client.post("/summarize", json={"text": ""})

    assert response.status_code == 422
```

本地验证：

```bash
pytest -q
```

测试不是“写一次就完成”。当你将演示摘要替换为真实模型调用时，应增加超时、上游 5xx、返回格式错误、用户取消和限流等失败场景测试。

### 6.3 GitHub Actions 最小 CI

`.github/workflows/ci.yml`：

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4

      - name: 安装 Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: 安装依赖
        run: python -m pip install --upgrade pip && pip install -r requirements.txt

      - name: 运行测试
        run: pytest -q

      - name: 构建 Docker 镜像
        run: docker build -t summary-service:${{ github.sha }} .
```

这段工作流做了四件可验证的事：检出某个提交、固定 Python 主版本、运行测试、验证 Dockerfile 能构建。`github.sha` 是当前提交的唯一标识，把它用于镜像标签能让“部署了什么”可追溯。

CI 常见演进顺序：

```text
语法/格式检查 -> 单元测试 -> 类型检查 -> 构建镜像 -> 集成测试 -> 安全扫描 -> 发布门禁
```

不要一开始就堆满工具。先让 `pytest` 与镜像构建稳定运行；当团队已约定格式化、类型或依赖扫描工具时，再把它们纳入 CI。否则只会得到经常失败、无人维护的流水线。

## 7. CD：把“发布”变成可追溯、可回滚的流程

### 7.1 一个安全的发布链路

下图是概念流程，具体平台可以是 Docker Compose、Kubernetes、云容器服务或企业平台：

```text
Pull Request
  -> CI 测试、构建、审查
  -> 合并 main
  -> 构建不可变镜像（标签为 Git SHA）
  -> 推送镜像仓库
  -> 部署到 staging
  -> 健康检查与验收
  -> 生产审批或自动门禁
  -> 小流量发布 / 全量发布
  -> 观测指标与错误日志
```

关键原则：**生产环境部署的是构建好的镜像，不是在服务器上再 `git pull` 和 `pip install`。**否则你难以证明服务器运行的依赖与 CI 验证的是同一份内容。

### 7.2 镜像版本与回滚

推荐至少同时使用两类标签：

```text
registry.example.com/summary-service:git-abc1234
registry.example.com/summary-service:2026.08.17-42
```

不要只部署 `latest`。回滚的实质是把部署声明重新指向一个**已经验证过的旧镜像标签**，而不是临时改代码碰碰运气。

每次部署应该被记录为：

```text
时间、环境、镜像 digest / Git SHA、配置版本、操作者或流水线运行编号、结果
```

如果线上错误率在新版本部署后立刻上升，这些记录能回答“变更是什么、何时发生、如何回滚”。

### 7.3 Docker Compose 的简单部署示意

学习或小型单机服务可以在服务器保存一个仅描述运行态的 Compose 文件：

```yaml
services:
  api:
    image: ghcr.io/your-org/summary-service:${IMAGE_TAG}
    restart: unless-stopped
    env_file:
      - /opt/summary-service/production.env
    ports:
      - "127.0.0.1:8000:8000"
```

部署脚本的核心动作是：

```bash
export IMAGE_TAG=git-abc1234
docker compose pull api
docker compose up -d --no-deps api
curl --fail --retry 5 http://127.0.0.1:8000/healthz
```

注意：这只是帮助你理解发布动作。生产环境还要处理远程访问授权、反向代理 TLS、密钥权限、备份、发布锁、防止并发部署、失败自动回退和审计。不要把服务器 SSH 私钥直接写入仓库或聊天记录；CI 使用的部署凭据应具有最小权限。

## 8. 日志：解释“某一次请求发生了什么”

### 8.1 日志、指标、追踪的分工

三者经常一起出现，但回答的问题不同：

| 信号 | 最适合回答的问题 | 示例 |
| --- | --- | --- |
| 日志（Logs） | 一次请求为何失败、具体上下文是什么 | 哪个 request_id 调用模型超时？ |
| 指标（Metrics） | 系统整体是否变慢、错误是否增加 | 过去 5 分钟 5xx 比例是多少？ |
| 链路追踪（Traces） | 跨服务调用在哪一段慢 | API、检索、模型调用各耗时多少？ |

小服务先把结构化日志和 HTTP 指标做好即可。服务开始调用数据库、向量库、模型网关等多个组件时，再学习 OpenTelemetry 分布式追踪。

### 8.2 什么是结构化日志

下面是一条容易被机器检索的 JSON 日志：

```json
{
  "timestamp": "2026-08-17T14:05:31+0800",
  "level": "INFO",
  "message": "request_completed",
  "request_id": "a1b2c3",
  "path": "/summarize",
  "status_code": 200,
  "duration_ms": 125.4
}
```

相比 `用户请求成功，耗时 125ms`，JSON 的好处是日志平台可以按 `request_id`、`status_code`、`path` 和时间范围做过滤、聚合和告警。

日志应该包含足以定位问题的上下文：

- 时间、级别、服务名、环境、版本；
- `request_id` 或 `trace_id`；
- 请求路径、方法、状态码、耗时；
- 上游服务名、错误类型、重试次数；
- LLM 场景常见的模型名、Prompt 版本、token 用量、是否命中缓存。

日志不应该包含：

- 密码、Token、Cookie、Authorization 头；
- 用户身份证号、手机号、完整敏感对话；
- 无需排障的整段 Prompt 或完整模型输出；
- 未经脱敏的数据库连接串。

一个实用规则是：**先问“排障是否真的需要这个字段”，再决定是否记录；如果必须记录敏感内容，优先记录摘要、长度、哈希或脱敏后的版本。**

### 8.3 日志级别怎么用

| 级别 | 使用场景 | 示例 |
| --- | --- | --- |
| DEBUG | 开发定位细节，生产一般关闭 | 重试前的请求参数摘要 |
| INFO | 正常但值得记录的业务事件 | 请求完成、配置加载、版本启动 |
| WARNING | 当前可恢复，但值得关注 | 第 2 次重试成功、依赖响应变慢 |
| ERROR | 当前操作失败，需要排查 | 模型调用超时、数据库写入失败 |
| CRITICAL | 服务或核心能力严重不可用 | 无法启动、关键配置缺失 |

避免两种极端：所有事情都记 `INFO` 会制造噪音；捕获异常却只写“出错了”又没有任何诊断价值。记录错误时应带上错误类型、关联 ID 和必要上下文，并保留异常堆栈。

## 9. 指标：从“单次证据”升级为“整体趋势”

### 9.1 四类最基础的服务指标

为了先把最重要的事情学清楚，可以使用 RED 方法：

- **R（Rate）请求速率**：单位时间处理多少请求。
- **E（Errors）错误率**：多少请求失败，失败类型是什么。
- **D（Duration）耗时**：平均值、P50、P95、P99 延迟。

再加上一个可用性指标：健康检查和实例数。对于 LLM 服务，还要增加上游模型调用失败率、首 Token 延迟、总生成耗时、输入/输出 token、排队长度和成本等。

| 指标 | 解释 | 为什么重要 |
| --- | --- | --- |
| `http_requests_total` | 请求计数器 | 计算 QPS、成功率、错误率 |
| `http_request_duration_seconds` | 请求耗时直方图 | 观察 P95/P99，而非只看平均值 |
| 进程存活/健康状态 | 实例能否响应 | 发现进程崩溃或未就绪 |
| 上游调用错误数 | 模型或数据库依赖是否异常 | 区分自身代码与依赖故障 |
| token 与费用 | LLM 请求资源消耗 | 发现成本突增或异常输入 |

### 9.2 为什么平均延迟不够

假设 100 个请求里，99 个用了 100ms，1 个用了 10 秒。平均值约 199ms，看起来“还行”，但那位等了 10 秒的用户体验很差。

- **P50**：一半请求不超过这个耗时，接近“典型用户”。
- **P95**：95% 请求不超过这个耗时，反映多数较慢用户。
- **P99**：99% 请求不超过这个耗时，关注尾部延迟和极慢请求。

Prometheus 中可用直方图近似计算 P95：

```promql
histogram_quantile(
  0.95,
  sum by (le) (
    rate(http_request_duration_seconds_bucket{path="/summarize"}[5m])
  )
)
```

不必一开始背下 PromQL。先理解：`rate(...[5m])` 是观察最近 5 分钟的增长速度；`sum by (le)` 汇总各桶；`histogram_quantile(0.95, ...)` 估计 P95。

### 9.3 指标设计的边界：不要高基数标签

Prometheus 的 label 适合低基数、有限集合，例如：

```text
method=GET/POST
path=/healthz、/summarize
status=200、422、500
model=gpt-4.1-mini、qwen-plus
```

不要把每个用户 ID、请求 ID、完整 URL、会话 ID、文章标题作为 label：

```text
# 错误示例：每个用户产生一条新时间序列。
requests_total{user_id="user-123456"}
```

这会让时间序列数量爆炸，最终使监控系统内存和查询性能恶化。单次请求的唯一 ID 应写进日志或追踪，不应作为指标 label。

## 10. 告警：让重要异常在合适的时间找到合适的人

### 10.1 告警不是“任何异常都通知”

告警必须具备可操作性：收到后，值班或负责人知道要看什么、如何止损、何时升级。没有行动价值的告警会造成告警疲劳，最终真正事故也会被忽略。

可以把信号分为：

| 类型 | 例子 | 处理方式 |
| --- | --- | --- |
| 页面级告警（Page） | 核心接口大面积不可用 | 立即通知值班人员 |
| 工单/通知级告警（Ticket） | 磁盘缓慢增长、成本日趋势异常 | 在工作时间处理 |
| 仪表盘观察项 | 单个低频 4xx | 暂不打扰人，保留趋势 |

### 10.2 两条入门告警规则

Prometheus 规则文件示意：

```yaml
groups:
  - name: summary-service
    rules:
      - alert: SummaryServiceHighServerErrorRate
        expr: |
          sum(rate(http_requests_total{path="/summarize",status=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{path="/summarize"}[5m])) > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "摘要接口 5xx 错误率超过 5%"
          runbook: "检查最近发布、应用错误日志和模型上游状态"

      - alert: SummaryServiceHighP95Latency
        expr: |
          histogram_quantile(0.95, sum by (le) (
            rate(http_request_duration_seconds_bucket{path="/summarize"}[5m])
          )) > 3
        for: 15m
        labels:
          severity: ticket
        annotations:
          summary: "摘要接口 P95 延迟持续超过 3 秒"
          runbook: "检查模型耗时、并发、队列和依赖服务延迟"
```

这里的 `for: 10m` 很重要：它表示条件要连续满足 10 分钟才触发，防止一次短暂抖动把人从休息中叫醒。阈值 `5%`、`3 秒` 只是示例，真实阈值必须结合基线、用户承诺（SLO）和业务影响设定。

### 10.3 告警要配 Runbook

每条告警最好有一份短小操作手册（Runbook），至少写清：

1. 这条告警代表什么用户影响。
2. 先看哪个仪表盘、哪个日志查询。
3. 常见原因和初步排查顺序。
4. 临时止损手段，例如回滚、降级、扩容、熔断上游。
5. 何时升级给哪个团队。

例如“5xx 错误率升高”的排查顺序可以是：

```text
1. 确认告警范围：只有 /summarize，还是所有接口都失败？
2. 对比新版本部署时间：是否与异常开始时间一致？
3. 用 request_id 查询 ERROR 日志，按异常类型聚类。
4. 检查模型 API 的超时、限流和 5xx 指标。
5. 若确认与刚发布版本相关，回滚到上一个已验证镜像。
6. 若上游故障，启用降级响应或限流，并同步状态。
```

## 11. 把这些能力放入一次真实故障中理解

假设上午 10:00 发布了新版本，10:05 “摘要接口 5xx 错误率超过 5%”触发。

```text
10:00  CD 部署镜像 git-abc1234
10:05  指标发现 5xx 错误率持续升高，告警触发
10:06  值班查看仪表盘，发现只有 /summarize 异常
10:07  使用 request_id 搜索 JSON 日志，看到模型 URL 配置为空
10:09  对比部署配置，发现生产环境漏注入 MODEL_BASE_URL
10:11  修复受控配置并重新部署，或回滚到旧镜像
10:15  错误率恢复；记录时间线、根因和防复发动作
```

其中各组件的作用很清楚：

- CD 的镜像标签告诉你上线了什么版本；
- 指标和告警让问题在用户大量投诉前被发现；
- 结构化日志和 `request_id` 让你定位具体失败原因；
- 分环境配置让你发现故障是配置遗漏，而不是代码本身；
- CI 可以在之后增加“生产必填配置校验”的测试或部署前检查，防止同类问题再次发生。

这就是工程化的真正价值：不是“永远不出错”，而是让错误可发现、可定位、可止损、可防复发。

## 12. 面向 LLM 服务的额外关注点

通用 Web 服务的实践是基础，大模型服务还应特别观测：

| 维度 | 推荐记录/指标 | 目的 |
| --- | --- | --- |
| 模型调用 | 模型名、区域、供应商、超时、重试、错误码 | 判断依赖故障和路由效果 |
| 性能 | TTFT、端到端耗时、tokens/s、排队时间 | 分解“为什么慢” |
| 成本 | 输入 token、输出 token、缓存命中、估算费用 | 防止成本失控 |
| 质量 | 结构化输出校验失败率、工具调用失败率、人工转接率 | 发现“能响应但没完成任务” |
| 安全 | 被拒绝请求数、权限拒绝数、敏感内容命中数 | 发现攻击或策略误伤 |

特别注意：完整 Prompt 和模型输出往往含业务或用户敏感信息。默认只记录长度、哈希、模板版本、模型版本、token、耗时和脱敏后的错误摘要；只有在有明确权限、保留周期和审计机制时才采样原文。

## 13. 从零开始的实践清单

建议按下面顺序练习，每一步都要独立验证：

1. **本地 API**：用 FastAPI 实现 `/healthz` 和一个带输入校验的业务接口；用 `pytest` 验证正常、空值和非法值。
2. **配置**：将端口、日志级别和模型地址改为环境变量；用 `.env.example` 说明本地必填项；确认 `.env` 未被 Git 跟踪。
3. **Docker**：编写 Dockerfile 与 `.dockerignore`；执行 `docker build`、`docker run`；在主机上访问健康检查。
4. **Compose**：用 `docker compose up --build` 启动 API 与 Prometheus；确认 Prometheus 能抓到 `/metrics`。
5. **日志**：为每个请求生成或透传 `X-Request-ID`；在日志中搜索一次失败请求；确认日志未包含密钥。
6. **指标**：调用接口制造成功、422 和 500（可临时增加受控测试分支）；在 Prometheus 查看请求数、错误率和延迟。
7. **CI**：添加 GitHub Actions，确保每次 push 或 Pull Request 都跑测试和镜像构建；故意改坏一个断言，确认 CI 会失败。
8. **CD 概念练习**：为镜像加 Git SHA 标签，写出“部署此标签”和“回滚到上一标签”的命令与记录格式。
9. **告警**：本地配置一条错误率规则，理解 `expr`、`for`、`labels`、`annotations` 分别做什么。
10. **复盘**：人为制造一次配置错误或上游超时，按日志、指标、告警、修复、测试补强的顺序走完闭环。

## 14. 建议的六周学习路线

| 周次 | 学习目标 | 可验证产物 |
| --- | --- | --- |
| 第 1 周 | FastAPI、异常处理、pytest、环境变量 | 一个有健康检查和 3 类测试的 API |
| 第 2 周 | Dockerfile、镜像、容器、端口、日志 | 容器化 API 可在本机访问 |
| 第 3 周 | Compose 与本地多服务联调 | API + Prometheus 一键启动 |
| 第 4 周 | 结构化日志、request_id、基础 RED 指标 | 能定位一次模拟 500，能看 P95 |
| 第 5 周 | GitHub Actions CI、镜像标签 | 每次提交自动测试并构建镜像 |
| 第 6 周 | CD、回滚、告警与故障演练 | 一份发布流程和一份告警 Runbook |

完成后，再把演示摘要替换成真实 LLM 调用：加请求超时、有限次数重试、并发控制、模型错误分类、token/成本指标，以及安全的 API Key 管理。不要跳过基础服务的可观测性；模型调用越昂贵、越不稳定，就越需要这些能力。

## 15. 自测题

在进入下一个主题前，试着不看答案解释下面问题：

1. Docker 镜像与容器的区别是什么？为什么生产部署应使用明确镜像标签而非 `latest`？
2. 为什么 `.env` 不应提交到 Git？生产环境的密钥应如何提供给进程？
3. CI 与 CD 分别在哪个阶段拦住什么问题？
4. `/healthz` 与 `/readyz` 的含义为什么不完全一样？
5. 日志里的 `request_id` 能解决什么问题？为什么不应记录完整 Authorization 头？
6. P95 延迟与平均延迟分别反映什么？
7. 为什么不能把 `user_id` 作为 Prometheus label？
8. 一条可操作告警至少应包含哪些信息？
9. 发现新版本导致错误率上升时，为什么回滚通常比“在线临时修改服务器代码”更可靠？
10. 对一个 LLM 服务，除了 HTTP 状态码和延迟，还应该观测哪些质量、成本和安全信号？

如果这些问题能结合自己的练习项目回答清楚，你已经具备了“大模型服务能稳定上线和维护”的第一层工程基础。下一步可以继续学习反向代理、数据库、Redis、异步并发、OpenTelemetry、容器编排与灰度发布，但始终坚持同一条主线：每一个改动都要可验证，每一次故障都要留下可定位、可复盘的证据。
