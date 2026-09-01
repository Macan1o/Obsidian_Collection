
---

> [!NOTE] 前置知识
> LangGraph 的节点函数约定是：`state → 新 state` ，也就是只允许 `state到state` 的数据流转，但对应的 handler 还需要知道「当前是哪个节点」和「整张 workflow」，这两个参数执行的时候是固定的因此需要一个闭包函数利用闭包绑定这两个参数

> [!NOTE] 文件说明
> 该组件主要说明了前后端各个关键逻辑的状态流转，直观表现 LangGraph 框架是怎么运行的

## 混淆点

### 关于 `_route`、 `_run`、 `route_next` 三者之间的对应关系

`_run` 决定「节点怎么改 state」，`_route` 基于条件边的一次封装决定「下一步走哪条边」。`route_next` 是_route 闭包下的底层执行函数，负责根据信号路由对 `path_map` 中的映射获得下一个运行节点，从而使LangGraph 执行正确的节点。


---

## 完整链路

前端选择对应节点的 type ，然后用户进行连线串联链路（条件边、非条件边）

### 编译期

#### 常规编译链路

- 将每个节点包装成 `_run` 闭包函数，通过配置表将 type 和具体的执行函数 handler 完成映射
	
- 将每条边编译成 <mark style="background:#d3f8b6">直连的无条件边</mark> 和 `(_route + path_map「when=passed/failed」)`的<mark style="background:#d3f8b6">条件边</mark>，

#### 带 human_gate 的编译链路

将边编译成 `(_route + path_map「when=approve/reject/retry」)` ，然后将该节点注册到 `interrupt_before` 中

### 执行期

> [!NOTE] 核心原则
> LangGraph 只做「调 `_run` → 看边 / 问 `_route` → 去下一 id」循环。每个有条件边的节点都会编译`(_route + path_map)`

#### 常规执行链路

常规执行链路走 `_handle_check`  handler 处理逻辑 `_route` 根据 `check_result`  字段返回 passed / failed ，然后匹配 `path_map` 对应的哪个节点执行逻辑

#### 带 human_gate 的执行链路

到该节点时通过 WS 事件等待用户决策，决策完后将 `human_decision` 合并到快照中，再通过 resume 进行续跑，此时由`_handle_human_gate` handler 处理逻辑，`_route` 读 `human_decision` → `path_map` 找到下一站


#### `_handle_check` 的逻辑判断分支

1. state 里已有 preset（测试/resume 注入）→ 直接用
	
2. 节点配了 `data.prompt` → 拿上游产出问 LLM，判 passed / failed
	
3. 否则 → 用 `data` 里的规则走 `evaluator`（如文件是否存在等）
	
4. 没工作区又走规则时 → 默认 `passed`


## 无条件边跟条件边的逻辑差异区别

> [!NOTE] 实现原理
> 在 `WorkflowCompiler.compile()` 依靠 LangGraph 提供的 API 实现， 把边登记进 `StateGraph` 「LangGraph自带的类」

### 具体逻辑

编译时先扫 `Workflow JSON` 的 edges 字段，没有 when 的用 `add_edge()` 函数处理，有 when 的用 `add_conditional_edges()` 函数处理，登记好后执行期 LangGraph 按登记好的边走，条件边才会多问一次 `_route`。

