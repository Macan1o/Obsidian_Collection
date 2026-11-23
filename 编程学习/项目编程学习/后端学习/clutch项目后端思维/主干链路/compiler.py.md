---
created: 2026-08-12T14:56
---


---

> [!NOTE]
> 主要职责是将用户拖出来的工作流JSON编译成可执行的LangGraph「可跑的状态机流水线」

# 串联链路

	chat_runner  -> compiler  -> engine_router

核心逻辑在 [[LangGraph运转链路]]

# 核心逻辑

## 五种节点handler逻辑

[[LangGraph 、五handler 、_wrap_handler之间的运行关系]]

#### `_handle_start`

开始打开节点，主要使用`state.task_logs`、`run_id` 推送日志，几乎不动其他参数

- 拷贝 task_logs
	
- 追加一行 WorkFlow graph entered
	
- 经 `_stream_compiler_log` 推到 terminal

> 日志刚开始是空的，只有在进行到关键逻辑的节点 / 路由时才会emit / 塞进 task_logs里

#### `_handle_agent_task`

核心处理逻辑节点

- 若已取消 → 抛 `WorkflowCancelled`
	
- 读 `agent` / `label`，`find_agent` 解析展示名
	
- `emit_workflow_agent_step`：先推「正在跑」给前端
	
- `resolve_agent_task_input`：拼本步输入（用户指令 / 上游产出）
	
- `execute_agent_task` → 内部再到 `engine_router`
	
- 把 `result.logs` / `result.message` 累进托盘
	
- `node_outputs[node_id] = result.output`
	
- 再推一帧成功状态；再查一次取消

> 返回结构

```
active_node_id = 本节点 id
active_agent   = result.agent
status         = "running"
node_outputs   = 含本节点产出
task_logs / task_messages = 已追加
```


#### `_handle_check`

> 主要进行质检，通过作用check_result写成passed / failed 给条件边提供分支信号

- 有data.prompt 则走 `_llm_judge_check`（问模型 passed / failed）
	
- 无 workspace 默认 passed，往下走走到逻辑节点`evaluate_node_data`抛错误，防止卡死
	
- 否则 走 `evaluate_node_data` 读节点的checks配置，在工作区做文件存在 / shell-lint命令检查

> 返回结构

```
active_node_id = 本节点 id
active_agent   = "Evaluator"
status         = "running"
check_result   = "passed" | "failed"   ← 条件边靠这个分叉
task_logs / task_messages = 已追加
```


#### `_handle_human_gate`

通过interupt_before将节点注册好后，运行到该阶段状态机会停止。暂停节点挂起前利用Checkpoint（MemorySaver）先保存完整的graph状态快照，通过更新状态逻辑将人类决策注入到 CompileState 中，再利用LangGraph的Command（resume=）从断点继续执行

- 无decision ：标 awaiting_human 通过事件来用户做决定
	
- 有decision ：放行，保留human_decision；清空check_result「防止failed短路后面的check」


#### `_hanle_end`

只改状态字段方便前端更新状态






## `WorkflowCompiler` JSON拼成图

> 只做一件事，把画布JSON装成可 invoke 的 LangGraph 图，交出一台可 `invoke` 的LangGraph的机器

```
已校验的 workflow JSON
        │
        ▼
 WorkflowCompiler.compile()     ← 你在学的这块（只装机）
        │
        ▼
 CompiledGraph（挂好 _run + 边 + 闸门 interrupt）
        │
        ▼
 begin_workflow / chat_runner.invoke   ← 才真正开工
```

- 按type给每个节点做_run按钮挂上图
	
- 按边接好直连 / 条件分叉
	
- 给人工闸门注册「进入前暂停」



## `begin_workflow / resume_workflow` 开跑与人工续跑

### 开跑

#### 辅助函数

`initial_compiler_state(run_id, instruction)` 空容器、带上用户指令。

`workflow_run_config(run_id)` 告诉 LangGraph：这次运行的 checkpoint 线程是 run_id。换 run_id = 另一条互不干扰的运行。

`is_awaiting_human_gate(compiled, config, workflow)` 看 get_state().next 里是不是某个 human_gate。 `interrupt_before` 停下时，next 就是「下一步本该进、但还没进」的闸门 id。

#### 链路

- `compile_workflow(workflow)` — 装图（你前面学的 Compiler）
	
- `config = workflow_run_config(run_id)` — 绑定本次 run 的 checkpoint
	
- 准备初始 state — 默认 `initial_compiler_state`；缺指令则补上
	
- `compiled.invoke(state, config)` — 同步往下跑
	
		 一直跑到 END，或跑到第一个 human_gate 前被 interrupt 停住
	
- 打包 WorkflowSession — 留给以后 resume
	
- 若停在闸门前 — 把返回状态改成：
```
active_node_id = 闸门 id
active_agent   = "Supervisor"
status         = "awaiting_human"
```

> 开跑 = 编译图 + invoke，跑到闸门前停止通过 checkpoint 交出session，存在内存中

### 人工续跑

- 往 checkpoint 写决策
	
- `invoke(Command(resume=decision), config)` 解开 interrupt，从闸门继续。这时才会真正跑 `_handle_human_gate`（读到已有 decision → 放行）。
	
- 补全返回里的 human_decision（有的路径会丢）
	
- 若后面还有第二个闸门，再次标 awaiting_human，上层继续等人。
	
- 若这次跑完了（不在闸门）把 checkpoint 里的 human_decision 清掉，防止下一个闸门被「自动批准」。

续跑时闸门 handler 在干什么
```
resume 后进入 _handle_human_gate
  → 已有 human_decision
  → status=running，保留 decision 给条件边
  → 清空 check_result
  → 边按 approve/reject/retry 分叉往下走
```

> 续跑 = 在同一 session 写入人的决策 + Command（resume）恢复后再invoke 从闸门继续跑

#### 续跑id机制

通过`config = {configurable: {thread_id: run_id}}` 每次 `invoke` 都带同一份 config ，MemorySaver 按这个 thread_id 存/读进度