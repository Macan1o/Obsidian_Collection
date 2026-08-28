
---
## 总览

> [!NOTE]
> 前端 Chat 输入 → 后端 EngineRouter 路由 → ShellSession PTY 执行 → 输出解析 → WebSocket 回传 → 前端渲染 本文档不定义新规则，仅描述 Hybrid-Runtime 子系统的完整数据链路。

- 解析两种路由分流，一种内置Agent走LLM + MCP ReAct，另一种匹配对应的 CLI 路由分配
	
- 边界校验，通过则走PTY，不满足则降级到subprocess 隔离执行
	
- 通过全局池管理实例获取并创建bash PTY，加载会话快照，并且构建marker能够精确判断CLI执行完毕
		sessionSnapshot数据结构
```Python
@dataclass
class SessionSnapshot:
    run_id: str              # 会话 ID
    workspace_path: str      # 工作目录
    cwd: str                 # 当前路径
    git_branch: str          # 当时在哪个分支
    task_summary: str        # 上次任务摘要
    open_todos: list[str]    # 未完成的 TODO
    cli_session_id: str      # CLI 的会话 ID（用于 --resume）
    captured_at: str         # 快照时间戳
```
	
- 对截取的部分进行解析筛选出需要的部分经 WebSocket 增量合并后渲染到前端


## 前端

```
用户输入 "帮我重构 UserService" → ChatInput 组件
  │
  ▼
① WebSocket.send(json({
     action: "chat",
     payload: { run_id, text, agent_id, ... }
   }))
  │
  ▼
② React 处于"等待"投影态：
   - 显示 loading skeleton
   - terminal_logs 实时追加（state_patch 增量）
   - 侧栏 ↻ 指示器：由 shell_session_status 字段驱动
     · "ready" → 绿色圆点
     · "busy"  → 旋转动画
     · "disconnected" → 橙色 ↻ + tooltip
     · "terminated" → 灰色圆点
  │
  ▼
③ state_patch 事件到达 → useClutchState reducer 合并 patch
   messages[] 追加新消息 → ChatMessageList 增量渲染
   hybrid_executions[msg_id] → 展开 CLI 原始输出面板
   shell_session_status → 侧栏 Shell 状态指示器
```

## 后端（FastAPI + LangGraph + ShellSession PTY）

