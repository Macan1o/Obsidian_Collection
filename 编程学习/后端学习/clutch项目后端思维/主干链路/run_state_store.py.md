
---

> [!NOTE]
> 该组件是一次会话 （run）完整的 clutchState 磁盘持久化层，相当于服务端版的 localStorage：关掉UI、重启Sidecar后，靠它把聊天记录和状态捞回来

也就是根据run_id将每一次对话的所有clutchState状态持久化存储到磁盘中

## 模块职责

因为LangGraph / 后端手里的 clutchState 是SSOT，React只映射投影WS推来的状态。但是进程内存（`_run_states`）会丢

- 刷新页面/重连WS：需要用HTTP拉一份状态，恢复UI
	
- Sidecar重启：内存清空
	
- 会话分叉、压缩、删会话：需要读写/删磁盘的那份状态

## 核心逻辑

### `_coerce_state`

	转换函数，把磁盘读取出来的普通字典，强制转换成合法完整的 ClutchState 状态实例

有个很巧妙的点在于，两步读取，先把必选字段通过`initial_state()`读取后写入，再根据`_PERSISTED_OPTIONAL_KEYS` 结构表读取可选扩展持久化字段，防止报错

### `load_run_state`

	从磁盘加载指定run_id的会话持久化状态

先获取路径，读取后经过`_coerce_state` 转换后返回，所有异常场景统一返回None

### `save_run_state`

	全量覆盖写入：把当前内存中的ClutchState完整序列化存入磁盘JSON

先根据run_id获取到路径，然后写入文件

### `persisted_state_preferred`

	磁盘持久化判定规则

- 磁盘消息条数 > 内存消息条数 → 磁盘数据更全，优先磁盘
	
- 消息数量一致时：磁盘状态是空闲结束(idle)、内存还在运行中(running)，说明上一轮对话已经正常跑完落地磁盘，内存是刚启动的临时运行态，优先磁盘存档

### `sync_run_state_from_disk`

	业务层重启会话时，整合磁盘存档、内存现有状态，根据规则择优选择要使用会话状态

附带清理过期失效shell阻塞脏数据，修正后自动落盘保存

