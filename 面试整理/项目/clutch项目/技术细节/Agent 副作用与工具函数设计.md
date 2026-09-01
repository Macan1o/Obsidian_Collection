
---

> [!NOTE] 策略原则
> 副作用必须通过工具，我采用的方法是 少量原语覆盖大多数副作用，其余用 Shell / MCP 扩展。

> 一套原语 + 可选外挂MCP

|能力|工具|覆盖面|
|---|---|---|
|读|`read_file` / `list_dir` / `grep`|观察，无副作用|
|改文件|`search_replace` + `apply_patch`|增删改移，绝大多数代码编辑|
|跑命令|`run_terminal_cmd`（可 background）|构建、测试、脚本——万能出口|
|Git|`git_status` / `git_diff` / `git_commit`|版本控制|
|交互/计划|`propose_plan` / `ask_user_question` / `todo_write`|HITL、任务拆解|


## 基本原则

### 参数化，而不是工具分裂

同一个 `apply_patch`，靠 patch 内容表达 Add/Update/Delete；不必 `create_file` / `delete_file` / `rename_file` 各一个。

### 横切能力做中间件，不做成工具

- 写前快照 / 回滚
	
- 风险审批（HITL）
	
- Diff 卡 / Changes  
     这些挂在工具实现或网关里，不增加 Agent 可见的 tool 数量。

### 扩展用 MCP，而不是改核心表


### Shell 当兜底

> 统一收束，只给Agent 一个叫 `run_terminal_cmd` 的工具。完成下列闭环

Agent 决定「要跑哪条命令」→ 调用 `run_terminal_cmd`，把命令字符串传进去 → 这个工具在工作区的终端里真正执行，再把输出返回给 Agent。

#### 项目联系

Agent 只传 `command` 字符串；真正执行是 `popen_shell` / `subprocess.run`（`shell=True`）。



---

> 代码分析逻辑在 Python_Study/run_terminal_cmd.py中