
---
### Agent 和普通 Chatbot / RAG 有什么区别？

- Chatbot 核心是<font color="#b7dde8">「把话说圆，把用户的需求回答好」</font>；
	
- RAG 是<font color="#b7dde8">「先检索再答」</font>；
	 
- Agent 是<font color="#b7dde8">「为了完成任务，在环境里闭环行动」</font>。

```
感知(用户意图+状态)

→ 决策(是否调工具/走哪条边) 
→ 行动(写文件/shell/CLI) 
→ 观察(tool result / PTY 输出) 
→ 再决策，直到满足终止条件（不再调工具、到 end 节点、或挂起等人）。
```

#### 项目延伸

1. <mark style="background:#d3f8b6">MCP ReAct</mark>：`run_mcp_react_loop` —— 模型出 `tool_calls`，执行 builtin / MCP 工具，结果回灌，默认最多约 24 步，可因风险闸门中断。
	
2. <mark style="background:#d3f8b6">CLI PTY turn</mark>：长驻 shell 里跑 `claude -p` / `codex` 等，只需要拼接命令，用 marker 切 turn 边界，用 `--resume` 延续会话即可，调用的是Claude、CodeX的工具 —— 环境是真实终端，不是纯文本生成。
	
3. <mark style="background:#d3f8b6">工作流节点</mark>：画布 JSON 编译成 LangGraph；每个 `agent_task` 节点执行完写入状态，经条件边 / `human_gate` 决定下一跳 —— 有状态、有终止（`passed`/`failed`/`awaiting_human`）。

> Clutch 里 MCP ReAct 循环、CLI PTY turn、工作流节点，都是「有工具、有状态、有终止条件」的 Agent 运行时。



---

## ReAct / Plan-and-Execute

### 什么是 ReAct？优缺点？

<mark style="background:#d3f8b6">Reason + Act</mark>：一轮里模型可以「想」+ 发 tool call；运行时执行后把 Observation 写回 messages，再叫模型 —— 交替直到不再调工具。

优点：灵活、适合探索型任务。  
缺点：步数不可控、易循环、成本高、工具选错。

#### 项目延伸

- 组装 messages + tools schema（builtin `clutch-tools` + 用户配置的 MCP server）
	
- 调 `LLMProviderRouter` / HTTP completions（带 tool calling）
	
- 若有 `tool_calls`：校验 → 风险判断 → 执行 → `on_tool_step` 推前端步骤条 → 结果当 tool 消息回灌
	
- 若模型不支持 tools：降级成普通 chat，打日志说明「no tools」
	
- 退出条件：无 tool_calls、触达 `max_steps`、或 `approval_required` 挂起



### Plan-and-Execute / ReWOO 呢？

先规划再执行（或规划与执行分离），可控性、可缓存更好；动态环境里计划易过时，常要重规划。

#### 为何不用纯模型自规划多 Agent

我零到一落地时发现：规划写在 prompt 里质量不稳定、难改、难审。把「规则红线 → 选型 → 出稿 → 还原 → 打磨」做成可配置节点后，质量更稳 —— 这就是用显式 Plan（图）替代隐式 Plan（prompt）。

> 内置 Clutch Agent 是 ReAct + 风险闸门；工作流更像显式 Plan（画布）+ Execute（节点），把不稳定规划外置成用户可编辑 SOP。


### 怎么防止 ReAct 死循环？

`max_steps`卡死最大轮数，同一工具、同一参数反复出现就当空转掐断，用户随时人工 Stop，任务完成判定（final answer / end 节点）。

#### 重复 tool 呢？

同 run 内风险工具用 `mcp_approval_key` 去重，避免同一高危调用反复弹审批；重复无意义调用靠步数上限 + 结果回灌让模型看到「已经试过」。


---

## Tool / Function Calling / MCP

### Function Calling 原理？

1. 把 tools 的 JSON Schema 放进请求（builtin + MCP 列表）。
	
2. 模型返回结构化 `tool_calls`（name + arguments）。
	
3. Sidecar 执行（可能先挂起等人审）。
	
4. 把 工具输出（文件内容、报错等）结果消息 append 回 conversation。
	
5. 再带着工具请求模型，直到纯文本 final answer。



### MCP 是什么？和自家 Function Calling 比？

> [!NOTE]
> Model Context Protocol：标准化「模型 ↔ 工具/资源/prompt」的接入协议，便于多客户端复用同一套 server。  

