
---
## 两种HITL链路对比

> [!NOTE] 类比
> 图级像高速公路收费站（整条路停住）；工具级像车间里某台机器要签字才能开机（环内暂停）。

## 图级 HITL（工作流闸门）

```
画布 human_gate
    → Compiler: interrupt_before=[所有 gate id]
    → begin_workflow / invoke 跑到闸门前停下
    → snapshot.next ∈ gate_ids → 改变状态 status=awaiting_human
    → WS: state_patch + human_required
    → 用户 Allow/Reject/Retry
    → _apply_human_decision → resume_workflow
    → update_state(human_decision) + Command(resume)
    → 闸门 handler 放行 → route_next 按 decision 走边
    → 用完清掉 human_decision（防下一闸门自动通过）
```

### 基本流程

> [[状态数据结构解析]] 中的「挂起时的完整快照」

先经过编译器组件，将需要暂停的节点注册，然后`invoke` 跑到闸门之前被框架挂起并写闸门 pending ；根据 checkpoint 的 `next` 逻辑判断在等人，就将 status 设置为 awaiting_human ，然后保存快照通过 WS 前端做出决策后端通过 update_state 并到快照中，通过 `Command(resume)` 将最新的快照注入进去，对应的 handler 放行后，路由根据三种选择按 decision 走对应的边，闸门通过后清理 human_decision 防止下一闸门自动通过


## 工具级 HITL（MCP 风险闸门）

```
Chat 调 Clutch Agent / MCP ReAct
    → 模型发出 tool_call
    → resolve_tool_gate（用户规则 allow/ask/deny）
    → is_risky_mcp_tool（token 表）+ permission_mode
    → 需审批：不执行工具，return approval_required
    → chat_runner: store_pending(McpPendingApproval)
    → status=awaiting_human + human_required
    → 用户 Allow / Reject / Retry(revise)
    → WS: 若 get_pending 且无 workflow_id → _handle_plain_chat_mcp_decision
    → approve: record_mcp_approval + 执行该工具 + 带着 chat_messages 继续 ReAct
    → reject: 给模型 tool 错误/拒绝文案，再继续或收尾
```

### 基本流程

- 先在工具真正副作用前做风险判定（规则 deny/ask/allow + 危险命令 + risky token + 权限模式）；
	
- 命中需审批则中断当前 ReAct 步、不执行工具，将挂起上下文存入 `mcp_pending`，`status=awaiting_human` 并写明 pending 类型然后经 WS 投影；
	
- 前端决策后合并进 pending 续跑路径（执行工具或写入拒绝/修订反馈），再进入下一轮 ReAct；
	
- 用完即 `pop_pending`，approve 还会记 approval key，防止下一相同调用自动再拦或重复打扰。

#### 续跑逻辑归纳

后端发送 工具清单 / Schema 给模型，然后模型根据函数的功能和描述返回 tool_calls 此时后端进行预处理，先校验如果有高危操作则 return ，存入上下文到 mcp_pending 中，然后推送 WS 请求到前端等待审批，前端审批点击对应的按钮后，发送 HTTP 审批 / resume请求到后端路由函数，后端执行该工具，把结果 append 到上下文中，一并发给模型，循环上述逻辑直到最终回答

	执行工具函数逻辑统一收束在 _execute_tool_call 中



## Check 节点的 handler 判断逻辑

### 核心逻辑

```
1. state 里已有 check_result（预设 / 测试 / resume 注入）→ 直接用
2. 节点 data.prompt 非空 → 调 LLM judge
3. 否则 → evaluator 规则检查（file_exists / shell / lint）
```


## 三种方式归纳


> [!NOTE]
> 三种暂停机制剖析

- <mark style="background:#d3f8b6">图级</mark>：链路执行层，根据 `interupt_before` 让 LangGraph停下来
	
- <mark style="background:#d3f8b6">工具级</mark> ：逻辑执行层，内部卡住写 pending ， WS 到前端进行审批
	
- <mark style="background:#d3f8b6">check节点</mark>：自动质检节点，根据检查提示词「和上游产物」作自动评判，直接根据 passed / failed 走图级逻辑走条件边 




```tandem-comments
// Schema: { "<id>": { anchor:{exact,prefix,suffix,pos?}, status:open|resolved, thread:[{author,ts,text}], suggestion?:{replacement,author,ts,result?} } }
// Anchor = quote from the prose. To locate: search for "exact", disambiguate via prefix/suffix.
{
  "cbf8": {
    "anchor": {
      "exact": "写明 pending 类型",
      "pos": 1517,
      "prefix": "us=awaiting_human` 并",
      "suffix": "然后经 WS 投影；\n\t\n- 前端决策后"
    },
    "status": "open",
    "thread": [
      {
        "author": "macando",
        "ts": "2026-09-01T06:33:30.514Z",
        "text": "这个跟图级HITL的pending有本质对比"
      },
      {
        "author": "macando",
        "ts": "2026-09-01T06:35:26.798Z",
        "text": "图级 HITL\n\n是 LangGraph MemorySaver checkpoint 进行存储的。pending 是snapshot.next 的待执行节点，主要存下一步跑哪个节点。\n\n通过interrupt_before 框架打断 ；update_state + Command(resume)恢复"
      },
      {
        "author": "macando",
        "ts": "2026-09-01T06:37:24.098Z",
        "text": "工具级 HITL\n\n是模块级 dict 内存。pending 是业务对象，主要存恢复信息：哪次 tool call、参数、对话历史…\n\n通过函数逻辑检测待审批主动 return；通过新开一轮 _llm_chat_reply，注入 mcp_resume后从断点恢复"
      }
    ]
  }
}
```
