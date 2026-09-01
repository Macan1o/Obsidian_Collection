
---

> [!NOTE] 
> 前后端建立联系后，通过特定事件通信来完成字段更新、追加数据等副作用事件

## LangGraph 运行的 WS 事件
### 链路图

![[Pasted image 20260831190554.png|822]]

### 链路节点说明

- **①**连接：前端切换会话时， 建 `new WebSocket()`，后端验 token 后，建立**一个 run 一条专用连接**
	
- **② 触发运行**：用户发消息，，后端 WS 主循环收到后启动 workflow，LangGraph 开始跑。
	
- **③④ 挂起并推送**：挂起后，返回`status="awaiting_human"` ，后端连发两个事件 `state_patch` 和 `human_required` 
	
- **⑤⑥ 前端弹窗（三层联动）**：clutchState.ts 的 `ws.onmessage` 解析 envelope ——`state_patch` 走 `applyPatch(data.patch)`，`human_required` 也强制 `applyPatch({status: 'awaiting_human'})`；`applyPatch` 改完状态后 `emit()` 通知订阅组件，完成渲染
	
- **⑦ 用户决策回传**：点 Allow → `App.tsx handleApprove` → `clutchStore.send({action: "human_decision", decision: "approve"})` （reject，retry 还带 `instructions`）→ 还是通过**同一条 WS** 发回。
	
- **⑧ 后端恢复**：WS 主循环 `receive_text()` 收到（`action == "human_decision"`）走`asyncio.to_thread(_apply_human_decision)` （锁去重 → `resume_workflow` → `update_state` + `Command(resume)`）→ 图从断点续跑。
	
- **⑨ 循环往复**：恢复后继续推 `state_patch`（status 变回 running，前端审批条自动收起）、`log`、`message`，终态时补发 `run_completed`。如果 retry 后又挂在同一个 gate，就再来一轮 ④


```tandem-comments
// Schema: { "<id>": { anchor:{exact,prefix,suffix,pos?}, status:open|resolved, thread:[{author,ts,text}], suggestion?:{replacement,author,ts,result?} } }
// Anchor = quote from the prose. To locate: search for "exact", disambiguate via prefix/suffix.
{
  "0a2c": {
    "anchor": {
      "exact": "两个事件",
      "pos": 277,
      "prefix": "aiting_human\"` ，后端连发",
      "suffix": " `state_patch` 和 `hu"
    },
    "status": "open",
    "thread": [
      {
        "author": "macando",
        "ts": "2026-08-31T11:27:19.814Z",
        "text": "state_patch ：负责更新 state 状态，因为最终维护的是唯一 clucthState 真相，需要更新后渲染"
      },
      {
        "author": "macando",
        "ts": "2026-08-31T11:27:24.078Z",
        "text": "human_required：负责通过 applyPatch 修改状态，前端根据状态来渲染出UI弹窗"
      }
    ]
  }
}
```