```
① WebSocket /ws → FastAPI lifespan (chat_runner.py:1412-1455)
   │
   ├─ _sweep_shell_sessions(): 每 60s 清扫一次
   │    · manager.sweep_idle() → 淘汰超时 IDLE session
   │    · prune_stale_snapshots() → 清理 30 天旧快照
   │
   ├─ WebSocket 消息循环 → 匹配 action="chat"
   │
   ▼
② _llm_chat_reply() → 解析 agent_id / agent_type
   │
   ├─ is_clutch_agent(agent) → LLM + MCP ReAct 路径
   │    └─ run_mcp_react_loop() → [MCP tool call → 审批门 → LLM 生成]
   │
   ├─ 非 Clutch agent → route_engine(raw=True)
   │    │
   │    ▼
   │ ③ _route_engine_raw() → 按 agent_type 路由：
   │    │
   │    ├─ anticragity-cli → CLI_ROUTING_CONFIGS["antigravity-cli"]
   │    ├─ claude-cli      → CLI_ROUTING_CONFIGS["claude-cli"]
   │    ├─ codex-cli       → CLI_ROUTING_CONFIGS["codex-cli"]
   │    ├─ aider-cli       → CLI_ROUTING_CONFIGS["aider-cli"]
   │    ├─ zcode-cli       → CLI_ROUTING_CONFIGS["zcode-cli"]
   │    ├─ opencode-cli    → CLI_ROUTING_CONFIGS["opencode-cli"]
   │    ├─ mimo-cli        → CLI_ROUTING_CONFIGS["mimo-cli"]
   │    ├─ codebuddy-cli   → CLI_ROUTING_CONFIGS["codebuddy-cli"]
   │    └─ ollama-cli      → chat_ollama() / 直连 LLM
   │    │
   │    ├─ 工作流模式下检测 shell 命令注入风险
   │    │    hybrid_pty_shell_command_risky() → 是 → legacy subprocess（双轨安全网）
   │    │
   │    ▼
   │ ④ try_shell_exec_hybrid() — 调度门控（runtime_registry.py）
   │    │
   │    ├─ 条件 1: provider_spec.runtime_strategy == RuntimeStrategy.SHELL_EXEC ✓
   │    ├─ 条件 2: hybrid_eligible(source, agent_type) ✓
   │    │    · CLUTCH_RUNTIME_MODE=hybrid
   │    │    · agent_type 在白名单中
   │    ├─ 条件 3: run_id 不为空 ✓
   │    ├─ 条件 4: workspace_path 不为空 ✓
   │    │
   │    ├─ 全部满足 → hybrid_route()（即 _route_generic_cli_hybrid）
   │    │
   │    ├─ 抛出 HybridPlainChatRejected → 向上传播：
   │    │    · run_in_progress    → 前端显示 "已在运行中"
   │    │    · session_busy       → 前端显示 Shell 繁忙
   │    │    · pool_full          → 前端显示 Shell 池满 + 队列位置
   │    │
   │    └─ 其他异常 → fallback to legacy_route()（subprocess 隔离）
   │
   ▼
⑤ _route_generic_cli_hybrid() — PTY 执行编排
   │
   ├─ 5a. ShellSession 获取
   │    manager = get_shell_session_manager()
   │    session = manager.get_or_create(run_id, workspace_path)
   │    │
   │    ├─ 已存在 & alive → 直接返回
   │    ├─ 已存在但已断开 → 重建 PTY (consume_shell_recovery)
   │    ├─ 不存在 → _spawn_with_timeout() 建立新 bash PTY
   │    │    · fork pty → exec /bin/bash
   │    │    · 发送 PS1='clutch$ ' 作为同步标记
   │    │    · cd 到 workspace_path
   │    │    · 状态: CREATING → READY
   │    ├─ 池满 (≥8) → _evict_oldest_idle_unlocked()
   │    │    · 驱逐最老的 IDLE/READY → close() → 再创建
   │    │    · 无 IDLE 可驱逐 → ShellSessionPoolFullError
   │    └─ 返回 BUSY 状态 session
   │
   ├─ 5b. 快照上下文注入
   │    snapshot = load_snapshot(run_id)
   │    if shell_recovered:
   │        context_prefix = format_handoff_prefix(snapshot)
   │        # 将 task_summary + open_todos 注入新 turn 的 CLI prompt
   │
   ├─ 5c. CLI 命令构建
   │    if agent == "claude":
   │        cmd = _build_claude_shell_cmd(
   │            claude_binary, prompt, marker, session_id,
   │            resume_session_id, system_prompt
   │        )
   │        # → CLUTCH_P='...'; claude -p "$CLUTCH_P" --session-id xxx; echo __CLUTCH_MARKER__
   │    elif agent == "antigravity":
   │        cmd = _build_agy_shell_cmd(...)
   │        # → CLUTCH_P='...'; agy -p "$CLUTCH_P" --conversation xxx; echo __CLUTCH_MARKER__
   │    else:
   │        cmd = _build_generic_cli_shell_cmd(...)
   │
   ├─ 5d. run_generic_cli_turn()
   │    │
   │    ├─ assert_no_interactive_command() — 阻止 vim/nano/ssh 等交互命令
   │    ├─ 获取 workspace CLI lock（asyncio.to_thread）
   │    │    · 超时: CLUTCH_SHELL_TURN_LOCK_TIMEOUT (默认 60s)
   │    │    · 同一 workspace 下只允许一个 CLI 进程活跃
   │    │
   │    └─ _execute_hybrid_turn()
   │         │
   │         ├─ write_line(session.master_fd, cmd)
   │         ├─ read_until_marker(session.master_fd, marker, timeout_s)
   │         │    · 轮询读取 PTY 输出，超时: CLUTCH_CLAUDE_CLI_TIMEOUT (600s)
   │         │    · 止于 marker = "__CLUTCH_xxx__" 字符串
   │         │    · 状态: BUSY → IDLE (执行完毕)
   │         │
   │         ▼
   │ ⑥ 输出解析 — 多阶段流水线
   │         │
   │         ├─ Stage 1: ClaudeHybridOutputParser.parse_structured()
   │         │    ├─ strip_ansi(raw) → 去除 ANSI 转义序列
   │         │    ├─ _erase_backspaces() → 处理 \b 退格擦除
   │         │    ├─ _strip_shell_preamble() → 移除 CLUTCH_P=...echo __CLUTCH_*__
   │         │    ├─ 按行分类 → OutputEvent 类型：
   │         │    │    · assistant       → AI 回复内容行
   │         │    │    · tool            → CLI 工具调用行
   │         │    │    · shell_echo      → Shell echo 原样输出
   │         │    │    · system_prompt   → 系统提示词注入
   │         │    │    · boundary_marker → 完成标记
   │         │    │    · ansi/debug      → 噪音行（前端不展示）
   │         │    │    · stderr          → 错误输出
   │         │    └─ HybridParseResult(events, raw)
   │         │
   │         ├─ Stage 2 (fallback 1): codex parser
   │         │    extract_codex_assistant_output(raw)
   │         │    # 仅当 Claude parser 返回空 assistant 事件时触发
   │         │
   │         ├─ Stage 3 (fallback 2): CLI issue extraction
   │         │    extract_cli_issue_message(raw)
   │         │    # 提取认证/连接/配置错误信息
   │         │
   │         └─ Stage 4 (fallback 3): assistant line extraction
   │              _extract_assistant_lines(raw)
   │              # 降级为原始行提取
   │
   ├─ 5e. 审计记录
   │    _record_hybrid_turn_audit(
   │        session, agent, turn_id, marker, cmd,
   │        duration_ms, result, cli_session_id, message
   │    )
   │    → audit log JSONL append
   │
   ├─ 5f. 快照持久
   │    _persist_hybrid_turn_snapshot(
   │        run_id, workspace_path, cli_session_id, prompt
   │    )
   │    → upsert_hybrid_turn_snapshot() → DB
   │
   └─ finally: manager.mark_idle(run_id)
   │
   ▼
⑦ EngineResult 返回 → _llm_chat_reply 组装响应
   │
   │  result.output              → chat message text
   │  result.raw_output           → hybrid_executions[msg_id].rawOutput
   │  result.output_events        → hybrid_executions[msg_id].outputEvents
   │  result.cli_session_id       → 前端暂存，下次复用
   │  result.shell_recovered      → 触发前端 ↻ 恢复提示
   │
   ▼
⑧ WebSocket 回传 — 三条事件管线
   │
   ├─ state_patch 事件：
   │    {
   │      "event": "state_patch",
   │      "data": {
   │        "run_id": "run_xxx",
   │        "patch": {
   │          "messages": [...新增消息],
   │          "hybrid_executions": { msg_id: { rawOutput, outputEvents } },
   │          "shell_session_status": "ready",
   │          "terminal_logs": [...],
   │          "token_input": ...,
   │          "token_output": ...
   │        }
   │      }
   │    }
   │
   ├─ message 事件（逐条消息）：
   │    {
   │      "event": "message",
   │      "data": { "run_id", "node_id", "message": {...} }
   │    }
   │
   └─ hybrid_execution 事件（CLI 原始输出）：
        {
          "event": "hybrid_execution",
          "data": { "run_id", "messageId", "rawOutput", "outputEvents": [...] }
        }
```

