
---

> [!NOTE] 作用
> 该组件是 Chat / 工作流运行的「大脑」：属于中枢编排层，指挥其他方法完成逻辑，负责接收用户消息与开跑指令，编排 Agent/工作流，把 ClutchState（运行真相）改完后经 WebSocket 推给前端。

## 核心逻辑

1. 接活：HTTP 开跑、WS 收消息、人工闸门决策
	
2. 干活：纯聊走单 Agent；画布 SOP 走 LangGraph 工作流
	
3. 投影：改内存态 → 落盘 → `state_patch` / `message` / `log` 推前端（React 只渲染，不自己 invent 进度）

![[Pasted image 20260806102504.png|734]]


## 核心场景

### 场景A ：chat

![[Pasted image 20260806102750.png|720]]

### 场景B ：flow

1. HTTP `start_run` → `_run_workflow`
	
2. `compile_workflow` + `compiled.invoke`（LangGraph）
	
3. 若卡在人工闸门 → `awaiting_human`，推 `human_required`
	
4. 用户 Approve → `_apply_human_decision` → `resume_workflow`
	
5. 结果投影回 `ClutchState`，前端画布节点高亮跟着变


## 运转链路汇总

| 用户动作      | 走过的模块                   |
| --------- | ----------------------- |
| 纯聊发消息     | 十 → 五 → 六 → 一 → 二（暂停则七） |
| 开跑工作流     | HTTP → 三 → 一 → 二        |
| 点 Approve | 十 → 四 → 三续跑 → 一 → 二     |
| 工作流里 @ 精修 | 十 → 八 →（可能五/六）→ 一 → 二   |

## 逻辑模块解读

### 模块一：状态仓

组件函数 ：`_get_or_create_run` · `_merge_patch` · `_commit_run_state` · `_touch_session`
全局函数：`_run_states` · `_run_sessions`

#### 逻辑链路

1. 用 `run_id` 找内存里的 `ClutchState`；没有就从磁盘加载，再没有就 `initial_state`
	
2. 业务只产出「补丁」`patch`，经 `_merge_patch` 合进现有 state
	
3. `_commit_run_state`：写回内存 + 落盘（`run_state_store`）
	
4. `_touch_session`：更新侧栏会话列表（标题、状态等）
	
5. `_run_sessions`：工作流专用，缓存 LangGraph 的 `WorkflowSession`


### 模块二：推送事件（对前端说话）

组件函数：`_send_state_patch` · `_send_message_event` · `_send_log_event` · `_send_human_required` · `_send_run_completed` · `_send_file_changed` · `_send_validation_result` · `_notify_run_state` · `_try_ws_notify`

#### 逻辑链路

1. 业务改完 state 后，组装 `{event, data}` 信封，经 WebSocket 发出
	
2. `state_patch`：增量更新前端投影（消息列表、status、工具步骤等）
	
3. `message` / `log`：单条气泡或 Terminal 行
	
4. `human_required`：弹出人工闸门 UI
	
5. `run_completed`：终态（passed/failed）收尾
	
6. `file_changed`：Changes 面板 diff
	
7. WS 断了用 `_try_ws_notify` 吞错，不拖垮主流程


### 模块三：工作流开跑与步进

组件函数：`_run_workflow` · `_apply_workflow_step_patch` · `_apply_workflow_refining_pause` · `_emit_workflow_graph_tail` · `_merge_graph_resume`

#### 逻辑链路

1. `_run_workflow`：解析 workflow → 编译 LangGraph → `invoke` 开跑
	
2. 跑之前可把用户 instruction 写成 User 消息，status 设为 `running`
	
3. 注册 step callback：图每走一步，`_apply_workflow_step_patch` 把节点进度刷进 state 并推前端
	
4. 若卡在人工节点 → status 改成 `awaiting_human`
	
5. 取消/中断 → 可能进入 `refining` 暂停态
	
6. 正常结束 → `project_graph_to_clutch` 把图结果投影回 ClutchState 并 commit
	
7. `_merge_graph_resume`：resume 后把图返回的消息/日志/status 合并回 state


### 模块四：工作流人工闸门

组件函数：`_apply_human_decision` · `_apply_human_decision_locked`

#### 逻辑链路

1. 只有 `status == awaiting_human` 才处理；否则当重复点击直接忽略
	
2. 用 lock + `_human_decision_inflight` 防并发双点
	
3. `approve`：写 Supervisor 气泡 → 先标 `running` → `resume_workflow` 继续图
	
4. `reject`：标 `failed`，结束本次 run
	
5. `retry`：带用户 instructions 再 resume
	
6. resume 结果经 `_merge_graph_resume` 写回，再推 patch；若仍卡闸门，再发 `human_required`


### 模块五：chat主流程

组件函数：`_handle_plain_chat` · `_persist_plain_chat_user_message` · `_finish_plain_chat_after_llm` · `_apply_plain_chat_stop` · `_recover_stuck_plain_chat`

