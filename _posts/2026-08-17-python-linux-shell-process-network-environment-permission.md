---
layout: post
title: "Python 工程基础讲义：Linux、Shell、进程、网络、环境变量与权限"
date: 2026-08-17 11:00:00 +0800
categories: [Python基础]
tags: [Python, Linux, Shell, 进程管理, 网络排障, 环境变量, 权限管理]
---

# Python 工程基础讲义：Linux、Shell、进程、网络、环境变量与权限

大模型应用不是一段孤立的 Python 代码。一次请求会经过域名解析、网络路由、端口监听、反向代理和 Python 进程；进程还要读取配置与密钥，并以某个操作系统用户身份访问文件、数据库和网络。理解这些层，才能把“服务不可用”变成可验证的问题，而不是反复重启和猜测。

本讲义面向初学者，以 Ubuntu / Debian 常见环境为例。其他 Linux 发行版的核心概念相同，只是包管理器和个别服务名不同。命令中的 <尖括号> 是占位符，运行前必须替换为真实值。请先在个人开发机、虚拟机或 WSL 练习；生产环境应先观察和记录，再做最小改动。

## 1. 学习目标与排障总图

完成后，你应能：

- 理解 Linux 的目录、路径、日志与文件元信息。
- 使用 Shell 执行、组合和检查命令，并写基础健康检查脚本。
- 确认 Python 服务是否运行、由谁启动、监听什么端口，以及如何优雅停止。
- 按 DNS、路由、端口、TLS/HTTP、应用日志的顺序排查网络问题。
- 正确使用环境变量和密钥，不将敏感内容写进仓库、命令历史或日志。
- 理解用户、组、读写执行权限，定位 Permission denied。
- 用 systemd 管理一个非 root 身份运行的 Python 服务。

一次 LLM 服务请求可以拆成：

~~~text
客户端 -> DNS 域名解析 -> 网络路由 / 防火墙 -> TCP 端口监听
       -> 反向代理 -> Python 服务 -> 模型 API、数据库或文件
       -> 日志、指标与响应
~~~

排障不是随机试命令。每一层都要问一个可验证的问题：

| 层次 | 应问的问题 | 常用证据 |
| --- | --- | --- |
| 名称 | 域名是否解析为预期 IP？ | getent、dig、nslookup |
| 路由 | 本机是否知道如何到达目标 IP？ | ip route、ping、traceroute |
| 传输 | 目标端口可否建立连接？ | ss、nc、curl |
| 协议 | TLS、HTTP 状态和时序如何？ | curl -v、响应头 |
| 应用 | Python 进程收到请求后发生什么？ | journalctl、应用日志、trace id |
| 配置/权限 | 它读到正确配置且有权访问资源吗？ | systemctl show、id、ls -l |

模型 API 超时不等于模型故障。它可能来自 DNS、代理、证书、超时配置、连接池、权限或应用阻塞。先定位失败层，再修改。

## 2. Linux 文件系统与路径

Linux 使用一棵目录树，根目录是 /。常用目录：

| 路径 | 职责 | 实务提示 |
| --- | --- | ---|
| /home/<用户> | 普通用户个人目录 | 开发项目和虚拟环境通常放这里 |
| /etc | 系统与服务配置 | 改动前确认影响并备份 |
| /var/log | 日志 | 服务故障的重要证据 |
| /var/lib | 服务持久化数据 | 不要当作临时目录清理 |
| /tmp | 临时文件 | 可能被清理，不能保存重要状态 |
| /usr/bin | 常用系统程序 | 通常在 PATH 中 |
| /proc | 内核导出的状态 | 虚拟文件系统，不是普通磁盘 |

基础操作：

~~~bash
pwd                         # 当前工作目录
ls -lah                     # 列出文件，包括隐藏文件
cd /home/vicky/project      # 切换到绝对路径
mkdir -p logs/archive       # 连同缺失父目录一起创建
find . -maxdepth 2 -type f -name "*.py"
realpath ./main.py          # 显示规范化后的绝对路径
~~~

