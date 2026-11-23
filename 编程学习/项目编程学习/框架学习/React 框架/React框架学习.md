---
created: 2026-08-12T10:17
---

#React

---
# React数据驱动与第三方库数据驱动差异

### React数据驱动「`useState`」

每个组件 `Fiber` 内部维护一条 **Hook 链表**，每个 `useState` 对应一个 Hook 对象，调用 `setXxx` → 往当前 Hook 的 queue 追加更新任务，将当前组件 Fiber 标记为待更新，调度器调度后，重新执行整个组件函数，遍历 Hook 链表计算最新 `state`

### 外部仓库订阅

这部分走订阅发布体系，`useSyncExternalStore` 把 React 的更新回调`listener`注册进Set仓库，修改数据后「`applyPatch`」通过手动触发`emit` 「遍历 Set **批量执行所有 listener**」，通过调用官方API `useSyncExternalStore` 负责新旧快照对比从而触发组件重渲染，实现外部仓库的数据驱动更新。

`useSyncExternalStore`使用逻辑 「内置对比逻辑，触发重渲染」

	useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)

1. `subscribe`：注入仓库订阅逻辑，组件挂载时注册 listener，卸载自动取消订阅；
2. `getSnapshot`：每次收到 listener 通知后，执行一次，拿到仓库最新数据切片；


---


