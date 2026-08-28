
---

> [!NOTE]
> 主要是梳理从用户**配置一个 Agent**，到 `route_engine` 选定执行引擎、组装 prompt / session / 工具， 再到 **CLI 执行 / 云端 LLM / MCP 工具调用**，最终把结果经 WebSocket 推回 Chat UI 的完整 API / 方法 / 数据流向

## 链路总览

- **Agent 配置模型** — 用户自定义 Agent = `dict`（prompt + engine + MCP + Skills），整表存 `agents.json`
	
- **路由分发** — `route_engine` 三步定位引擎（find_agent → resolve_provider_spec → tool_available_for_routing）→ 四条出口
	
- **Prompt 组装** — `compose_agent_system_prompt`：header + 用户协议 + Skills 全文注入
	
- **会话绑定** — `cli_session_id` 生成 / `--resume` 复用 / 切 Agent 废弃
	
- **执行三路径** — CLI 适配器体系（配方表 + argv 组装 + hybrid/legacy）｜LLM Provider Router（多模型云 API）｜clutch + MCP ReAct
	
- **能力工具** — MCP（stdio JSON-RPC + ReAct≤10 + 审批门）+ apply_patch（虚拟 MCP `clutch-tools`）
	
- **结果回传** — `EngineResult` → `state_patch` / `message` 事件 → 前端 `ClutchState` 投影


---

## 阶段一：路由分发 — RuntimeStrategy 策略模式

```
用户在 Chat 输入 "帮我写一个 Python 脚本"
  │
  ▼
WebSocket → chat_runner（plain_chat）/ agent_executor.execute_agent_task（flow）
  │
  ▼
engine_router.route_engine(agent_name="Researcher", prompt="帮我写...", source="plain_chat")
  │
  ├─ ① find_agent("Researcher")                       // 读 agents.json，模糊匹配
  ├─ ② agent_type_from_record → "claude-cli"
  ├─ ③ resolve_provider_spec("claude-cli")
  │       → ProviderSpec(runtime_strategy=SHELL_EXEC)     // 注册表统一状态
  │
  ├─ ④ tool_available_for_routing("claude-cli")?
  │       ├─ claude 二进制在 PATH? 或已 Connect? 或 CLUTCH_E2E_FAKE_HYBRID?
  │       └─ YES ✅
  │
  ├─ ⑤ 分四条出口：
  │       ├─ agent_type=="clutch"  → 云端 LLM / 生图 / 生视频（阶段四·路径 B/C）
  │       ├─ "ollama-cli"          → chat_ollama（HTTP_DAEMON）
  │       ├─ CLI_ROUTING_CONFIGS 命中 → CLI（阶段四·路径 A）
  │       └─ CLI 不可用            → fallback 云端 LLM
  │
  └─ ⑥（CLI 路径）CLI_ROUTING_CONFIGS["claude-cli"] 取配方
          → binary_name="claude", conversation_mode="separate", prompt_flag="-p", ...
          → 选 hybrid_fn: "claude"→_route_claude_hybrid / "agy"→_route_agy_hybrid / 其它→_route_generic_cli_hybrid
          → try_shell_exec_hybrid(  // 策略模式分发核心
                spec.runtime_strategy == SHELL_EXEC ✅
                hybrid_eligible(source, agent_type) ✅         // 白名单 + CLUTCH_RUNTIME_MODE=hybrid
                run_id ∧ workspace_path ✅
             )
             ├─ 全满足 → hybrid_route（长驻 bash PTY，见第 3 部分）
             └─ 否则/异常 → legacy_route（一次性 subprocess）
```

### Hybrid 三重闸门

`SHELL_EXEC` + `hybrid_eligible` 白名单 + `run_id/workspace` 齐备，缺一即 legacy。


---

## 阶段二：Prompt 组装 + Skills 注入

```
Agent 配置
  ├─ name, markdownDoc          → 一定进 prompt（①②）
  ├─ agentType == clutch?
  │     ├─ 否（claude-cli 等）  → 拼完 header+协议就结束（③④不加）
  │     └─ 是
  │           ├─ 有没有绑 MCP？
  │           │     ├─ 没有 → 加一句「你不能改文件」
  │           │     └─ 有   → 加 "Workspace root: ..."
  │           └─ agent.skills 非空？
  │                 └─ 是 → 把对应 SKILL.md 全文贴上（④）
  ▼
拼好的 system prompt 字符串
  → 和用户消息一起进入阶段三/四执行
```

如果存在 skill 就把 skill.md 拼接到提示词中；如果绑了 MCP，就通过字段 tool 给模型，让模型自主调用函数


---

## 阶段四：执行三路径

### 路径 A — CLI 适配器体系（SHELL_EXEC）

```
_route_generic_cli_hybrid（+ 薄包装 _route_claude_hybrid / _route_agy_hybrid）
  → ShellSessionManager.get_or_create(run_id, workspace)   // 长驻 bash PTY 池
  → run_generic_cli_turn → 构造 cmd + marker → write_line → read_until_marker → 解析
  → EngineResult(engine="Claude CLI (Hybrid)", cli_session_id, raw_output, output_events)

Legacy 分派：默认 chat_generic_cli；claude→chat_claude_cli；zcode→chat_zcode_cli
```

### 路径 B — LLM Provider Router（云端 API / SDK_NATIVE）

```
router.chat(messages, model_id, tools?)
  → resolve_for_model → (ModelSpec, api_key)      // provider_id 选 base_url/鉴权/格式
  → http_chat_complete(provider_id, base_url, api_model, api_key, ...)
       ├─ _openai_chat   /chat/completions   (deepseek/openai/google/agnes/custom/ollama)
       ├─ _anthropic_chat /messages          (anthropic 原生)
       └─ OpenCode Zen transport 分流
  → 返回 {content, reasoning_content} | 含 tool_calls 的 dict
  → LLMProviderRouter.extract_content(raw)         // ★ 必须先解包，否则 re.sub 会 TypeError
```


### 路径 C — clutch + MCP ReAct（工具调用）

> clutch 引擎绑定了 MCP 时，进入 ReAct 循环边想边调工具

```
resolve_agent_mcp_servers(agent)
  → [local-fs（npx server-filesystem）, clutch-tools（虚拟，含 apply_patch）, ...勾选]
  → run_mcp_react_loop(max_steps=10)
       ├─ connect(McpClient stdio JSON-RPC) → list_tools → openai_tools（tools 参数注入）
       └─ loop(≤10):
            response = router.chat(messages, tools)          // 调 LLM（路径 B）
            ├─ tool_calls?
            │    ├─ is_risky_mcp_tool? (write/delete/patch...)
            │    │    ├─ 高风险 + ask → 暂停 approval_required → human_required 事件
            │    │    │      → 用户 human_decision(approve) → 带 approved_tool 续跑
            │    │    └─ 放行/auto_edit → _execute_tool_call → observation 回填
            │    └─ virtual server → execute_builtin_tool（apply_patch）
            │           → parse_patch → resolve_allowed_path(白名单) → 写盘
            │           → file_changed 事件（前端刷新文件树）
            └─ 无 tool_calls → extract_content → 收尾
```


---

## 引擎决策矩阵   / 名词积累