绝对路径从 / 开始，例如 /home/vicky/app/main.py；相对路径从当前目录开始，例如 ./main.py。脚本、服务配置和排障记录优先使用绝对路径，避免不同启动目录导致“找不到文件”。

查看内容与容量：

~~~bash
cat README.md                   # 适合小文件
less /var/log/syslog            # 分页查看，按 q 退出
head -n 20 file.txt             # 前 20 行
tail -n 50 app.log              # 后 50 行
tail -f app.log                 # 持续跟踪日志，Ctrl+C 停止
stat app.log                    # 大小、时间、所有者、权限
file data.bin                   # 判断可能的文件类型
du -sh .                        # 当前目录占用
df -h                           # 各挂载点可用空间
~~~

df 统计文件系统空间，du 统计目录内可见文件。两者差距很大时，可能是进程仍打开一个已删除文件，或数据处在别的挂载点。

软链接是另一个路径入口：

~~~bash
ln -s /opt/llm-api/releases/20260817 /opt/llm-api/current
ls -l /opt/llm-api/current
readlink -f /opt/llm-api/current
~~~

部署常用软链接切换版本。如果“代码已更新但服务仍运行旧版本”，要检查 systemd 的 WorkingDirectory、ExecStart 和软链接最终指向的位置。

## 3. Shell：安全执行与组合命令

Shell 是命令解释器，常见为 bash 或 zsh。可确认当前会话：

~~~bash
echo "$SHELL"
ps -p $$ -o pid,ppid,comm,args
~~~

### 3.1 命令、参数和退出码

命令通常是“程序 [选项] [参数]”。结束时会返回退出码，0 通常成功，非 0 通常失败：

~~~bash
curl --connect-timeout 3 https://example.com/
echo $?                        # 读取上一条命令的退出码
~~~

不要只看终端是否有输出。脚本必须检查退出码和预期结果。

### 3.2 stdin、stdout、stderr 与重定向

| 名称 | 文件描述符 | 用途 |
| --- | ---: | --- |
| 标准输入 stdin | 0 | 接收键盘或上游命令数据 |
| 标准输出 stdout | 1 | 输出正常结果 |
| 标准错误 stderr | 2 | 输出错误与诊断 |

~~~bash
python app.py > app.out                 # stdout 覆盖写入
python app.py >> app.out                # stdout 追加
python app.py 2> app.err                # stderr 单独写入
python app.py > app.log 2>&1            # 两类输出写到一个文件
curl -sS https://example.com/ | head    # 将 stdout 交给下游
~~~

符号 > 会覆盖旧文件。无限使用 >> 会耗尽磁盘；生产日志应由 journald、容器日志或 logrotate 管理。

### 3.3 管道、过滤与引号

管道 | 将左侧 stdout 传递给右侧 stdin：

~~~bash
ss -ltnp | grep ':8000'
journalctl -u llm-api --since "10 minutes ago" | grep -i error
~~~

ps aux | grep python 容易把 grep 自身也匹配进去，进程查询优先使用：

~~~bash
pgrep -af "uvicorn|gunicorn|python"
ps -C python -o pid,ppid,user,etime,%cpu,%mem,args
~~~

管道默认只以最后一条命令的退出码为准，可能掩盖上游失败。脚本里应启用 pipefail。

Shell 会解释空格、星号、美元符号等。路径和值应尽可能使用双引号：

~~~bash
cd "My Project"
grep -R -- "error code" logs/
printf '%s\n' "$MODEL_BASE_URL"
rm -- "./-unusual-file-name"
~~~

- "$VALUE" 保留空格，但仍展开变量。
- '$VALUE' 按字面量处理，不展开变量。
- 未加引号的值可能按空格拆分，或让 * 展开为文件列表。
- -- 表示选项结束，文件名以 - 开头时十分重要。

绝不可把用户输入、模型输出或外部文本直接拼入 Shell 命令。Python 启动子进程时传递参数列表，不要使用 shell=True：

~~~python
import subprocess

result = subprocess.run(
    ["curl", "--fail", "--silent", "--show-error", "https://example.com/health"],
    check=False,
    capture_output=True,
    text=True,
    timeout=5,
)
print(result.returncode)
~~~

### 3.4 变量、export 与脚本骨架

