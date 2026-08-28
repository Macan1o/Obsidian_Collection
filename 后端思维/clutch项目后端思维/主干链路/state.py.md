
---

> [!NOTE]
> 主要负责clutchState等完整的数据结构以及一些读取更新逻辑


### `from __future__ import annotations`

开启**延迟解析类型注解**，在加载阶段先把**注解当成纯字符串存起来**，等到整个文件全部加载完毕、所有类、变量都定义完成之后，再统一解析这些类型。

	再更高版本的解释器中已经自动引入了这一特性，可以不用引用

```Python
class Task:
    def set_next(self, other: Task): # 解析这行注解时 Task 类还没定义完，找不到名字
        pass

```

因为Python从上到下逐行解析代码：读到 `other: Task` 时，`class Task` 还没加载完毕，解释器不认识 `Task`。


---

### `from collections.abc import Mapping`

> 这里需要理清一个概念，传参写dict限定必须是正经可变字典，支持随便增删改；传参写Mapping，相当于类型层面的抽象接口。类型工具会默认你只做读取操作，修改时类型检查器会告警。但因此它也兼容各种各样的键值结构，适用范围更广

- 写工具函数，只读取数据，不改里面内容时用Mapping
	
- 函数内部需要新增、删除、修改键值用传统的dict

#### 能被判定为`Mapping` 的东西

原生 `dict`、封装后的只读字典、web 请求解析出来的参数对象



---

### `TypedDict`

对标 TS 的 `interface / type object` ，对Python的原生对象字面量类型约束，给**普通字典**套上结构约束。例如

```Python
# 定义字典结构，对标 TS interface
class RunInfo(TypedDict):
    run_id: str
    status: ClutchRunStatus
    progress: float
    error_msg: NotRequired[str]  # NotRequired = TS 的 ? 可选字段
```



---

### `ClutchState`

> 一次run的全部投影字段，从这读取数据


### `read_cli_session_id / read_cli_session_agent_id`

- chat_runner 开跑前去除跟当前选中Agent对比，若不一致则强制开新CLI会话，避免跨Agent续聊串上下文
	
- Flow节点的 agent_executor 同样校验匹配才允许resume，不匹配就当没有session


### `cli_session_patch`

> CLI session指的是本地CLI自己记的多轮上下文

- 本轮CLI成功返回session 打进final_patch 落盘 + WS的 state_patch，下次继续续
	
- Agent切换且本轮没新session就清除旧绑定
	
- flow里 `_persist_flow_cli_session` 用它把session写回 run state

#### 联动链路

`cli_session_patch`将改变的键值打包成一个补丁，落款到`final_patch`集合中。最后通过state_patch事件推送到前端。前端读state `clutchStore`一变，订阅的组件就会按新的 snapshot 快照渲染一遍

	final_patch数据结构

```Python
final_patch = {
  "messages": [...],
  "status": "idle",
  # ... 其它字段
}
final_patch.update(cli_session_patch("sess-9", "claude-1"))
# final_patch 里现在多了 cli_session_id / cli_session_agent_id
```

- patch 里有的键：新值盖掉
	
- patch 里没有的键：保持不动