- 自研 FC：耦合你的后端；
	
- MCP：生态与边界更清晰。

> MCP ReAct + `is_risky_mcp_tool`；高危写删先审批再落盘——八股里要补一句：Tool 不只是能力，还有权限与副作用模型。

#### 项目的延伸

- 内置 FC：`builtin_tools`（读写、shell、todo、web、apply_patch…）—— 和产品强绑定，走同一套风险与 UI 步骤条。自家 FC 迭代快、权限好控；
	
- 外部 MCP：`mcp_client` 拉起用户配置的 Server「**<mark style="background:#d3f8b6">独立的外部程序，向 agent 暴露一批 tools</mark>」，工具名进同一 ReAct 循环。MCP 方便接社区/内部已有工具，但要统一鉴权、超时、错误回灌格式。[[MCP 缺点]]

> 为了解决上述问题，我们做了以下逻辑，实现来源可插拔：

- 执行入口统一（ReAct）「统一 模型出 `tool_calls` → Sidecar 执行 → 结果回灌 → 再问模型」
	
- 来源可插拔（builtin | MCP）。

> [!NOTE]
> Tool 不只是能力目录，还有副作用模型：读偏安全，write/edit/delete/shell 在 mcp_risk 里标高危；pause_on_risky 时审批通过才真正执行。Agent 模式可自动批准部分编辑，危险 shell/delete 仍可问人。


### Tool 设计原则

- <mark style="background:#d3f8b6">单一职责</mark>：  「一个 tool 只干一件事，名字、参数、返回都要围绕这一件事。」
	
- <mark style="background:#d3f8b6">参数可校验</mark>： 「入参后端前必须有明确 schema，调用前做校验和归一化。避免缺字段、格式不统一」
	
- <mark style="background:#d3f8b6">错误信息对模型友好</mark>： 「看到错误后要决定下一轮动作，容易自我修正。」
	
- <mark style="background:#d3f8b6">幂等优先</mark>： 「通过去重，同样的 tool 调用一次和调用多次，结果和副作用应该一致。」
	
- <mark style="background:#d3f8b6">危险操作显式</mark>： 「走 HITL」
	
- <mark style="background:#d3f8b6">返回结构化且短</mark>：「tool 返回给模型的信息要精简、结构化，不要大段原始日志。」


---

## Multi-Agent

### 多 Agent 怎么协作？常见拓扑？

- <mark style="background:#d3f8b6">Supervisor / Router</mark>：一个调度分发多个专家Agent <font color="#b7dde8">「任务派错专家，整体就失效」</font>
	
- <mark style="background:#d3f8b6">流水线 Pipeline</mark>：A→B→C，完整上下文输出，强顺序依赖 <font color="#b7dde8">「一步错步步错」</font>
	
- <mark style="background:#d3f8b6">辩论 / 评审</mark>：生成 + Critic <font color="#b7dde8">「增加 token 消耗，耗时变长」</font>
	
- <mark style="background:#d3f8b6">手递手 Handoff</mark>：把上下文摘要交给下一 Agent <font color="#b7dde8">「工作流用的」</font>


### 什么时候该多 Agent，什么时候不该？

- 需要：角色冲突（写 vs 审）、上下文隔离、工具权限分离、可替换专家。
	
- 不该：任务简单、通信开销 > 收益、状态难对齐（你强调的黑盒问题）。

> [!NOTE] 项目映射
> 画布节点 = 显式多 Agent；
> 
> 单 Chat = 单 Agent + 工具。
> 
> 监督台解决「多 Agent 黑盒、产物分散、流程难改」。


### Agent 间怎么传上下文？


1. <mark style="background:#d3f8b6">图内共享 State</mark>：LangGraph `CompilerState` + 节点输出汇总；下游注入上游产物。
	
2. <mark style="background:#d3f8b6">分层 System Prompt（D53）</mark>：`markdownDoc` 只当协议层，再叠 Environment、仓库规则（`AGENTS.md`/`CLAUDE.md`）、Skills 目录 —— 避免每个节点复制一整份巨 prompt。
	
3. <mark style="background:#d3f8b6">CLI 会话</mark>：`cli_session_id` 进 `ClutchState`，下一 turn `--resume`，不靠把全文历史塞命令行。
	
