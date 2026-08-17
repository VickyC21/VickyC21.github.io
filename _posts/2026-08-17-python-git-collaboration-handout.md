---
layout: post
title: "Python基础：Git 从入门到团队协作实战讲义"
date: 2026-08-17 11:00:00 +0800
categories: [Python基础]
tags: [Python, Git, GitHub, 分支协作, 代码审查, 冲突处理]
---

# Python基础：Git 从入门到团队协作实战讲义

大模型开发不只是把 Python 代码写出来。提示词、评测样本、服务配置和部署资产都需要被多人安全地修改，并能在效果回退或线上故障后追查原因。Git 就是保存、协作、审查和追溯这些变化的基础工具。

本文面向第一次系统学习 Git 的 Python 开发者。目标不是背命令，而是能回答：我的改动在哪里？怎样安全交给团队？出了问题怎样定位和撤回？

## 1. Git 的四个位置

| 位置 | 含义 | 常见操作 |
| --- | --- | --- |
| 工作区 | 磁盘上正在编辑的文件 | 编辑、测试、git diff |
| 暂存区 | 下一次提交准备包含的文件清单 | git add、git restore --staged |
| 本地仓库 | 本机已确认保存的提交历史 | git commit、git log |
| 远程仓库 | GitHub/GitLab 上团队共享的历史 | git fetch、git pull、git push |

    编辑文件 -> 工作区
     git add 选择本次要提交的内容 -> 暂存区
     git commit 创建可追溯快照 -> 本地仓库
     git push 与团队共享 -> 远程仓库

git add 不会上传文件，git commit 也不会推送。前者选择快照内容，后者只在本机保存快照。

提交（commit）是项目的不可变快照，带作者、时间、说明和唯一 ID。分支（branch）是指向某个提交的可移动名称。HEAD 表示你当前检出的分支或提交。远程跟踪分支如 origin/main 是本地记录的远程状态，不是远程实时状态。

    A -- B -- C  main
          \
           D -- E  feature/streaming-timeout

## 2. 首次配置与每日检查

确认 Git 已安装，并设置真实、稳定的提交身份：

    git --version
    git config --global user.name "你的名字"
    git config --global user.email "你的邮箱"
    git config --global init.defaultBranch main

已有项目应克隆：

    git clone https://github.com/组织名/项目名.git
    cd 项目名
    git remote -v

本地新项目则执行 git init。无论何时不确定自己在哪个分支、改了什么或是否落后远程，都先运行：

    git status

## 3. 每日最小工作闭环

假设为 FastAPI 的 LLM 客户端增加请求超时：

    # 编辑 app/llm_client.py 和 tests/test_llm_client.py
    git status
    git diff
    git add app/llm_client.py tests/test_llm_client.py
    git diff --staged
    pytest tests/test_llm_client.py -q
    git commit -m "fix(llm): 为模型请求增加超时处理"

git diff 展示未暂存内容；git diff --staged 展示真正将被提交的快照。不要习惯性使用 git add .，否则可能把 .env、日志、缓存或本地评测产物一并纳入。

好提交只解决一个可描述、可测试、可回滚的问题。推荐提交标题：

    <type>(可选作用域): 简短说明

| 类型 | 用途 | 示例 |
| --- | --- | --- |
| feat | 新功能 | feat(api): 支持流式响应 |
| fix | 修复缺陷 | fix(rag): 修复空检索结果异常 |
| test | 测试改动 | test: 覆盖超时重试边界 |
| docs | 文档改动 | docs: 补充启动说明 |
| refactor | 行为不变的重构 | refactor: 拆分提示词构造函数 |

重要改动应使用多行提交信息，说明背景、主要改动和验证命令。避免 update、修改代码、fix bug 这类无法解释意图的标题。

## 4. 撤销前先确认改动的位置

| 目的 | 命令 | 风险 |
| --- | --- | --- |
| 丢弃未暂存修改 | git restore 文件名 | 工作区内容会丢失 |
| 取消暂存但保留修改 | git restore --staged 文件名 | 仅改暂存区 |
| 补充最近一次提交 | git commit --amend | 改写最新本地提交 |
| 临时保存未完成工作 | git stash push -m "WIP: 排查超时" | 恢复时可能冲突 |
| 恢复暂存工作 | git stash pop | 可能冲突 |

先执行 git status 和 git diff 再做任何可能丢内容的操作。已推送且可能被别人拉取的提交，不要用 reset 改写公共历史，应使用后文的 revert。

## 5. .gitignore 与密钥安全

.gitignore 只忽略尚未跟踪的文件，不会自动删除已经提交的敏感内容。

    __pycache__/
    *.py[cod]
    .pytest_cache/
    .venv/
    .env
    .env.*
    !.env.example
    logs/
    htmlcov/

绝不能提交 API Key、数据库密码、生产配置或真实用户数据。若密钥已经提交，即使后来删除也可能留在 Git 历史中：立即在对应平台吊销并重新生成密钥，通知维护者并按团队安全流程处理。

## 6. 分支协作和远程同步

主分支 main 应尽量保持可测试、可部署。每个独立任务从最新主分支建立分支：

    git switch main
    git pull --ff-only origin main
    git switch -c fix/llm-request-timeout

推荐名称：feature/streaming-timeout、fix/retry-on-429、docs/git-handout。小分支、小提交和单一主题 PR 会减少冲突，也让审查和回滚更可靠。

    # 编码、验证、提交后
    git push -u origin fix/llm-request-timeout

