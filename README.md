<div align="center">

# 54 集

**Orchestrating six AI agents that review, reject, and sign off on each other's work.**

</div>

<br>

一部多智能体系统的实战连载。至少 54 集，每周更新。这是第 1 集。

<br>

```
                        五问门禁                  双确认门禁
                          │                        │
 ┌────────┐    EARS    ┌──▼──┐   dispatch   ┌─────▼─┐  build  ┌────┐  verify  ┌──────┐
 │ BA-01  │ ─────────► │ ORC │ ───────────► │ DEV   │ ──────► │ QA │ ───────► │ DEMO │
 └────────┘            └─────┘              └───────┘         └────┘          └──────┘
                          │
            SQLite 状态机 · A2A/JSON-RPC · 六节点 Gate 互审 · 无人体中枢
```

<br>

---

### 第 1 集 · 六个 Agent 如何学会互相打电话

**设定**：五台云主机 + 一台本地工作站，各跑一个 Hermes Agent，角色分别是需求、编排、编码、测试、部署。目标：废弃消息队列，让六个 Agent 用 A2A 协议（JSON-RPC over HTTP）直接对话。两周建成，全网贯通。以下是建成之前的四个夜晚。

**第一夜 · 全网红灯。** 六台机器部署完毕，每一台的 Agent Card 都能正常拉取，但任何呼叫都是 connection refused。查了四个小时防火墙和路由，最后发现：客户端不是往配置的地址发消息——它先拉对端的 Agent Card，然后往**卡片里写的地址**发。而卡片默认写的是 `127.0.0.1`。每个呼叫方都在给自己发消息。修法是每台机器配一行 `A2A_PUBLIC_URL`，公告自己的真实地址。

**第二夜 · 失能的 Agent。** 通道通了，消息能进了，但对端 Agent 只会聊天不会干活——让它读配置，它说"我没有这个工具"。根因是一行看似无害的配置 `platform_toolsets.a2a: ['a2a']`：A2A 入站会话被裁到只剩 5 个通话工具，文件和终端全没了。最诡异的副作用是：运维方远程探测时，失能的 Agent 连"读一下自己配置文件自证"都做不到，于是报告"该节点配置未生效"——冤案。配全工具集，立即复活。

**第三夜 · 十秒死锁。** 又一个全军覆没：所有终端命令全部超时被拒。这次的根因藏得更深——配置里写了 `approvals.mode: auto`，而这个枚举值**根本不存在**（合法值只有 manual/smart/off）。框架静默回落到 manual：每条命令弹审批，无人值守，10 秒后 fail-closed。一个拼写级别的错误，瘫痪整条管道的执行力。

**第四夜 · 总是拒绝。** 最后一晚，对端报"通道测试总是被拒"，但本端日志里一条拒绝记录都没有。真相是网关用的单线程 HTTP 服务器：一个 6 分钟的真实任务在处理期间，健康检查、通道测试、下一个任务**全部堵在连接层**。对端看到的"拒绝"其实是"排队到死"。一行改成 ThreadingHTTPServer，世界清净了。

**下一集**：同步轮询状态机 —— 一次 95 小时的流程阻塞，如何被一张 SQLite 表终结。

---

### 连载主线项目

**[agent-pipeline-engine](https://github.com/jipin-ai/agent-pipeline-engine)** — 多 Agent 管道引擎：YAML 配置驱动 + SQLite 状态机门禁 + cron 调度。零外部依赖，纯标准库。本连载的全部代码沉淀于此。

最近：给 [Hermes Agent](https://github.com/NousResearch/hermes-agent)（NousResearch）提交了中文文档 PR。

---

### 信条

**配置 > 硬编码** · **状态机 > 工作流引擎** · **cron > 常驻进程** · **零依赖 > 全家桶**

---

<p>
  <img src="https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white">
  <img src="https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white">
  <img src="https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black">
  <img src="https://img.shields.io/badge/Git-F05032?logo=git&logoColor=white">
  <img src="https://img.shields.io/badge/Markdown-000?logo=markdown&logoColor=white">
</p>

📮 shihetongtai@gmail.com
