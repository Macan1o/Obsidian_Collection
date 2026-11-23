---
created: 2026-09-03T09:40
---


---

> [!NOTE] 代码溯源
> 代码在 compiler.py


## 节点闭包函数
### 基本逻辑

```Python
def _wrap_handler(self, node: dict[str, Any]) -> Callable[[CompilerState], CompilerState]:
    node_type = node["type"]
    handler = self.NODE_HANDLERS.get(node_type)
    if handler is None:
        raise ValueError(f"No handler registered for node type: {node_type}")
    workflow = self._workflow

    def _run(state: CompilerState) -> CompilerState:
        return handler(state, node, workflow)

    return _run
```


## 边闭包函数

### 基本逻辑

> 需要 workflow 、当前节点id 等数据

```Python
def _route(
    state: CompilerState,
    _workflow: dict[str, Any] = workflow,
    _source: str = source,
    _path_map: dict[str, str] = path_map,
) -> str:
    branch, _method = route_next(_workflow, _source, state)
    if branch not in _path_map:
        raise ValueError(...)
    return branch

graph.add_conditional_edges(source, _route, path_map)
```

```
节点 handler 跑完，写出 state
  （如 check_result="failed" / human_decision="approve"）
        │
        ▼
_route(state)                     # LangGraph 只传入 state
        │
        ▼
route_next(workflow, source, state)
  → routing_signal：按节点类型从 state 取分支键
        │
        ▼
返回 "failed" / "approve" / ...
        │
        ▼
LangGraph 用 path_map["failed"] → 下一个节点 id
```