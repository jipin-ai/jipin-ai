<div align="center">

# 54 集

**Orchestrating six AI agents that review, reject, and sign off on each other's work.**

</div>

<br>

多智能体系统实战连载。至少 54 集，每周一集。这是第 1 集。

<br>

```
+-----------+
| Architect | <- local workstation, SSH reverse tunnel
+-----------+
     :  governance only (not in pipeline)
     v
+-----------+
|   BA-01   |
+-----------+
     |
     | [G1] five-questions gate
     v
+-----------+
|   ORC-01  |
+-----------+
     |
     | dispatch
     v
+-----------+
|   DEV-01  |
+-----------+
     |
     | build done
     v
+-----------+
|   QA-01   |
+-----------+
     |
     | [G2] dual-confirm gate
     v
+-----------+
|  DEMO-01  |
+-----------+

comm: A2A / JSON-RPC   state: SQLite state machine
```

五台云端 ECS 各跑一个角色；架构师在本地，只管治理不进流水线。G1 = 五问门禁 · G2 = 双确认门禁 · 通信 = A2A (JSON-RPC) · 状态 = SQLite 状态机。

<br>

---

### 主线项目

**[agent-pipeline-engine](https://github.com/jipin-ai/agent-pipeline-engine)** — 多 Agent 管道引擎：YAML 配置驱动 + SQLite 状态机门禁 + cron 调度，零外部依赖纯标准库。本连载全部代码沉淀于此。

[![Readme Card](https://github-readme-stats.vercel.app/api/pin/?username=jipin-ai&repo=agent-pipeline-engine)](https://github.com/jipin-ai/agent-pipeline-engine)

最近：给 [Hermes Agent](https://github.com/NousResearch/hermes-agent)（NousResearch）提交了中文文档 PR。

<br>

---

连载：[第 1 集](#第-1-集已完结) · **第 2 集（当前）**

<br>

### 第 2 集 · 一张表终结 95 小时阻塞

先交代背景。我们的管道有条规矩：任务进编码之前，DEV 和 QA 都得先确认拆解方案，两边点头才放行。确认靠 Agent 之间发消息。至于对方到底确认没有，说实话当时查不到，记录散落在各自的会话里，没有一台机器能直接回答这个问题。

然后出事了。

一次拆解方案同时发给 DEV 和 QA。DEV 当天就确认了。QA 没动静。第一天没动静，第二天也没有。没有任何报错，没有告警，管道看起来一切正常。第四天有人去问，才发现 QA 的模型配额早就用完了，它根本没看到那条消息。从发出到被发现，95 个小时。

我后来复盘，问题不复杂，就是三个小缺口凑到了一起。消息发出去没有回执，发的人不知道对方收没收到；"确认了吗"的答案只存在于聊天记录里，没有实体可查；QA 挂了这件事，只有 QA 自己的日志知道。单看每条都不致命，叠在一起就是四天盲区。

修法土得掉渣：一张 SQLite 表，三个脚本，一共 398 行。

表里每个任务一行，dev_confirmed 和 qa_confirmed 两列就是全部信息。一个脚本挂 cron，每 5 分钟主动问一遍两边"确认了吗"，不再干等推送。还有一个脚本专门回答"卡在哪"：跑一下，输出 `BLOCKED dev=1 qa=0`，exit code 直接喂给门禁。

没有消息队列，没有工作流引擎。轮询听起来笨，但在这个场景里，笨就是可靠——推送会丢，轮询不会。

跑起来的样子（真实日志）：

```
10:47:39 询问 dev ...
10:47:56 dev → 已确认 ✅        （真实往返，17 秒）
10:49:56 qa → 无有效回复        （配额耗尽那天，容错记录，不崩）

gate_judge.py fengyun-v1.2-five-questions
→ BLOCKED dev=1 qa=0 （exit 1）
```

95 个小时里没人说得清卡在哪。现在一行输出说清了。

**结局**：第二天早上八点，QA 的配额恢复。又过了一天，我翻账本做验收的时候看到这一行：

```
gate_judge.py fengyun-v1.2-five-questions
→ PASS dev=1 qa=1 （exit 0）
```

95 小时的阻塞，最后是被一张 20KB 的表终结的。

<br>

<details>
<summary><b>第 1 集：全网红灯 · 失能的 Agent · 十秒死锁 · 总是拒绝</b>（已完结，点击展开）</summary>

<br>

五台云主机加一台本地工作站，各跑一个 Agent。原来的通信靠消息队列，我们把它废了，改用 A2A 协议直连。前后折腾两周。下面这四个晚上，就是通车之前发生的事。

**全网红灯。** 六台机器部署完，每台的 Agent Card 都能拉取，但谁呼谁都是 connection refused。我查了四个小时防火墙和路由，最后发现根本不是网络的事：客户端拿到对端地址后，不是直接发消息，而是先拉对方的 Agent Card，然后往**卡片里写的地址**发。卡片默认写的是 127.0.0.1。也就是说，每个呼叫方都在给自己打电话。修法很朴素，每台机器配一行 A2A_PUBLIC_URL，把真实地址写进卡片。

**失能的 Agent。** 通道通了，消息能进了，但对端只会聊天不会干活。让它读个配置，它说没这个工具。查到最后是一行配置：platform_toolsets.a2a 只写了 ['a2a']，入站会话被裁到只剩 5 个通话工具，文件和终端全没了。更麻烦的是，运维那边远程探测时，这个失能的 Agent 连"读一下自己的配置"都做不到，于是被记成"配置未生效"，背了半天冤案。把工具集配全，它当场复活。

**十秒死锁。** 这次是全军覆没，所有终端命令全部超时被拒。根因藏得最深：配置里写了 approvals.mode: auto，但这个枚举值根本不存在，合法的只有 manual/smart/off。框架没报错，悄悄回落到 manual。于是每条命令都弹审批，无人值守，10 秒不答就拒绝。一个拼写级别的错误，瘫痪了整条管道的执行力。

**总是拒绝。** 对端说"通道测试总是被拒"，我翻遍本端日志，一条拒绝记录都没有。真相是网关用的单线程 HTTP 服务器：一个 6 分钟的任务在处理期间，健康检查、通道测试、下一条任务，全部堵在连接层排队。对端看到的"拒绝"，其实是排队排到死。改成 ThreadingHTTPServer，一行，世界清净。

</details>

**下一集**：Agent 说"完成了"，你信吗？——有一次它发来的 receipt 写得漂漂亮亮，复测一跑，交付物是 46 行骨架。

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
