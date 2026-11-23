---
created: 2026-08-12T10:17
---


---

> [!NOTE]
> 算是一个LangGraph最小教学骨架的组件，不是生产组件，只负责演示「状态 -> 节点 -> 边 -> 跑一遍」的链路过程

用来验证LangGraph 依赖能跑通、节点能改状态。不接 Chat、不接 CLI、不接 WebSocket。

invoke === 把图从头跑一遍，并返回最终状态

完整链路如下：

	先定义好Graph状态结构，模拟状态改变的函数，搭建最小Graph函数，执行Graph并Invoke最终结果