## ShellSession 生命周期全景

```
                   Session Pool (max 8, 全局单例)
 ┌─────────────────────────────────────────────────────────────┐
 │                                                             │
 │  get_or_create(run_id, workspace_path)                      │
 │    │                                                        │
 │    ├─ pool[run_id] 存在?                                    │
 │    │   ├─ alive() == True → 返回 BUSY                      │
 │    │   └─ alive() == False → 重建 (DISCONNECTED → READY)   │
 │    │                                                        │
 │    ├─ pool.size >= 8?                                       │
 │    │   ├─ _evict_oldest_idle_unlocked()                     │
 │    │   │   · oldest IDLE/READY → close()                    │
 │    │   │   · 驱逐成功 → 继续创建                             │
 │    │   └─ 驱逐失败 → ShellSessionPoolFullError              │
 │    │                                                        │
 │    └─ _spawn_with_timeout(workspace_path, timeout=15s)      │
 │        ├─ fork() → exec /bin/bash                           │
 │        ├─ send PS1='clutch$ ' (同步标记)                     │
 │        ├─ cd workspace_path                                  │
 │        ├─ CREATING → READY                                  │
 │        └─ 返回 BUSY session                                 │
 │                                                             │
 ├─ 状态机:                                                    │
 │   CREATING → READY → BUSY → IDLE → (循环)                   │
 │                     ↓                                       │
 │               DISCONNECTED → RECOVERING → READY              │
 │                     ↓                                       │
 │               TERMINATED (不可恢复)                           │
 │                                                             │
 ├─ 定时清扫 (每 60s):                                         │
 │   sweep_idle()                                              │
 │   ├─ idle > CLUTCH_SHELL_IDLE_SEC (30min) → TERMINATED      │
 │   └─ age > CLUTCH_SHELL_MAX_SEC (6h) → TERMINATED           │
 │                                                             │
 └─ mark_idle(run_id): BUSY → IDLE (turn 完成后由调用方触发)    │
```