未 export 的变量只在当前 Shell 中存在；export 后子进程才会继承：

~~~bash
name="vicky"
export APP_ENV="development"
python app.py
APP_ENV="test" python app.py  # 仅对这一条命令有效
port=$(python -c "print(8000)")
echo "$port"
~~~

一个可靠的健康检查脚本：

~~~bash
#!/usr/bin/env bash
set -euo pipefail

service_url="http://127.0.0.1:8000/health"
if [ "$#" -gt 0 ]; then
  service_url="$1"
fi

if ! curl --fail --silent --show-error \
    --connect-timeout 3 \
    --max-time 5 \
    "$service_url" > /dev/null; then
  echo "健康检查失败：$service_url" >&2
  exit 1
fi

echo "健康检查通过：$service_url"
~~~

其中 set -e 让简单命令失败时退出；set -u 暴露未定义变量；pipefail 防止管道上游失败被掩盖；错误消息写入 stderr。执行前检查：

~~~bash
bash -n healthcheck.sh
shellcheck healthcheck.sh
~~~


## 4. 进程管理：Python 服务究竟在哪里

进程是正在运行的程序实例。它有 PID、父 PID（PPID）、运行用户、工作目录、环境变量、打开文件和网络连接。

~~~bash
ps aux | head
ps -p <PID> -o pid,ppid,user,stat,etime,%cpu,%mem,args
pstree -ap <PID>
pgrep -af "uvicorn|gunicorn|python"
~~~

PID 会复用。操作前应同时确认命令行、启动时间和运行用户，不能只根据一个 PID 判断目标。

### 4.1 前台、后台与作业控制

~~~bash
python app.py                 # 前台运行
python app.py &               # 当前 Shell 的后台作业
jobs -l                       # 当前 Shell 的作业列表
fg %1                         # 作业 1 拉回前台
bg %1                         # 已暂停作业在后台继续
~~~

后台作业不是可靠服务，终端断开后常会退出。开发临时任务可以使用：

~~~bash
nohup python app.py > app.log 2>&1 &
echo $!
~~~

正式服务应交由 systemd、容器平台或专用进程管理器，以获得重启、依赖、日志、资源限制和权限隔离。

### 4.2 信号：先优雅，再强制

| 信号 | 常见含义 | 建议 |
| --- | --- | --- |
| SIGTERM | 请求优雅退出 | 服务停止默认选择 |
| SIGINT | 中断，通常来自 Ctrl+C | 开发环境常见 |
| SIGHUP | 重载或终端挂断 | 含义由应用决定 |
| SIGKILL | 内核立即终止，无法捕获 | 最后手段 |
| SIGSTOP | 立即暂停，无法捕获 | 排障时谨慎使用 |

~~~bash
kill -TERM <PID>
sleep 3
ps -p <PID> -o pid,stat,args
kill -KILL <PID>              # 确认无法退出且理解后果时才使用
~~~

不要使用 kill -9 python 或 pkill -f python。这些范围模糊的命令可能杀掉 Notebook、部署工具或无关服务。优雅退出会给应用机会停止接收新请求、关闭连接、写完缓冲内容并释放资源。

### 4.3 状态、资源与 OOM

Z 表示僵尸进程：子进程已退出但父进程未回收。僵尸不能靠 kill 修复，应修复父进程的 wait 逻辑，或让父进程在正确监督下退出。D 表示不可中断 I/O 等待，常与磁盘、网络存储或驱动有关，SIGKILL 也未必立即有效。

~~~bash
ps -eo pid,ppid,user,stat,wchan:24,comm,args | grep -E ' Z| D'
top
free -h
uptime
lsof -p <PID>
ls -l /proc/<PID>/fd
cat /proc/<PID>/limits
~~~

LLM 服务变慢时，按证据检查：

1. CPU 是否长期满载？Python 的同步 CPU 密集操作会堵塞事件循环。
2. 可用内存是否持续下降？是否有内存泄漏或请求堆积？
3. 打开文件数是否持续增长？HTTP 响应、数据库连接或文件是否未关闭？
4. 网络连接是否大量处于 CLOSE_WAIT、TIME_WAIT 或 ESTAB？
5. 应用的错误率、请求延迟和队列长度是否同步恶化？