-u 建立本地和远程分支的追踪关系。之后在 GitHub/GitLab 创建 PR，目标为 main，来源为你的功能分支。

| 命令 | 作用 | 是否修改工作区 |
| --- | --- | --- |
| git fetch origin | 下载远程提交信息 | 否 |
| git pull | fetch 后整合远程改动 | 是，可能冲突 |
| git push | 上传本地提交 | 改变远程仓库 |

不确定远程有什么变化时，先安全查看：

    git fetch origin
    git log --oneline HEAD..origin/main
    git diff HEAD..origin/main

## 7. merge、rebase 与冲突

主分支在你开发期间会前进。整合它有两种常见方式：

    # merge：不改写已有提交，适合共享分支和新人
    git merge origin/main

    # rebase：让个人提交排在最新主分支之后
    git rebase origin/main

rebase 会生成新的提交 ID，即改写历史。不要对其他人使用的共享分支 rebase，更不要强推主分支。个人分支 rebase 后如需推送，应谨慎使用 git push --force-with-lease；它会在远程出现未知提交时拒绝覆盖，比 --force 更安全。

冲突不是错误，而是 Git 无法替你做业务判断。例如：

    <<<<<<< HEAD
    DEFAULT_TIMEOUT_SECONDS = 30
    =======
    DEFAULT_TIMEOUT_SECONDS = 15
    >>>>>>> origin/main

从第一个标记到分隔线是当前分支内容；分隔线后是另一侧内容。解决步骤：

1. 运行 git status，确认冲突文件和当前操作是 merge 还是 rebase。
2. 阅读上下文，理解两侧为什么修改，不要机械保留其中一边。
3. 编辑成正确的最终代码，删除全部冲突标记。
4. 运行格式化、类型检查和受影响测试。
5. git add 文件名；merge 后 git commit，rebase 后 git rebase --continue。

不想继续时使用 git merge --abort 或 git rebase --abort 回到开始前状态。

LLM 服务中的常见冲突是：你提供可传入的 timeout 参数，同事让默认值来自环境配置。最终通常应同时保留两种需求：未传 timeout 时读配置，显式传入时覆盖配置，并分别写测试。

## 8. 代码审查：PR 是质量门

PR 不是要求别人点击同意的通知，而是团队在合并前审查风险的地方。作者应确保：

    [ ] PR 只包含一个主题，没有无关重构或格式化
    [ ] git diff --staged 和 PR diff 没有密钥、日志和临时文件
    [ ] 覆盖正常、空值、非法输入与关键边界
    [ ] PR 描述写清背景、改动、验证命令和已知限制

审查顺序应是：需求行为、数据与权限安全、异常和并发边界、兼容性、测试缺口，最后才是命名和风格。好的评论应指向可验证事实：

> 当上游返回非 JSON 响应时，response.json() 会抛异常而绕过当前错误映射。请补充该场景测试，并统一返回 UpstreamServiceError。

收到意见后逐项确认、修改、测试并回复处理结果；不同意时说明技术依据和替代方案。

## 9. 可追溯提交与安全回滚

查看历史和某个文件的变化：

    git log --oneline --graph --decorate --all
    git log -- app/streaming.py
    git show a1b2c3d
    git blame -L 42,55 app/streaming.py

blame 用于找出代码引入时的上下文、PR 和测试，不是追责工具。定位到提交后应阅读完整 diff 和提交说明。

已共享提交应通过反向提交撤回：

    git revert a1b2c3d
    git push

revert 保留公共历史，并记录撤回了什么。误操作导致本地提交看似消失时：

    git reflog
    git show HEAD@{3}
    git switch -c rescue-work HEAD@{3}

已知旧版本正常、新版本异常时，用二分定位回归：

    git bisect start
    git bisect bad
    git bisect good v1.4.0
    # 在中间版本测试后标记 good 或 bad
    git bisect reset

若测试稳定，可用 git bisect run pytest tests/test_streaming.py -q 自动二分。

## 10. 两周练习路线

第 1-2 天：创建 hello-git，练习 status、diff、add、diff --staged、commit 和 restore --staged。

第 3-5 天：创建 GitHub 练习仓库，从 main 建 feature/add-greeting，提交、推送并创建 PR。

第 6-8 天：在两个分支故意修改同一行，练习手动冲突解决、merge --abort、rebase --abort 和 reflog 救援分支。

第 9-14 天：为 FastAPI 或 CLI 小项目创建功能 PR，写清验证与限制；对人为引入的 bug 添加回归测试，再用 bisect 找到问题提交；最后对旧提交执行 revert。

## 11. 给 LLM 开发者的额外建议

- 版本化可公开的 Prompt 模板、评测规则和变更原因，不提交真实对话或敏感上下文。
- 记录评测样本来源、脱敏规则和版本，避免训练与评测数据混淆。
- 把模型名称、超时、采样参数与生产密钥分离；密钥通过环境变量或密钥管理系统注入。
- 改 Prompt、检索策略或模型路由时，在 PR 中记录测试集版本、指标和运行方式。
- 一次 PR 只改变一个主要变量，才能判断效果变化由什么造成。

Git 熟练的标志不是记住所有子命令，而是形成稳定习惯：先用 git status 认识现状，用 git diff 确认内容，用小提交记录完整意图，用分支和 PR 接受审查，发生问题后先定位，再用可追溯的方式修复。