## 问题解析

### 为什么需要 Hybrid / Legacy 双轨

```
问题背景：
  Clutch 最初使用 subprocess 隔离执行 CLI（每个 turn fork 一个进程），
  每次执行需要重新加载 CLI 上下文、重建 conversation history，
  对多轮对话体验不友好且浪费资源。

双轨设计理由：
  ① PTY (Hybrid) = 主要路径：
     - 长生命周期 bash PTY，跨 turn 保持 CLI session
     - 通过 claude --resume / agy --conversation 延续对话
     - 输出通过 PS1 marker 同步精确截断
     - ShellSession 池化管理，复用 PTY fd

  ② Subprocess (Legacy) = 安全网：
     - 当 Hybrid 不可用时的无条件降级
     - 不依赖 PTY 状态，完全隔离
     - 用于：非 hybrid 模式、风险命令检测、Hybrid 异常回退

切换决策树（try_shell_exec_hybrid）：
  CLUTCH_RUNTIME_MODE=hybrid?
  └─ agent_type 在白名单?
     └─ run_id + workspace_path 非空?
        └─ ShellSession 可用?
           ├─ YES → Hybrid PTY 执行
           └─ NO  → HybridPlainChatRejected (不降级，让用户感知)
        └─ 异常 → 自动 fallback to Legacy subprocess
```

### Marker-based 完成检测如何工作

```
问题背景：
  PTY 是字符流，没有天然的"命令完成"信号。需要一种机制
  精确判断 CLI 何时输出完毕。

Marker 协议：

① 命令注入 marker：
   _build_claude_shell_cmd() 构建的命令末尾：
   
   CLUTCH_P='user prompt'; claude -p "$CLUTCH_P" ...; echo __CLUTCH_a1b2c3__
                                                      ↑
                                              唯一 marker (uuid)

② 读取循环（read_until_marker）：
   while True:
       chunk = os.read(master_fd, 4096)
       buffer += chunk.decode('utf-8', errors='replace')
       if marker in buffer:
           return buffer  ← 检测到 marker，输出完整
       if elapsed > timeout_s:
           raise TimeoutError

③ marker 验证（marker_completed_in_output）：
   # 确保 marker 不是被截断或混淆
   return marker in raw

④ marker 生成：
   marker = f"__CLUTCH_{uuid.uuid4().hex[:8]}__"
   # 例: __CLUTCH_a1b2c3d4__

为什么用 echo 而不是 exit code？
  - PTY 上的 exit code 不可靠（bash 可能被后台进程影响）
  - echo 是同步的：只有当 claude 进程退出后，echo 才会执行
  - marker 在 claude 命令的 ; echo __CLUTCH_xxx__ 中，
    确保 claude 已完成 stdout 写入
```

### ShellSession 池化管理如何应对并发

```
问题背景：
  多个 tab / 多个 workflow 可能同时请求 ShellSession，
  需要在有限资源（max 8 PTY）下公平调度。

并发控制层次：

Layer 1: ShellSessionManager (全局)
  - asyncio.Lock 保护 get_or_create()
  - max 8 active sessions (CLUTCH_SHELL_MAX_SESSIONS)
  - 池满驱逐策略：oldest IDLE/READY → close()
  - 全部 BUSY 时：ShellSessionPoolFullError

Layer 2: Workspace CLI Lock (文件级)
  - 同一 workspace_path 下只允许一个 CLI 进程
  - 通过 run_generic_cli_turn() 中的 on_lock_wait 回调获取
  - 超时：CLUTCH_SHELL_TURN_LOCK_TIMEOUT (60s)
  - 目的：防止两个 claude 进程同时写同一目录

Layer 3: ShellSession BUSY 状态 (会话级)
  - ShellSessionBusyError：尝试对 BUSY session 写入新命令
  - 由 ShellSessionManager 在 get_or_create 时检查

优先级：
  无优先级队列，FIFO + 驱逐。池满时新请求等待或失败。

前端反馈：
  - shell_pool_blocker_run_ids[]  → 哪些 run 被阻塞
  - shell_pool_queue_position     → 当前排队位置
  - shell_pool_queue_depth        → 总排队深度
  - shell_session_status          → 每个 run 的 Shell 状态
```

