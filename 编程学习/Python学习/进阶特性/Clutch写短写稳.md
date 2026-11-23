---
created: 2026-08-19T15:08
---


---
## 13. 海象运算符 `:=` — 赋值同时当表达式

作用： 先赋值再判断真假，少写一行临时变量。

场景： 环境变量、可选查找、正则匹配后立刻用。

```Python
        if (v := os.getenv("VIDEO_RESOLUTION")):
            cfg.resolution = v  # type: ignore[assignment]
        if (v := os.getenv("VIDEO_CODEC")):
            cfg.codec = v  # type: ignore[assignment]
        if (v := os.getenv("VIDEO_FORMAT")):
            cfg.output_format = v  # type: ignore[assignment]
```

```Python
    for provider_id in _MANAGED_PROVIDERS:
        if key := get_provider_key(provider_id):
            out[provider_id] = key
```

空字符串是 falsy，所以 `if key := get_...` 会跳过没配的 key。这正是想要的。


## 14. 推导式、`**` 解包、`defaultdict`

### 列表推导：过滤 + 变换一行完成

```Python
    def as_system_prompt(self) -> str:
        parts = [layer.content.strip() for layer in self.layers if layer.content.strip()]
        return "\n\n".join(parts)
```

### `**state`：浅拷贝再改几个字段

不可变更新：旧 dict 不动，新 dict 覆盖部分键。LangGraph 节点很爱这种写法。

```Python
def _orchestrator_node(state: GraphState) -> GraphState:
    """示例节点：假装「编排器」跑了一步，改 active_* 和 status。"""
    return {
        **state,
        "active_node_id": "n1",
        "active_agent": "Builder",
        "status": "running",
    }
```

`main.py` 里还有条件解包：生产环境关掉 Swagger。

```Python
app = FastAPI(
    title="Clutch Orchestrator",
    version="1.0.0",
    lifespan=_lifespan,
    **(
        {"docs_url": None, "redoc_url": None, "openapi_url": None}
        if _docs_disabled
        else {}
    ),
)
```

`*nodes_by_id.keys()` 是集合解包：把 dict 的键散进集合字面量。

```Python
            if source not in {"start", *nodes_by_id.keys()}:
                raise ValueError(f"Unknown edge source: {source}")
```

### `defaultdict(list)` — 按 key 分组不必先判断 key 在不在

```Python
        edges_by_source: dict[str, list[dict[str, Any]]] = defaultdict(list)
        for edge in edges:
            edges_by_source[edge["source"]].append(edge)
```

第一次访问 `edges_by_source["n2"]` 自动得到 `[]`。


## 15. `raise ... from` — 异常链

作用： 把底层异常包成业务异常，同时 `__cause__` 指向原始错误。日志里能看到完整因果，调用方只看到「CLI 超时」「base64 非法」。

```Python
        exit_code = proc.wait(timeout=timeout)
    except subprocess.TimeoutExpired as exc:
        proc.kill()
        proc.wait()
        for thread in threads:
            thread.join()
        raise _timeout_error(command, timeout) from exc
```

```Python
    try:
        payload = base64.b64decode(raw_b64, validate=False)
    except Exception as exc:  # noqa: BLE001
        raise WorkspaceError("Invalid image base64 payload") from exc
```

对比：`raise NewError()` 会丢掉原因；`from None` 是故意切断链。本项目几乎总是 `from exc`。

自定义异常也很常见：`class CliAdapterError(RuntimeError)`、`class WorkflowValidationError(ValueError)`——按「调用方该怎么处理」选基类，而不是一律 `Exception`。