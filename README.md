<div align="center">

# 54 集

**Orchestrating six AI agents that review, reject, and sign off on each other's work.**

</div>

<br>

多智能体系统实战连载。至少 54 集，每周一集。这是第 1 集。

<br>

```
                      +-----------+
                      | Architect |      <- 本地工作站，SSH 反向隧道接入
                      +-----+-----+
                            :  治理 / 元管理（不入管道，不被审查）
                            :
   G1                       v                                       G2
   |                  +-----------+                                 |
 +-------+    EARS    |           |   dispatch   +-------+  build   +-------+  verify   +--------+
 | BA-01 | ---------> |   ORC-01  | -----------> | DEV-01| -------> | QA-01 | --------> | DEMO-01|
 +-------+            +-----------+              +-------+          +-------+           +--------+
```

五台云端 ECS 各跑一个角色；架构师在本地，只管治理不进流水线。G1 = 五问门禁 · G2 = 双确认门禁 · 通信 = A2A (JSON-RPC) · 状态 = SQLite 状态机。

<br>

---

### 主线项目

**agent-pipeline-engine** — 多 Agent 管道引擎：YAML 配置驱动 + SQLite 状态机门禁 + cron 调度，零外部依赖纯标准库。本连载全部代码沉淀于此。

最近：给 [Hermes Agent](https://github.com/NousResearch/hermes-agent)（NousResearch）提交了中文文档 PR。

<br>

---

### 第 1 集 · 六个 Agent 如何学会互相打电话

五台云主机 + 一台本地工作站，废弃消息队列，改用 A2A 协议直连。两周建成，全网贯通——以下是建成之前的四个夜晚。

<details>
<summary><b>四夜全文：全网红灯 · 失能的 Agent · 十秒死锁 · 总是拒绝</b>（点击展开）</summary>

<br>

**第一夜 · 全网红灯。** 六台机器部署完毕，每台卡片都能拉取，但任何呼叫都是 connection refused。查了四个小时防火墙，最后发现：客户端不是往配置的地址发消息——它先拉对端 Agent Card，然后往**卡片里写的地址**发。而卡片默认写 `127.0.0.1`，每个呼叫方都在给自己发消息。修法：每台配一行 `A2A_PUBLIC_URL` 公告真实地址。

**第二夜 · 失能的 Agent。** 通道通了，消息能进了，但对端只会聊天不会干活。根因是 `platform_toolsets.a2a: ['a2a']`：入站会话被裁到只剩 5 个通话工具，文件和终端全没了。最诡异的副作用：运维远程探测时，失能的 Agent 连"读自己配置自证"都做不到，于是被误报"配置未生效"——冤案。配全工具集，立即复活。

**第三夜 · 十秒死锁。** 所有终端命令超时被拒。根因：`approvals.mode: auto` 这个枚举值**根本不存在**（合法值仅 manual/smart/off），框架静默回落 manual——每条命令弹审批，无人值守，10 秒 fail-closed。一个拼写级错误，瘫痪整条管道的执行力。

**第四夜 · 总是拒绝。** 对端报"通道测试总是被拒"，本端日志却一条拒绝记录都没有。真相：单线程 HTTP 服务器，一个 6 分钟任务处理期间，健康检查和后续任务全部堵在连接层。对端看到的"拒绝"其实是"排队到死"。一行改成 ThreadingHTTPServer，世界清净。

</details>

**下一集**：同步轮询状态机 —— 一次 95 小时的流程阻塞，如何被一张 SQLite 表终结。

<br>

---

### 信条

**配置 > 硬编码** · **状态机 > 工作流引擎** · **cron > 常驻进程** · **零依赖 > 全家桶**

<br>

<p>
  <img src="https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white">
  <img src="https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white">
  <img src="https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black">
  <img src="https://img.shields.io/badge/Git-F05032?logo=git&logoColor=white">
  <img src="https://img.shields.io/badge/Markdown-000?logo=markdown&logoColor=white">
</p>

---

📮 shihetongtai@gmail.com