#### 逻辑链路

1. 解析 Agent（默认内置 Clutch Agent）
	
2. 先落 User 消息、status=`running`、清空临时 tool/reasoning，commit 并推前端
	
3. 定义一堆 `emit_*` 回调（log、tool_step、reasoning、todos…），执行中实时刷 UI
	
4. 调用 `_llm_chat_reply` 真正跑一轮 Agent
	
5. 若引擎返回 MCP/Plan 暂停 → 走审批模块，不立刻结束
	
6. 否则 `_finish_plain_chat_after_llm`：写入助手回复、封存 tool steps、更新 token/文件变更、标终态
	
7. `stop`：打断 shell/任务，把 status 拉回可继续态
	
8. 任务异常：`_recover_stuck_plain_chat` 防止卡在 `running`


### 模块六：chat引擎桥（调手脚）

组件函数：`_llm_chat_reply` · `_compose_agent_system_prompt` · `_history_for_llm` · `_uses_configured_llm`

#### 逻辑链路

1. 按 Agent 解析模型（Clutch 云模型 vs CLI 类型）
	
2. 处理附件图/出图出视频等特殊模型分支
	
3. 拼 system prompt + 裁剪历史 `_history_for_llm`
	
4. 调用 `engine_router.route_engine` 执行（内部再调 LLM/CLI/MCP/工具）
	
5. 执行过程通过传入的 `emit_*` 往上回调，由模块五写 state
	
6. 返回：回复文本、日志、mcp_pause、改过的文件、raw_output 等元组，交给模块五收尾


### 模块七：chat审批（Plan / 提问 / MCP）

组件函数：`_handle_plain_chat_mcp_decision` · `_patch_plan_card_status` · `_patch_question_card_status` · `_messages_for_mcp_pause` 等

#### 逻辑链路

1. 从 `mcp_pending` 取当前待批工具
2. 区分：普通 MCP 工具 / `propose_plan` / `ask_user_question`
3. `approve`：记录批准 → 带着 tool 结果再调 `_llm_chat_reply` 继续
4. `reject`：拒绝该工具，通知模型后继续或结束
5. Plan 的 `revise`：改 plan 卡片状态 → 把用户反馈塞回对话 → 让模型重新 `propose_plan`
6. Question：用户选项或自由回答写回卡片，再继续对话


### 模块八：flow侧边聊天 + Refine

组件函数：`_handle_workflow_chat_message` · `_handle_flow_refine_message` · `_commit_flow_refine_and_continue` · `_prepare_workflow_refine_state`

#### 逻辑链路

1. `_handle_workflow_chat_message`：工作流运行中用户再发消息
    - 若符合 Refine 触发 → 转 Refine
	    
    - 若正在 `awaiting_human` → 当成闸门 `retry`（带这段文字当指令）
	    
    - 否则只追加 User 消息/日志，不自动开跑
2. Refine：进入 `refining`，用 Agent 改某节点草稿输出
	
3. `_commit_flow_refine_and_continue`：把草稿写回图节点 → `continue_workflow_after_refine` → 投影 state → 可能再次进闸门


### 模块九：Hybrid / 队列 / 壳进程

组件函数：`_apply_pool_full_queue` · `_resume_pool_queued_turn` · `_refresh_pool_queued_run_states` · `_interrupt_plain_chat_shell` · `_interrupt_workflow_run` · `_apply_bg_jobs_update` · `_apply_foreground_shell_update` · `_send_pty_output` 等

#### 逻辑链路

1. Hybrid 下 shell 池满：用户消息先入队，UI 显示排队位
	
2. 有空位后 `_resume_pool_queued_turn` 再真正开纯聊
	
3. stop：打断 plain chat shell 或 workflow cancel
	
4. 后台 job / 前台 shell 状态变化 → 写 state + 推 patch
	
5. 交互 PTY 输出经 `_send_pty_output` 推到对应 lane

### 模块十：WS 总入口（分发器）

组件函数 ： `ws_run`（内部还有 `_start_plain_chat_turn` · `_enqueue_plain_chat` · `_drain_plain_chat_queue`）

#### 逻辑链路

1. 鉴权 → accept → 从磁盘同步 state → 注册各类 notifier
	
2. 首包推送全量 state；若已在闸门/终态，补发对应事件
	
3. 循环 `receive_text`，按载荷分流：
    - 有 `text` 且有 `workflow_id` → 模块八（或 Refine）
	    
    - 有 `text` 且无 workflow → 模块五（忙则入队）
	    
    - `human_decision` → 有 pending MCP 走模块七，否则走模块四
	    
    - `stop_run` / `continue_run` → 停/续
	    
    - `pty_*` / `worktree_*` / `kill_bg_job` 等 → 模块九
4. 断开时注销 WS、notifier，清理转发器