### 输出解析器的多阶段 fallback 长什么样

```
问题背景：
  CLI 输出格式复杂：ANSI 颜色、进度条、退格擦除、多行代码块、
  认证错误、网络超时等。单一解析器无法覆盖所有场景。

解析流水线 (ClaudeHybridOutputParser):

① Pre-processing（通用）
   ├─ strip_ansi(raw)           → \x1b[...m → ""
   ├─ _erase_backspaces(raw)    → "a\bb" → "b" (处理退格覆盖)
   ├─ _strip_shell_preamble(raw) → 移除 CLUTCH_P=...echo __CLUTCH_*__
   └─ 噪音行过滤（空行、纯 ANSI、进度条行）

② Stage 1: Claude Structured Parser（主解析器）
   ClaudeHybridOutputParser.parse_structured(raw) → HybridParseResult
   
   逐行分类：
   ┌─────────────────────┬──────────────────────────────────┐
   │ OutputEvent.type    │ 判定逻辑                          │
   ├─────────────────────┼──────────────────────────────────┤
   │ assistant           │ 非特殊前缀的普通文本行              │
   │ tool                │ 以 Tool: / Bash: / Read: 等开头     │
   │ shell_echo          │ 回显的命令行或 Shell 提示符          │
   │ system_prompt       │ 注入的系统提示词内容                │
   │ boundary_marker     │ __CLUTCH_xxx__ 检测到 marker        │
   │ ansi / debug        │ 纯 ANSI 序列或 debug 日志           │
   │ stderr              │ Error: / Traceback / 异常堆栈       │
   └─────────────────────┴──────────────────────────────────┘

   前端渲染规则：
   - assistant (visible: true)  → ChatMessage 气泡
   - tool (visible: true)       → 可折叠工具调用面板
   - system_prompt (visible: false) → 不渲染（仅后台记录）
   - ansi/debug (visible: false)    → 不渲染

③ Stage 2 (fallback): Codex Parser
   触发条件：Stage 1 返回 0 条 assistant 事件
   
   extract_codex_assistant_output(raw)
   → 尝试识别 Codex CLI 的 JSON 输出格式
   → 提取 data.message.content.text 字段

④ Stage 3 (fallback): CLI Issue Extraction
   触发条件：Stage 2 也未找到有效内容
   
   extract_cli_issue_message(raw)
   → 提取认证错误 / 连接失败 / 配置缺失等用户可操作信息
   → 前端展示为 error 级别消息

⑤ Stage 4 (fallback): Raw Assistant Line Extraction
   触发条件：以上全部失败
   
   _extract_assistant_lines(raw)
   → 贪心提取所有非系统行作为 assistant 输出
   → 降低结构化程度，但确保用户总能看到输出
```

### 快照与上下文连续性如何工作

```
问题背景：
  ShellSession 断开后重建 PTY，之前的 CLI conversation 上下文
  丢失。需要通过快照机制在恢复时注入上下文。

快照生命周期：

① 写入时机：
   每次 _route_generic_cli_hybrid 成功执行后：
   upsert_hybrid_turn_snapshot(
       run_id, workspace_path, cli_session_id, prompt
   )
   → 存储 SessionSnapshot { run_id, workspace_path, cwd,
       task_summary, open_todos, cli_session_id }

② 读取时机：
   _route_generic_cli_hybrid 开头：
   snapshot = load_snapshot(run_id)
   shell_recovered = manager.consume_shell_recovery(run_id)
   
   if shell_recovered and snapshot:
       context_prefix = format_handoff_prefix(snapshot)
       → "[Previous task: ...] [Open TODOs: ...] Continue where left off."

③ 注入方式：
   context_prefix 通过 run_generic_cli_turn 的 context_prefix 参数传入，
   追加到 CLI prompt 前面，使 CLI 在恢复时了解之前的任务上下文。

④ 清理：
   prune_stale_snapshots() (30 天) → 删除过期快照

⑤ 前端感知：
   shell_recovered = True → EngineResult.shell_recovered = True
   → 前端显示 "Shell 已恢复，上下文已注入" 提示
```

