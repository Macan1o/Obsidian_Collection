---
created: 2026-08-12T16:01
---


---

> [!NOTE]
> LangGraph已经决定「Agent该干活了」，该组件主要负责用哪条引擎真正执行「本地CLI / Hybrid Shell / Ollma / 云端LLM」，属于逻辑执行的分发层

主要解决了 编排 和 执行（用什么工具跑）解耦 ，也就是 `agent_executor` 的下一链路。如果没有这个模块，`chat_runner`、`agent_executor`、compiler 节点会各自拼 CLI 参数、各自处理会话/失败，代码会炸成一锅粥。


## 核心分流逻辑

[[执行引擎分流]]

## 关键模块解读

### `CLI_ROUTING_CONFIGS`

选定某家CLI之后，命令怎么拼的配置字典。例如：

	claude -p "你的问题" --dangerously-skip-permissions

### `EngineResult`

#### 主要逻辑

是 `route_engine` 的统一回执，不管走 Claude / HTTP / Hybrid / Legacy，统一收束成该数据结构，`chat_runner` / `agent_executor` 只读 `EngineResult`，把 `output` 写进消息、`logs` 推 Terminal、有 `cli_session_id` 就存起来下次带上。


### `route_engine`

#### 主要逻辑

1. E2E 沙箱？ → 直接返回假结果（不真跑 CLI）
	
2. 否则 → 调 `_route_engine_raw`（真正分流执行）
	
3. 把结果 sanitize 一遍 → 包成 EngineResult 返回

类似于一层编排层，先前置检验是否是沙盒测试，然后调核心方法 `_route_engine_raw`  ，将处理后的数据通过`sanitize_engine_output` 方法清洗一遍后，构造成 `EngineResult` 返回


### `_route_engine_raw`

真正执行分流的核心逻辑，按优先级试分支，命中就 return。

[[执行引擎分流]]

#### 内置 Agent 和 HTTP LLM 的区别

Clutch Agent 

> 带手脚的 Agent 产品形态

是一等公民 Agent（`agent_type=clutch`，策略 `SDK_NATIVE`）。它会选你在 Settings 里配的云模型，用 HTTP 调 Completions，但外面还包着一整层：默认挂 `clutch-tools`、走 MCP ReAct（想→调工具→看结果）、权限与高危审批、计划/提问卡、读写工作区、可选再绑 Hub MCP。对用户来说，这是「能动手的本地 Agent」。


HTTP Completions LLM 

> 无工具时的纯聊天兜底

更偏 拨号层 / 兜底路径：`llm/http_complete.py` 和 `LLMProviderRouter` 负责按 Key 打 OpenAI 兼容接口。路由里若没有可用 CLI，或只需要纯文本补全，就会落到「只聊、不带工具箱」的 Completions。它也可以被 Clutch Agent 当「大脑」来用，但单独出现时通常没有那套工具循环和闸门。