进程没有 Python traceback 却消失时检查 OOM：

~~~bash
journalctl -k --since "1 hour ago" | grep -i -E "out of memory|killed process"
journalctl -u llm-api --since "1 hour ago"
~~~

### 4.4 systemd 管理服务

~~~bash
systemctl status llm-api
sudo systemctl start llm-api
sudo systemctl stop llm-api
sudo systemctl restart llm-api
sudo systemctl enable llm-api
journalctl -u llm-api -f
~~~

示意 unit 文件如下。路径、模块、端口必须按真实项目调整：

~~~ini
# /etc/systemd/system/llm-api.service
[Unit]
Description=LLM API service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=llmapi
Group=llmapi
WorkingDirectory=/opt/llm-api
EnvironmentFile=/etc/llm-api/llm-api.env
ExecStart=/opt/llm-api/.venv/bin/python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
Restart=on-failure
RestartSec=3
NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
~~~

关键点：

- 应用使用专用非 root 用户，降低代码漏洞的影响范围。
- WorkingDirectory 决定相对路径，是“找不到文件”的高频根因。
- EnvironmentFile 与代码分离，但必须限制权限，且不能提交 Git。
- 127.0.0.1 只允许本机访问，适合交给反向代理；0.0.0.0 会监听所有网卡，应配套鉴权和防火墙。
- Restart=on-failure 只能恢复崩溃，不能替代业务错误处理。

修改 unit 后完成验证闭环：

~~~bash
sudo systemctl daemon-reload
sudo systemctl restart llm-api
systemctl status llm-api --no-pager
journalctl -u llm-api -n 100 --no-pager
~~~

## 5. 网络排障：从域名到 HTTP 的分层检查

IP 标识网络接口，端口标识主机上的服务。TCP 连通不代表 HTTP 路径、鉴权、请求体或业务逻辑正确。

### 5.1 本机地址和路由

~~~bash
ip addr show
ip route show
hostname -I
~~~

你要能回答：机器有哪些网卡与 IP？访问目标 IP 时走哪个网关？VPN、容器、多网卡会改变实际答案。

### 5.2 DNS

~~~bash
getent hosts api.example.com
dig api.example.com
nslookup api.example.com
resolvectl status
~~~

getent 接近程序使用系统名称服务时的实际结果；dig 更适合查看 DNS 细节。异常时检查 /etc/hosts、企业 DNS、VPN、容器 DNS 与代理。域名可能有多个 A/AAAA 记录，IPv6 不可用会导致偶发超时。临时 hosts 记录不要遗留。

### 5.3 监听、端口与连接

~~~bash
ss -ltnp
ss -ltnp '( sport = :8000 )'
sudo lsof -iTCP:8000 -sTCP:LISTEN
nc -vz -w 3 127.0.0.1 8000
nc -vz -w 3 <服务器IP> 8000
~~~

连接被拒绝通常说明目标可达，但端口没有监听或被主动拒绝；连接超时更常见于路由错误、防火墙丢包、IP 错误或服务阻塞。先区分这两类现象。

### 5.4 curl：HTTP 证据与耗时

~~~bash
curl -i http://127.0.0.1:8000/health
curl -v --connect-timeout 3 --max-time 10 https://api.example.com/health
curl -sS -o /dev/null -w "%{http_code} %{time_connect} %{time_starttransfer} %{time_total}\n" \
  https://api.example.com/health
~~~

-i 显示响应头；-v 显示连接和 TLS 细节，可能泄露 Authorization 头，不能直接公开；connect-timeout 限制建连时间；max-time 限制总时长；time_starttransfer 接近首字节时间，对流式 LLM 的首 token 时间特别有帮助。

| 状态 | 优先检查方向 |
| --- | --- |
| 200 / 204 | 成功，但仍检查业务结果与延迟 |
| 400 | 参数、JSON、编码 |
| 401 / 403 | 令牌、身份、授权策略 |
| 404 | 路由前缀、反向代理重写 |
| 429 | 配额、限流、退避、并发 |
| 500 | 用 trace id 查应用日志 |
| 502 / 503 / 504 | 网关、上游进程、超时、网络 |

