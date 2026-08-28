
---

> [!NOTE] 前置知识
> LangGraph 的节点函数约定是：`state → 新 state` ，也就是只允许 `state到state` 的数据流转，但对应的 handler 还需要知道「当前是哪个节点」和「整张 workflow」，这两个参数执行的时候是固定的因此需要一个闭包函数利用闭包绑定这两个参数

## 完整链路

- 先通过包装函数 `_wrap_handler` 把「三件套参数（node、workflow、state）」收成「只收 state」的 `_run` 
	
- LangGraph 只认这个 `_run`，轮到哪个节点就调哪个节点的 `_run`

### `_run`的 state 状态流转

> state也就是CompilerState

- `invoke(初始 state)` 时，把第一份 state 交给 LangGraph
	
- LangGraph 调当前节点，将state透传给闭包函数`_run`
	
- 用返回的新 state 继续往下走（下一个节点再透传给 `_run`）


> [!NOTE] 全阶段逻辑
> 完整关系是用户在节点选择对应的五种type，然后通过映射将type映射到对应的函数handler中，工作流编译时，通过包装函数将参数封装到闭包函数 `_run` 上，真正LangGraph执行时，运行到对应节点时调用_run，根据handler的不同执行不同的逻辑，完成整个链路的流动

### 状态改变的驱动方式

LangGraph 调条件边的 `_route` → 读 `check_result` → 对照边上的 `when`（`passed` / `failed`）→ 走到对应目标

## check节点单独说明

> check 节点有两种角色分工，分别是用户配置和自动执行

1. 用户配置：在画布上放一个 `check` 节点，并写好要验什么（自然语言 `prompt`，或规则如文件是否存在等）。
	
2. 自动执行：工作流跑到这个节点时，`_handle_check` 自动按你配的规则判 `passed` / `failed`，再走后面的条件边。

### 两种check是否会冲突？

不存在并行同时跑，存在优先级：

1. 节点配了 `data.prompt` → 走 LLM 自然语言判断
	
2. 否则 → 走 `evaluator` 规则（文件存在等）

最后都收束成一个字段`check_result`，后面分支只认这个字段



