
---

> [!NOTE]
> Clutch 的 Skills/MCP 只注入 Clutch Agent（云端 LLM + ReAct）。Claude CLI 等走自己环境，Clutch 不叠加 Hub / Registry。

## 总览

```
设置页配置（落盘）
    ↓
Agent 勾选绑定（Agent 档案里记 ids）
    ↓
发 Chat 时组装
    ├─ Skills → 写进 system prompt 目录（正文用 read_skill 再读）
    └─ MCP    → 起 stdio 进程，工具列表交给 LLM 循环调用
```


## Skills：配置 → 注入

1. 配置（Settings → Skills）  
扫描目录里的 `SKILL.md`，登记到 Sidecar 的 skills registry（启用/挂载路径）。
	
2. 绑定（可选）  
Agent 上的 `skills[]` = 强制带上这些 key；没勾选时，已启用的 global / 项目 / 自定义也会进目录（项目同名优先）。
	
3. 运行时注入（`compose_agent_system_prompt`）
	
	- 拼一层 catalog：名字 + 一句话，不把全文塞进 prompt
		
	- 模型觉得相关 → 调内置工具 `read_skill(key)` → 再读 `SKILL.md` 正文

所以 Skills 是 提示词目录 + 按需加载，不是一上来把所有手册灌进去。


## MCP：配置 → 注入

1. Hub 里已登记启动命令  「启动命令写在磁盘 JSON 里，用的时候 `load_servers()` 读文件」
	
2. 本轮 Chat 绑定了这个 MCP → 先起子进程
	
3. 用 stdin/stdout 问：你有哪些工具？（list_tools）
	
4. 把工具列表交给 LLM
	
5. LLM 选出要调的工具和参数
	
6. 再通过 JSON-RPC （用 JSON 发「请你做某件事」和「结果是什么」 的约定）让进程执行
	
7. 结果返回给 LLM，可能再调别的工具，直到给出最终回复

```
resolve_agent_mcp_servers(agent)
    ↓
run_mcp_react_loop
    ↓
起 MCP 子进程 → list_tools
    ↓
LLM 选工具 → 执行（高风险会暂停等人批）
    ↓
结果回灌模型，循环直到给出最终回复
```

工具轨迹经 WS 推到 Chat 步骤条。