4. <mark style="background:#d3f8b6">长对话压缩</mark>：`compaction` 把中间轮归档到 `runs/archive/{run_id}.jsonl`，Chat 留 `system_digest`；Todo/计划经 `task_state` 每轮再注入，压缩后仍答得了「还剩什么」。
	
5. <mark style="background:#d3f8b6">摘要 handoff</mark>：PTY transcript 可经清理再摘要，避免把 ANSI 垃圾送给下一模型。


---


## Memory（必问，对接你痛点）

### Agent Memory 分哪几类？

- <mark style="background:#d3f8b6">短期</mark>：当前 turn / 窗口内 messages
	
- <mark style="background:#d3f8b6">长期</mark>：向量库、画像、偏好
	
- <mark style="background:#d3f8b6">工作记忆 / Scratchpad</mark>：todo、计划、中间变量
	
- <mark style="background:#d3f8b6">检查点</mark>：图状态可恢复（checkpoint）


### 上下文满了怎么办

截断、摘要（compaction）、重要信息钉住（todo/规则）、RAG 外置、分段 Agent。

> MemorySaver 挂起续跑；Compaction + 归档；Todo 钉进摘要；CLI session resume——四层都要能点名。

1. <mark style="background:#d3f8b6">MemorySaver 挂起续跑</mark>  
	
    工作流/运行状态可存盘；暂停后再开，从断点续，而不是上下文丢了就全没了。
    
1. <mark style="background:#d3f8b6">Compaction + 归档 </mark> 
	
    自动或手动压缩：原文归档 → 对话里留摘要气泡 → 关键任务状态（如 `task_state`）钉住。大约 15k tokens 量级可触发策略（量级概念即可）。
    
1. <mark style="background:#d3f8b6">Todo 钉进摘要 </mark> 
	
    待办不靠聊天记忆：用 todo 工具写入状态，之后每轮注入 prompt，压缩后也不丢「还要干什么」。
    
1. <mark style="background:#d3f8b6">CLI session resume</mark>  
	
    走 Claude/Codex 终端时，长记忆可以留在 CLI 自己的会话里；Clutch 主要记 `session id`，用 `--resume` 接着聊。


---

## 编排框架：LangGraph / 状态机

### 为什么用 LangGraph 而不是纯手写 while + if？

手写的话缺少条件边与校验（混用条件/无条件边编译期直接报错）、interrupt / checkpoint 级 HITL、与画布 JSON 的编译映射。

### StateGraph / Checkpoint / interrupt 是什么

State 是 SSOT；节点是 `state→state`；边决定下一跳；checkpointer 持久化中间态；`interrupt_before` 实现 HITL 挂起后续跑。

- <mark style="background:#d3f8b6">State</mark>：一次 run 的真相；对外投影是 `ClutchState`（`status`、`active_node_id`、`messages`、`cli_session_id`、shell 池字段等），经 WebSocket `state_patch` 增量推前端。
	
- <mark style="background:#d3f8b6">Node</mark>：`state → state`；我们用闭包把 `node` 配置打进去。
	
- <mark style="background:#d3f8b6">Edge</mark>：无条件扇出或 `when` 条件路由；`route_next` 读 check / human 信号。
	
- <mark style="background:#d3f8b6">Checkpoint</mark>：挂起时图状态可恢复。
	
- <mark style="background:#d3f8b6">interrupt_before</mark>：跑到 `human_gate` 前停下 → `awaiting_human` → 用户决策写入 `human_decision` → resume；用完清空，防下一闸门被上次 Allow 误放行。

### 和 AutoGen / CrewAI / 语义核的差异？

AutoGen 偏多 Agent 对话协作；CrewAI  角色班组协作；语义核偏企业插件编排。

选 LangGraph：要的是可控状态机 + 原生 HITL + 可被「JSON 编译器」驱动，服务零代码画布，而不是再包一层聊天式多 Agent。桌面本地 CLI / PTY 也更适合我们自己的 EngineRouter，而不是绑定某一云编排 SaaS。



---

## 规划、路由、Fallback
### Agent 路由怎么做？

规则（意图/权限/工具可用性）+ 模型分类 + 代价 / 延迟策略。  

1. 用户 / 节点选定 `agent_type`（clutch、claude-cli、codex-cli…）。
	
