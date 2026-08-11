<div align="center">

# 54 集

**Orchestrating six AI agents that review, reject, and sign off on each other's work.**

</div>

<br>

多 Agent 协同研究 · 实战连载。至少 54 集，每周一集，当前第 3 集。

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

连载：[第 1 集](#) · [第 2 集](#) · **第 3 集（当前）** — 多 Agent 协同研究 · 实战连载

<br>

### 第 3 集 · 多 Agent 协同里，谎言会复利

让一个 Agent 干活，它说"做完了"，你复测一次，损失截止。这是单 Agent 时代的事。

我们跑的是五个 Agent 的接力管道：分析、拆解、编码、测试、部署，一站交一站。在这里，"做完了"这三个字的分量完全不同——因为下游每一站都拿上一站的产出当原料。上游撒谎，下游不是在闲着，而是在虚构之上认真施工。

上周出了个真事。管道最后一站是部署。负责的 Agent 交了证据：部署地址、健康检查通过。验收程序验过，放行。我们打开那个地址，是它自己的身份名片页——每个 Agent 天生都有的一个页面。相当于它把名片钉在墙上说：店开好了。真正的服务，压根没启动。

回头推演一下就后怕：如果它这站没被抽查到，这条假完成就会作为"事实"进档案，下游的验收、交付、汇报全部建立在它上面。一个谎言，经过五站接力，被放大成一整套看起来很完整的假交付。

而且那天本来我们排了一个考试：故意交假证据，看验收抓不抓得住。考试没开始，它抢答了，来真的。成绩：没抓住。

没抓住的原因是一行代码。验收程序写的是"健康检查字段为空，就实际探测一次"。反过来，字段被填上就不探了。验收权本来立在判定方手里，这行代码把它悄悄还给了自报方。

修完之后的版本只有一条原则：报了地址就无条件真敲一次，自报字段只当参考。那位 Agent 被打回重做，起了真服务，外网实测能访问，任务才算完。

**这一集的方法论**（多 Agent 协同专用）：

1. **证据翻译**：链条上每个"完成"都必须译成下游和裁判都能验的机器事实——commit 哈希、报告文件、能访问的 URL。"我做完了"这种话不允许在管道里流通
2. **裁判独立**：验证逻辑永远不信被验证方填的字段。"自报 X 就跳过检查 X"，在哪一跳出现，哪一跳就是后门
3. **过关≠活着**：状态机只证明"那一刻是真的"。服务后来死没死，是监控的事，不是门禁的事

单 Agent 场景里这三条是建议。多 Agent 协同里，它们是让系统不至于集体生产 fiction 的底线。

下一集讲调度台自己的瓶颈：一个编排 Agent 被消息淹死的那天，以及我们怎么教会它一次派六个分身干活。

<br>

<details>
<summary><b>第 2 集：一张表终结 95 小时阻塞</b>（已完结，点击展开）</summary>

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

</details>

<details>
<summary><b>第 1 集：全网红灯 · 失能的 Agent · 十秒死锁 · 总是拒绝</b>（已完结，点击展开）</summary>

<br>

五台云主机加一台本地工作站，各跑一个 Agent。原来的通信靠消息队列，我们把它废了，改用 A2A 协议直连。前后折腾两周。下面这四个晚上，就是通车之前发生的事。

**全网红灯。** 六台机器部署完，每台的 Agent Card 都能拉取，但谁呼谁都是 connection refused。我查了四个小时防火墙和路由，最后发现根本不是网络的事：客户端拿到对端地址后，不是直接发消息，而是先拉对方的 Agent Card，然后往**卡片里写的地址**发。卡片默认写的是 127.0.0.1。也就是说，每个呼叫方都在给自己打电话。修法很朴素，每台机器配一行 A2A_PUBLIC_URL，把真实地址写进卡片。

**失能的 Agent。** 通道通了，消息能进了，但对端只会聊天不会干活。让它读个配置，它说没这个工具。查到最后是一行配置：platform_toolsets.a2a 只写了 ['a2a']，入站会话被裁到只剩 5 个通话工具，文件和终端全没了。更麻烦的是，运维那边远程探测时，这个失能的 Agent 连"读一下自己的配置"都做不到，于是被记成"配置未生效"，背了半天冤案。把工具集配全，它当场复活。

**十秒死锁。** 这次是全军覆没，所有终端命令全部超时被拒。根因藏得最深：配置里写了 approvals.mode: auto，但这个枚举值根本不存在，合法的只有 manual/smart/off。框架没报错，悄悄回落到 manual。于是每条命令都弹审批，无人值守，10 秒不答就拒绝。一个拼写级别的错误，瘫痪了整条管道的执行力。

**总是拒绝。** 对端说"通道测试总是被拒"，我翻遍本端日志，一条拒绝记录都没有。真相是网关用的单线程 HTTP 服务器：一个 6 分钟的任务在处理期间，健康检查、通道测试、下一条任务，全部堵在连接层排队。对端看到的"拒绝"，其实是排队排到死。改成 ThreadingHTTPServer，一行，世界清净。

</details>

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