### 5.5 代理、TLS 与标准 502 流程

~~~bash
env | grep -i proxy
curl --noproxy '*' http://127.0.0.1:8000/health
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null
date -Is
~~~

HTTP_PROXY、HTTPS_PROXY、NO_PROXY 会影响 curl 与 Python。localhost 和 127.0.0.1 通常要放入 NO_PROXY。TLS 失败不要长期用 curl -k 或 verify=False 绕过验证；应检查证书链、主机名、系统时间和企业 CA。

出现外部 502 时：

1. 客户端 curl -v 记录状态、响应头、时序。
2. 在代理机器检查域名解析、代理进程和错误日志。
3. 在同机请求 http://127.0.0.1:8000/health，绕过代理检查 Python 上游。
4. ss -ltnp 确认 8000 被预期进程监听。
5. 用 systemctl status 与 journalctl -u llm-api 查重启、导入失败、缺少配置或 OOM。
6. 上游正常后再核对代理 upstream、超时、Host 头与权限策略。
7. 记录每层结论；成功层可以排除，失败层进入下一轮。

## 6. 环境变量、虚拟环境与密钥

环境变量是父进程传给子进程的键值集合。子进程修改自身环境不会回写父 Shell：

~~~bash
export APP_ENV=development
python -c "import os; print(os.environ.get('APP_ENV'))"

python -c "import os; os.environ['APP_ENV']='changed'"
echo "$APP_ENV"
~~~

在终端 export 变量，不代表 systemd、Docker、IDE 或远程服务能看到。配置不生效时，先问：这个具体进程由谁启动，环境来自哪里？

### 6.1 Python 配置校验

区分必需配置和可选配置。必需配置缺失时应尽早、明确、但不泄密地失败：

~~~python
import os


def require_env(name: str) -> str:
    """读取必需环境变量，避免空配置进入运行时。

    Args:
        name: 环境变量名称。

    Returns:
        去除首尾空白后的配置值。

    Raises:
        RuntimeError: 变量不存在或为空时抛出。
    """
    value = os.getenv(name)
    if value is None or not value.strip():
        raise RuntimeError(f"缺少必需配置：{name}")
    return value.strip()


app_env = os.getenv("APP_ENV", "development")
model_base_url = os.getenv("MODEL_BASE_URL", "https://api.example.com")
api_key = require_env("MODEL_API_KEY")
~~~

环境变量都是字符串。布尔值和数字要严格验证，不能把任意非空字符串当作真：

~~~python
def read_bool(name: str, default: bool = False) -> bool:
    """读取布尔环境变量，只接受明确值。

    Args:
        name: 环境变量名称。
        default: 变量缺失时的默认值。

    Returns:
        解析得到的布尔值。

    Raises:
        ValueError: 值不是允许布尔文本时抛出。
    """
    raw_value = os.getenv(name)
    if raw_value is None:
        return default

    normalized = raw_value.strip().lower()
    if normalized in {"1", "true", "yes"}:
        return True
    if normalized in {"0", "false", "no"}:
        return False
    raise ValueError(f"{name} 必须是 true/false")
~~~

不要打印完整 api_key 验证加载是否成功。可以记录“已加载/缺失”，或用不回显密钥的受控健康检查验证。

### 6.2 .env 与虚拟环境

推荐约定：

~~~text
提交仓库：.env.example，只含变量名和无敏感示例
本地私密：.env，加入 .gitignore
生产环境：密钥服务、平台机密变量，或仅服务账号可读的 EnvironmentFile
~~~

虚拟环境隔离 Python 依赖，但不隔离系统权限和网络：

~~~bash
python3 -m venv .venv
source .venv/bin/activate
which python
python -m pip install -r requirements.txt
python -m pip check
deactivate
~~~

始终使用 python -m pip，避免 pip 指向另一个解释器。systemd 不应依赖 source activate；ExecStart 直接写 .venv/bin/python 的绝对路径。

### 6.3 密钥处理底线