2. `ProviderSpec`（`provider_registry`）映射到 `RuntimeStrategy`：`SDK_NATIVE` <mark style="background:#d3f8b6">「内置Agent策略，LLM + 工具循环」</mark> / `SHELL_EXEC`<mark style="background:#d3f8b6">「外部 CLI 的默认策略，拼接命令」</mark> / `HTTP_DAEMON` <mark style="background:#d3f8b6">「长驻 HTTP 服务，直接发请求」</mark>
	
3. `/api/tools/status` 探测本机二进制是否 Connect。
	
4. `route_engine` 分流：MCP ReAct、Hybrid PTY、legacy subprocess、云 LLM。
	
5. 权限模式、是否 hybrid、Flow 多行 prompt（可能降级 legacy）再修正路径。


### Fallback 策略

CLI 不可用 → 云模型；视觉拒图 → OCR/调色板；工具失败 → 重试 / 换工具 / 升人审。强调失败可操作。



---

## 评估与可靠性
### 怎么评估 Agent？

任务成功率、步数/成本、工具正确率、是否越权、人审干预率。

- <mark style="background:#d3f8b6">契约/单测</mark>：后端用 pytest 把关键规则写成测试，改代码不能破坏约定。比如compiler 边校验、risk 工具识别等 pytest。
	
- <mark style="background:#d3f8b6">产品验收</mark>：任务是否完成、是否越权写盘、人审是否出现在该出现时、压缩后是否还能续做。
	
- <mark style="background:#d3f8b6">可观测</mark>：Terminal 审计字段（`run_id`/`source`/…）、hybrid turn JSONL（marker、耗时、result）。



### Safety / 权限

> 最小权限、写操作闸门、沙箱/工作区边界、审计日志、人在环。对接你的风险分级 + Allow / Reject / Retry。

- <mark style="background:#d3f8b6">最小权限</mark>：工作区白名单；Ask ≈ 只读对话。
	
- <mark style="background:#d3f8b6">写路径闸门</mark>：MCP 风险分级 + 图级 `human_gate`。
	
- <mark style="background:#d3f8b6">本机 loopback + Bearer token</mark>：防止本机别的程序乱调
	
- <mark style="background:#d3f8b6">审计</mark>：关键动作计入 terminal_logs / hybrid JSONL。方便事后对账。
	
- <mark style="background:#d3f8b6">同 workspace CLI 互斥</mark>：降低并写损坏。



---

## Prompt / 结构化输出

### System Prompt 怎么分层

<mark style="background:#d3f8b6">协议（markdownDoc）</mark>「人设/输出格式/行为公约」换Agent主要改这一层

→ <mark style="background:#d3f8b6">Environment</mark>「运行环境信息，跟这次运行环境走」

→ <mark style="background:#d3f8b6">仓库规则链</mark> 「当前仓库的 CLAUDE.md / 规则」

→ <mark style="background:#d3f8b6">Skills 目录</mark>（需要时 `read_skill`）「真正内容按需 `read_skill`」

→ <mark style="background:#d3f8b6">当前任务 / state</mark>「这轮要干什么，最易变、每轮更新」

换 Agent 只改协议层；换仓库自动换规则；Skills 按需读，避免一条 10k 巨 system 永久占窗。


### JSON / 结构化输出不稳怎么办

- 约束模型工具参数优先走 结构化 `tool_calls`，直接让 Sidecar 按工具名执行，不要靠模型散文里嵌 JSON
	
- <mark style="background:#d3f8b6">对返回值进行边界预处理</mark>：返回若是 `{content, reasoning_content}` 这类 dict，要设定解包规则，先解包再当字符串用（route 层踩过的坑）
	
- <mark style="background:#d3f8b6">todo 等入参要 normalize</mark>：可能是 list，也可能是 JSON 字符串，禁止乱 `list(str)` 要做兼容处理
	
- 编译/契约侧用校验（不合法就明确失败），而不是默默吞错。



---

## 和「传统工程」对比

### Agent 系统的本质难点是什么

相对传统工程「路径写死、结果可预期」，Agent 的本质难点在于决策非确定、控制流难完全预知，却又带着写盘/执行命令等真实副作用，还要同时扛住上下文膨胀、成本、评估和权限安全 —— 难的是在不确定中仍保持可控、可审计、可停止与可恢复。

### Agent 会取代工作流引擎吗

结合上个问题的内容，短时间内不会取代，引擎擅长固定 SOP、合规与人审闸门，Agent 擅长模糊目标下的局部探索；更务实的形态是工作流定骨架、节点内嵌 Agent，而不是把全部编排交给模型现场自规划。


