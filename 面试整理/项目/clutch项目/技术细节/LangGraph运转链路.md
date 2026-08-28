
---

> [!NOTE] 文件说明
> 该组件主要说明了前后端各个关键逻辑的状态流转，直观表现LangGraph框架是怎么运行的

## 前端

选择对应节点的 type ，然后用户进行连线串联链路（条件边、非条件边）

## 后端

### 编译期

#### 常规编译链路

- 将每个节点包装成 `_run` 闭包函数，通过配置表将 type 和具体的执行函数 handler 完成映射
	
- 将每条边编译成 直连的无条件边 和 `(_route + path_map「when=passed/failed」)`的条件边，

#### 带 human_gate 的编译链路

将边编译成 `(_route + path_map「when=approve/reject/retry」)` ，然后将该节点注册到 `interrupt_before` 中

### 执行期

> LangGraph 只做「调 `_run` → 看边 / 问 `_route` → 去下一 id」循环。每个有条件边的节点都会编译`(_route + path_map)`

#### 常规执行链路

常规执行链路走 `_handle_check`  handler 处理逻辑 `_route` 根据 `check_result`  字段返回 passed / failed ，然后匹配 `path_map` 对应的哪个节点执行逻辑

	handler 改 state；
	条件边靠 state 里的 check_result 等出信号。根据`_handle_check` 的逻辑判断从而改变 check_result 的结果

#### 带 human_gate 的执行链路

到该节点时通过WS事件等待用户决策，决策完后将 `human_decision` 合并到快照中，再通过 resume 进行续跑，此时由`_handle_human_gate` handler 处理逻辑，`_route` 读 `human_decision` → `path_map` 找到下一站


#### `_handle_check` 的逻辑判断分支

1. state 里已有 preset（测试/resume 注入）→ 直接用
	
2. 节点配了 `data.prompt` → 拿上游产出问 LLM，判 passed / failed
	
3. 否则 → 用 `data` 里的规则走 `evaluator`（如文件是否存在等）
	
4. 没工作区又走规则时 → 默认 `passed`


## 无条件边跟条件边的逻辑差异区别

> 具体逻辑在 `WorkflowCompiler.compile()` 依靠 LangGraph 提供的 API 实现， 把边登记进 `StateGraph` 「LangGraph自带的类」

### 具体逻辑

编译时先扫 `Workflow JSON` 的 edges 字段，没有 when 的用 `add_edge()` 函数处理，有 when 的用 `add_conditional_edges()` 函数处理，登记好后执行期 LangGraph 按登记好的边走，条件边才会多问一次 `_route`。

