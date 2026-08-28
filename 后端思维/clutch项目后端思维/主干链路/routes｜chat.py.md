
---

> [!NOTE]
> 是整个项目 HTTP + Websocket的门面：收请求、校验参数、返回结果。是main.py里「点发送/开跑/看进度」的核心路由包

	app.include_router(chat_router)

真正跑对话、推state_patch、开工作流的任务大多在 `chat_runner.py` 里

## 在项目中的链路

```
前端 (Chat / Flow)
    │  HTTP 短操作 / WS 长连接
    ▼
routes/chat.py     ← 你现在看的：路由 + Pydantic 入参
    │  import / 委托
    ▼
chat_runner.py     ← 大脑：plain chat、workflow、WS 消息循环
    │
    ▼
compiler / engine_router / state …
```

## 组件导读
### 先引用基础组件和业务需要的业务组件

### 定义需要参数的请求体模型，规范接口参数

[[文件回滚部署]]

### 四大路由逻辑

> 核心理念：只做接入层，业务逻辑下沉到别的组件，易维护


#### ① 模块：工作流 Workflow CRUD 接口（配置管理）

	POST /api/workflows/validate
	GET /api/workflows/templates
	GET /api/workflows/templates/{workflow_id}
	GET /api/workflows/user
	GET /api/workflows/user/{workflow_id}
	POST /api/workflows/user
	DELETE /api/workflows/user/{workflow_id}

> 前端画布编辑 → 实时调用 validate 校验 → 确认无误 POST 保存到用户目录 → 开跑时传入`workflow_id`加载配置启动 LangGraph 图。


#### ② 模块：侧栏历史会话 + Shell 运行快照（上下文持久化）

##### 会话接口

	GET /api/runs/history
	POST /api/sessions

##### Shell 上下文快照接口

	GET /api/shell-snapshots
	GET /api/shell-snapshots/{run_id}
	PUT /api/shell-snapshots/{run_id}

> 用户关闭浏览器重进页面 → 拉取 history 展示历史会话 → 选中会话加载 shell 快照恢复终端现场。


#### ③ 模块：人工闸门、启动运行、状态查询（系统核心调度入口）

	POST /api/runs/start
	POST /api/runs/{run_id}/start
	POST /api/runs/{run_id}/human-decision ：人工审批闸门
	POST /api/runs/{run_id}/reassign ：重分配
	GET /api/runs/{run_id}/state ：状态查询
	GET /api/runs/{run_id}/debug ：Debug调试接口

##### 启动标准时序

1. 前端选择 workflow\_id + 输入任务指令 → `POST /api/runs/start`拿到 run\_id；
	
2. 前端建立 WebSocket 连接 `/ws/runs/{run_id}` 订阅实时推送；
	
3. LangGraph 执行途中命中人工节点 → WS 推送`human_required`事件，前端弹出审批框；
	
4. 用户点击同意 → 调用`human-decision`放行，引擎继续执行。


#### ④ 模块：运行干预：停止、回滚文件、会话分叉、上下文压缩、删除运行

	POST /api/runs/{run_id}/stop ：终止运行
	POST /api/runs/{run_id}/rewind ：文件回滚
	POST /api/runs/{run_id}/fork ：会话分叉Fork
	POST /api/runs/{run_id}/compact ：上下文手动压缩
	DELETE /api/runs/{run_id} ：彻底删除运行

#### ⑤ 模块：WebSocket 长连接 `/ws/runs/{run_id}`

> 此组件中仅作为接入层

实际逻辑在`src.main.ws_run` ，主要职责如下：

- 前端建立 ws 连接绑定 run_id；
	
- 后端订阅该 run_id 的全局状态变更、日志流、事件推送；

也就是说，前端run_id跟后端绑定后，活状态事件的所有状态修改变化均通过WS订阅的事件完成两端的通信；开跑/停止/回滚/拉全量等扔走HTTP，实现双端通信。