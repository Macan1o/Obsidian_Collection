---
created: 2026-08-19T14:55
---


---
## 1. `from __future__ import annotations` — 推迟求值类型注解

> [!NOTE] 前置知识
> Python 原生类型说明不是注释！注解是真实存在的对象，会被存进函数的属性 `__annotations__`，运行期会执行表达式计算。

打印后结果如下：

```Python
def fn() -> int | None:
    pass

print(fn.__annotations__)
# {'return': int | None}
```

因此如果不引入这段的话

```Python
class A:
    def func(self) -> A:  # 运行到这一行，A类还没完全定义完！
        return self
```

**遇到类型注解，运行时立刻执行、求值这个注解**。解析`-> A`的时候，A 还没构造完成，Python 要马上计算`A`，找不到。


## 2. 现代类型：`X | Y` 和 `dict[str, T]`

Python 3.10+ 用 `|` 表示「或」；`list`/`dict` 本身就能当泛型，不必再写 `Optional[str]`、`Dict[str, Any]`。让类型注解更加灵活，常用在 <mark style="background:#d3f8b6">函数返回值、可选字段、JSON 形状</mark>


## 3. `Literal[...]` — 把字符串收成枚举的轻量版

> 限定某个值只能是列出的那几个字符串。类型检查器会拦拼写错误；IDE 能补全。

- 对外 API、JSON 字段、http 参数，值就是字符串本身 → 优先用 Literal 「<mark style="background:#d3f8b6">纯静态检查</mark>」
	
- 需要在代码内部大量遍历、比较、业务逻辑绑定 → 直接继承 StrEnum 类「<mark style="background:#d3f8b6">运行有实体，可以做运行逻辑</mark>」

### 两者对比

```Python
from typing import Any, Callable, Literal, Protocol
ProviderId = Literal[
    "deepseek", "openai", "anthropic", "google", "ollama", "agnes", "opencode", "custom"
]
ModelKind = Literal["chat", "image", "video"]
```

```Python
from enum import StrEnum

class ProviderId(StrEnum):
    DEEPSEEK = "deepseek"
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    GOOGLE = "google"
    OLLAMA = "ollama"
    AGNES = "agnes"
    OPENCODE = "opencode"
    CUSTOM = "custom"


# ✅ 直接循环枚举，拿到全部服务商原始字符串
all_providers = [item.value for item in ProviderId]
print(all_providers)
# ['deepseek', 'openai', 'anthropic', 'google', 'ollama', 'agnes', 'opencode', 'custom']
```


## 4. `TypedDict` + `NotRequired` — 给「就是个 dict」立契约

> [!NOTE]
> dict 仍是 dict（方便 JSON / LangGraph / WebSocket），但字段名和类型被钉死。`NotRequired` 表示「可以不出现」，对应前端「没有就不渲染」。

```Python
class ClutchState(TypedDict):
    """一次 run 的全部可投影字段。前端 Chat / Flow / Terminal 都吃这里的数据。"""

    run_id: str  # 唯一运行 ID，例如 run_a1b2c3d4；WS 也按它订阅
    workflow_id: str  # 绑定的工作流；单 Agent 纯聊天时可为空字符串
    # ...
    status: ClutchRunStatus  # 上面那几种生命周期
    messages: list[dict[str, object]]  # Chat 气泡列表
    # --- 可选：Agent 活动条 / 工具步骤（没有就不出现）---
    pending_tool_steps: NotRequired[list[dict[str, object]]]  # 正在或刚执行的工具步骤
    live_reasoning: NotRequired[str]  # 流式「思考」片段
```


## 5. `Protocol` + 仅关键字参数 `*` — 鸭子类型的接口

着重整理


## 6. `TypeVar` — 让「两条路返回同一类型」

`T` 是占位符：调用方传入什么，返回就是什么。当某些函数返回啥类型是调用方决定的，写函数的时候不清楚。有可能返回 `str`，有可能返回 `dict`，有可能返回自定义对象。为了避免写死返回`Any` 所以引入TypeVar

```Python
T = TypeVar("T")  # 定义一个类型占位符T

hybrid_route: Callable[[], T]
# hybrid_route 是一个函数，无参数，返回类型是 T

legacy_route: Callable[[], T]
# legacy_route 也必须返回同一个T类型

-> T
# 本函数最终返回的，也是这个T类型
```

#### 出处

```Python
T = TypeVar("T")


def try_shell_exec_hybrid(
    *,
    agent_type: str,
    source: str,
    run_id: str | None,
    workspace_path: str | None,
    provider_spec: ProviderSpec | None,
    hybrid_route: Callable[[], T],
    legacy_route: Callable[[], T],
    logs: list[str],
    on_log: Callable[[str], None] | None,
    emit_log: Callable[[list[str], Callable[[str], None] | None, str], None],
) -> T:
```


IDE 记住这个类型，后续继续补全、做类型检查。