- API Key、数据库密码、私钥和 token 不提交 Git。
- 不在 curl -v、printenv、日志、异常对象或截图中输出完整密钥。
- 不用命令行参数传递密钥，本机其他用户可能通过 ps 看到。
- 服务账户只读取它需要的密钥。
- 怀疑泄漏时立即撤销、轮换和审计，不是只删除日志。
- 环境变量是传递通道，不是完整的密钥治理；生产环境优先密钥管理服务。

## 7. 权限管理：为什么会 Permission denied

先确认身份：

~~~bash
whoami
id
groups
ls -l /etc/llm-api/llm-api.env
~~~

典型输出：

~~~text
-rw-r----- 1 llmapi llmapi 1200 Aug 17 10:00 llm-api.env
~~~

第一个字符是类型：- 表示普通文件，d 表示目录，l 表示软链接。后面三组是所有者、所属组和其他人的权限。r=读，w=写，x=执行或进入。

| 模式 | 常用场景 | 含义 |
| --- | --- | --- |
| 600 | 私钥、私密配置 | 仅所有者读写 |
| 640 | 服务配置 | 所有者读写，组只读 |
| 644 | 一般文本 | 所有者读写，其他只读 |
| 700 | 私密目录/脚本 | 仅所有者可访问 |
| 755 | 程序目录 | 所有者可写，其他可读/进入 |
| 750 | 服务目录 | 所有者全权，组可读/进入 |

目录的 x 并不是“执行文件”，而是可进入、可按名称访问内容。文件看似可读，但任何父目录缺少 x，进程仍无法读取：

~~~bash
namei -l /etc/llm-api/llm-api.env
ls -ld /etc /etc/llm-api
ls -l /etc/llm-api/llm-api.env
~~~

最小化修复示例：

~~~bash
chmod 600 /etc/llm-api/llm-api.env
sudo chown llmapi:llmapi /opt/llm-api
chmod 750 /opt/llm-api
~~~

修改前确认目标、当前权限、需要授权给的用户/组及最小权限。禁止把 chmod -R 777 当作修复，也不要使用 sudo chown -R <用户> /。用 root 跑应用只是掩盖权限错误，并放大漏洞影响。

sudo 是执行一条明确高权限命令的机制：

~~~bash
sudo -l
sudo systemctl restart llm-api
~~~

应用应使用专用非登录账号：

~~~bash
sudo useradd --system --create-home --shell /usr/sbin/nologin llmapi
~~~

该示例会改变系统账号，只能在测试机或已批准部署流程中使用。权限看起来正确仍被拒绝时，还要考虑 ACL、SELinux/AppArmor、挂载参数和容器安全策略：

~~~bash
getfacl /srv/shared
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
~~~

SSH 私钥不应上传到聊天、工单、仓库或截图。团队访问应使用个人密钥、短期凭证和可撤销账号，而不是共享 root 密码。

## 8. 综合练习：启动与排障一个 LLM 服务

启动前：

~~~bash
cd /home/vicky/llm-api
source .venv/bin/activate
python --version
python -m pip check
test -n "$MODEL_API_KEY" && echo "MODEL_API_KEY 已设置" || echo "MODEL_API_KEY 缺失"
~~~

不要打印密钥。启动服务后，另开终端确认：

~~~bash
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
ss -ltnp '( sport = :8000 )'
curl -i --connect-timeout 2 http://127.0.0.1:8000/health
~~~

若本机 curl 失败，先看 Python traceback；只有进程确实监听后，才继续排查代理、远程访问或 DNS。

如果“环境变量已设置，服务却说缺失”，按证据链检查：

~~~bash
systemctl show llm-api -p Environment -p EnvironmentFiles
systemctl cat llm-api
sudo ls -l /etc/llm-api/llm-api.env
sudo journalctl -u llm-api -n 100 --no-pager
~~~

根因常是：编辑了登录 Shell 的 .env 而非 systemd 的 EnvironmentFile；变量名不一致；文件不可读；服务未重启；服务用了不同虚拟环境。不要公开 systemctl show 的完整输出，其中可能带机密。

如果“外部访问 502，但本机健康检查正常”，重点检查反向代理与边界网络：

