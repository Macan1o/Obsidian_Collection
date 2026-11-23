---
created: 2026-08-12T10:17
---

#React原理

---
## 虚拟DOM相关问题

> 是基于真实DOM的一种抽象，作用在减少真实DOM的操作成本，提升性能。**缺点**就是内存需要额外维护虚拟DOM树

### 核心优势

**批量更新**：多次状态变化合并成一次DOM更新

**最小化重排重绘：**通过Diff算法计算差异

**跨平台**：同一套Virtual DOM可渲染到Web、Native

## React渲染原理

> React渲染分为**调和**（Reconciliation）和**提交**（Commit）两大阶段

### 初次渲染流程

执行组件函数 -> 返回JSX -> 转换成虚拟DOM -> React创建Fiber树（每个虚拟DOM对应一个Fiber） -> 调用 `ReactDom.render()` 将Fiber树挂载到真实DOM上

### 更新渲染流程

`setState`或`useState`更新 -> 触发更新任务

> 调和阶段

比较新旧Fiber树「Diff算法」 -> 生成EffectList「需要更新的节点」

> 提交阶段

执行**副作用**「DOM更新、生命周期回调」

Fiber架构

---

## Hooks原理

> 解决了 **函数组件无状态** 和 **逻辑复用困难** 的问题

## Diff算法

React 的 `Reconciliation`（调和）算法通过三个假设将 **O(n³)** 的树形比较降至 **O(n)**

### 基本原则

1. **Tree Diff（层级比较）：**只比较同一层级的节点，不跨层级移动。跨层级的操作（如把某节点从一个父节点移到另一个）会直接销毁重建，而非移动。
    
2. **Component Diff（组件比较）：**同类型组件才会继续向下比较，不同类型直接销毁旧树、创建新树。
    
3. **Element Diff（元素比较）：**同一层级的多个子元素，通过 `key` 来识别节点身份，避免重复创建销毁。


---

## 为什么引入Fiber

Fiber 是 React 内部一种新的数据结构（链表结构），每个 `Fiber` 节点对应一个组件/DOM 元素，包含 `child`、`sibling`、`return` 指针，形成可遍历的工作单元。

**解决同步渲染长任务堵塞，支持可中断、优先级调度**

### 解决的核心问题

- **可中断与恢复：**把渲染拆成一个个「工作单元」，利用 `requestIdleCallback`（Scheduler 包模拟）在浏览器空闲时执行，高优先级任务（如用户输入）可打断低优先级渲染。
    
- **优先级调度：**通过 Lane 模型（React 18）区分任务优先级，紧急更新优先渲染。
    
- **双缓冲机制：**同时维护两棵 Fiber 树（current tree + workInProgress tree），渲染完成后一次性切换，避免白屏。

### 工作阶段

- _Render 阶段（_**_可中断_**_）_：构建 `workInProgress Fiber` 树，执行 Diff 算法，收集副作用（effectList）。
    
- _Commit 阶段（_**_不可中断_**_）_：将副作用同步应用到真实 DOM，执行 `useEffect` / 生命周期等。


---

## Hooks为什么不能写在条件语句里

Hooks依赖固定调用顺序

## SetState是同步还是异步

在 React 控制的事件处理器、生命周期中：**_异步_**（批量合并更新）。

在 `setTimeout`、`Promise`、原生事件中：**_同步_**（立即触发重渲染，每次 `setState` 都渲染一次）。


---

## React Hooks的规则是什么？ 底层原理是什么

> React 通过调用顺序来匹配 Hook 与其对应的状态。每次渲染时 Hook 依次按顺序存储在一个链表（或数组）中，用索引来关联状态。

**两条规则：**

只在函数组件顶层调用 Hooks，不能在条件语句、循环、嵌套函数中。

只在React _函数组件_或_自定义_ Hook 中调用 Hooks。

> **如果在条件语句中调用 Hook，渲染时 cursor 的对应关系会错乱，导致状态张冠李戴**

```
// ❌ 第一次渲染：Hook顺序 A -> B -> C
// ❌ 条件为 false 后：Hook顺序 B -> C，B 拿到了 A 的状态！
if (condition) useState('A'); // Hook A
useState('B');               // Hook B
useState('C');               // Hook C
```


---

## 如何实现一个自定义的Hook？ 举例说明useFetch 和 useDebounce

自定义 Hook 是一个以 use 开头的函数，内部可以调用其他 Hook，用来封装和复用有状态逻辑。

类似于一个函数，封装和复用有状态的逻辑