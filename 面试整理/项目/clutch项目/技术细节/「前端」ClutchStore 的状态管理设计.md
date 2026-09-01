
---

> [!NOTE] 设计原理
> `clutchStore` 是全局单例，用字段/方法管 WS、PTY、轮询等副作用，并维护一份 `state`；组件通过 `useSyncExternalStore` 订阅它；状态更新并 `emit` 后，React 再 `getSnapshot()` 用新 state 渲染 UI。

## 设计模式

```
ClutchStateStore（一个全局单例类）
        │
        │ subscribe / getSnapshot
        ▼
useSyncExternalStore  ← React 官方「订阅外部数据源」钩子
        │
        ▼
各个组件用 useClutchState() 自动重渲染
```

React 组件通常用 `useState` 管自己的 UI 状态。但 Clutch 的「聊天真相」不在组件里，前端只负责投影，而在

- 后端通过 WebSocket 不停推 `state_patch`
	
- 前端还要管连接、PTY 终端、重连、乐观更新



## `useSyncExternalStore` 逻辑链路


```
1. 组件挂载
   useSyncExternalStore → subscribe(React的回调)
   → listeners 里多了一个「React回调」

2. WS 来了，applyPatch 改 state
   → emit()
   → for (listener of listeners) listener()
   → 轮到 React 那个回调被执行

3. React：有变化 → 再 getSnapshot() → 用新 state 渲染 UI
```



## 为什么不用 Zustand / Redux

这些库适合用新对象表达数据怎么变 + UI 订阅。

但 `ClutchState` 还要管很多 命令式副作用，这种更适合放在一个类里当字段/方法

- WebSocket 连上 / 断开 / 收消息
	
- PTY attach、写入、resize、多 lane
	
- 3 秒 HTTP 轮询定时器
	
- 乐观发送（先显示用户消息）
	
- `pendingHydrate` / `reconnectHydrate`

> 自定义类只需要对外只暴露 React 需要的 `subscribe` / `getSnapshot`，内部怎么连 WS 组件不用知道。



## Store的作用

```
                  ┌─────────────────────────┐
  后端 WS 推送 ──▶│     ClutchStateStore      │──▶ React 界面
  用户点发送   ──▶│                           │
  PTY 输出     ──▶│  state + 连接 + 合并逻辑   │
                  └─────────────────────────┘
```