~~~bash
curl -i http://127.0.0.1:8000/health
sudo ss -ltnp '( sport = :80 or sport = :443 )'
sudo systemctl status nginx
sudo tail -n 100 /var/log/nginx/error.log
~~~

核对 upstream 是否等于 Python 的真实监听地址。localhost、127.0.0.1、容器名和 IPv6 不完全等价，应以真实解析与监听证据为准。

如果“外部模型 API 超时”，从近到远验证：

~~~bash
getent hosts api.example.com
curl -v --connect-timeout 3 --max-time 10 https://api.example.com/health
env | grep -i proxy
~~~

若由 systemd 启动，检查服务配置里的代理变量，而不只看当前终端。curl 成功而 Python 失败时，再检查 Python HTTP 客户端的超时、代理、CA 证书、连接池和请求体大小。

## 9. 常见误区

| 误区 | 问题 | 更好的做法 |
| --- | --- | --- |
| “端口在监听，服务没问题” | 只证明有进程监听 | curl 健康检查并看日志 |
| “curl 通，Python 一定通” | 代理、证书、超时、连接池可能不同 | 对比进程环境和客户端配置 |
| “sudo 能修复” | 掩盖权限设计并扩大攻击面 | 用运行用户和最小权限定位 |
| “kill -9 最有效” | 无清理机会，可能损坏状态 | 先 SIGTERM，再查阻塞 |
| “.env 天然安全” | 仍可能提交、备份或泄漏 | Git 忽略、权限、轮换和密钥服务 |
| “ping 不通就是网络不通” | ICMP 可被禁，TCP/HTTPS 仍可用 | 用 nc/curl 验证实际协议 |
| “localhost 等于 127.0.0.1” | localhost 可解析为 IPv6 ::1 | getent 验证并明确监听 IP |
| “改 unit 就生效” | systemd 需要 reload 和重启 | reload、restart、status、journal 闭环 |

## 10. 六周学习计划与自检

### 第 1 周：Linux 文件与日志

每天使用 pwd、ls、find、less、tail，找出一个 Python 项目的入口、日志、依赖和虚拟环境。验收：能解释绝对路径、软链接和目录 x 权限。

### 第 2 周：Shell

练 stdout/stderr、管道、退出码；写健康检查脚本；使用 bash -n 和 shellcheck。验收：脚本失败返回非 0，错误输出 stderr，不依赖当前目录。

### 第 3 周：进程与服务

启动 Python HTTP 服务，观察 ps、pgrep、ss；用 SIGTERM 停止；查看 CPU、内存、文件描述符和日志；在测试机写 systemd unit。验收：能说出服务 PID、PPID、运行用户、工作目录、端口和日志位置。

### 第 4 周：网络

使用 getent、dig、ip route、ss、nc、curl；故意写错本地端口或路径，比较拒绝、404、502；用 curl 时间指标记录连接、首字节和总耗时。验收：遇到访问失败时能写出五层排查顺序。

### 第 5 周：配置与密钥

实现必需变量、默认值、布尔/数字校验；建立 .env.example 并忽略 .env；比较 Shell、IDE、systemd 中变量可见性。验收：缺失配置给出明确且不泄密的错误。

### 第 6 周：权限与综合演练

在实验机模拟 Permission denied，用 chmod、chown、namei 定位；确认服务不以 root 运行；完成一份 502 或模型 API 超时故障报告。报告应包含现象、时间线、命令证据、根因、修复、验证和防复发措施。

宣布“修好”前至少确认：

- 原始失败请求是否成功，退出码或 HTTP 状态是否符合预期？
- 正常路径与错误路径是否都验证？
- 服务是否由预期的非 root 用户运行？
- 端口是否只暴露给需要的网络范围？
- 日志是否足以排障且没有泄漏密钥？
- 配置和密钥是否不在 Git、命令行、公开日志中？
- 机器重启后服务能否恢复？
- 是否记录了根因和证据，而不只是“重启后好了”？

这六类能力并非 Python 的附属知识。它们让你能把大模型应用当作可观察、可验证、可维护的服务，为继续学习 FastAPI、Docker、RAG、模型推理和可观测性打下可靠基础。
