
---

### 解释一下 SKill 与 MCP 关系

> 我认为 skill 可以拆分成两个领域来看。

- <mark style="background:#d3f8b6">在产品侧</mark> skill 是提示词模板，降低用户写提示词的成本，也就是md文档。
	
- <mark style="background:#d3f8b6">在技术侧</mark>「Agent框架」来说，Skill是一个可复用的功能模块，包括了完整的感知、决策、规划、tool，可以复用到不同Agent。

#### 那既然 Skill 也像 MCP 那样具备工具调用能力且可复用，如何权衡 MCP 和 Skill 的使用？

可以这样来看，当工具涉及外部系统，如浏览器、网站（Stitch、Figma等），相关人员已经开发维护好了，直接用 MCP 即可

当工具涉及到业务逻辑，比如我的项目如何结合实际开发需求落地，是项目核心需要定制的就写在 Skill 里面 <mark style="background:#d3f8b6">「比如我的0到1落地产品skill」</mark>

> 总结来看业务逻辑放skill，外部连接走MCP，二者相辅相成。



---

### MCP 和 Function Calling 的区别

`Function Calling` 是大模型API的原生能力，相当于模型到本地代码之间的闭环；

而 MCP 是一个标准协议，它解决的是工具从哪来、怎么发现、怎么管理。

两者是相互配合的关系，MCP 管工具的发现和管理，`Function Calling` 管模型怎么调用



---

### 没有背景知识时怎么确保答案正确

- <mark style="background:#d3f8b6">RAG（检索增强）</mark>：外挂检索知识库、文档，把相关的内容塞进Prompt，基于既定规范回答
	
- <mark style="background:#d3f8b6">工具调用</mark>：在模型不确定时调用工具「网络搜索、查DB、读文件」，杜绝瞎编
	
- <mark style="background:#d3f8b6">System Prompt 约束</mark>：让模型将不确定的回答标记出来
	
- <mark style="background:#d3f8b6">ReAct模式</mark>：多步验证，让 Agent 先查再回答，而不是一次性输出
	
- <mark style="background:#d3f8b6">人工审批</mark>：关键操作前暂停等待人类确认，增加纠正渠道


### 专化 Agnet 和通用 Agent 有什么区别

> 我觉得可以这样理解，通用Agent要求什么都能干，就是服务各种领域的需求；而专化Agent追求单领域，强相关。把特定事情干可控；

例如以下几个方面两者有显著不同：

- <mark style="background:#d3f8b6">系统提示词</mark>，通用Agent大而全，专化Agent精简，强约束；
	
- <mark style="background:#d3f8b6">可靠性上</mark>，通用Agent行为难预测，专化Agent可靠性高易复现；
	
- <mark style="background:#d3f8b6">成本上</mark>，通用Agent上下文重，token贵，专化Agent上下文小，跑得快；

我的取舍逻辑一般是，通用 Agent 适合做**调度层** —— 理解意图、拆解任务、决定派给谁，执行层交给**专化 Agent**，这就与我的项目思想理念一致，就是一个节点负责跑工作流节点 <mark style="background:#d3f8b6">「调工具，管节点回调」</mark>，另一个只负责和PTY交互 <mark style="background:#d3f8b6">「处理终端回显」</mark>


### 平时怎么使用大模型

> **我的核心原则是"可靠优先于提效"**—— 代码场景下模型输出不可控会直接造成返工，所以我围绕"把不确定性关进可控流程"来用模型，分三层

- **1. <mark style="background:#d3f8b6">分场景沉淀 skill 体系</mark>。** 把单元测试、功能开发、架构设计等场景各自的最佳实践，连同模型常踩的坑，固化成专用 skill —— 而不是用一个万能 prompt 通吃。
	
- **2. <mark style="background:#d3f8b6">场景内加流程约束</mark>。** 每个 skill 内部强制"先计划再决策"，并对关键链路（如 DB 变更、对外接口）设审批节点（HITL），用流程压制随机性避免黑盒。
	
- **3. <mark style="background:#d3f8b6">基于场景的提示词路由</mark>。** 运行时按任务类型动态注入对应指令，避免全量 prompt 互相干扰、也控制上下文膨胀。
	
- **兜底：** 模型仍有不可控时，我沉淀"修正类"提示词——自检清单、反例约束、schema 强校验——做输出纠偏。
	
- 这些组合成我个人的**大模型使用 SOP**：可复用、可迭代，把一次性的 prompt 工程变成可持续的工程资产，目标是显著降低日常开发的返工率。


### 如何降低llm幻觉

参考上面没有背景知识时怎么确保答案正确的问题

> 模型是"参数化记忆 + 概率续写"，因此幻觉的本质在于对于自己不知道的内容会编造，我的环节是分层的：

- <mark style="background:#d3f8b6">利用RAG</mark> ：给事实锚点，通过外挂资料库的形式纠偏，这在我的个人效率工作站的向量数据库里有所体现，让模型读我给的资料给出答案。
	
- <mark style="background:#d3f8b6">工具代替记忆</mark> ：凡是可计算 / 可查询的，交给确定性工具（搜索、API、SQL），模型只负责"组织答案"。
	
- **<mark style="background:#d3f8b6">提示约束</mark>**：明确要求"不知道就说不知道"、强制**引用来源**、用 few-shot 定格式，降低自由发挥。
	
- **<mark style="background:#d3f8b6">人在环 + 评估</mark>：高风险动作必须 HITL 确认；「离线建评测集「[[离线评测集]]」（含对抗样本）持续监控幻觉率，而不是上线后靠运气」


### RAG 的切分与召回权衡

> 关于RAG切分与召回的权衡上，本质上是「精度、完整性、成本」的三方取舍。

- 事实回答偏小，两三百 token。长文摘要就偏大。但光调大小是在精度和完整性之间二选一，所以需要**用父子块把这对矛盾解耦：** 索引时存大块、再细分成小块，只有小块做向量检索保精度，召回后回溯到对应的父块喂给模型保语境。等于"用小块找、用大块答"，两头都拿。
	
- **召回端默认混合检索加重排。** 稠密加 BM25，先召回一二十个再 rerank 到五六个，比纯向量稳得多。

> 上述是一些理论情况，我在打造个人全栈效率平台因为比较集中，加上知识库不大。所以没有采用父子块以及重排。避免过度工程

- <mark style="background:#d3f8b6">切分上</mark>：我按照文档结构切分，技术文档天然有层级——标题、小节、代码块。按 Markdown 标题或代码块边界切，语义边界比硬切 token 干净太多。而且个人库里文档长短不一，得用**自适应大小**，不能一刀切。
	
- <mark style="background:#d3f8b6">召回上</mark>：我采用了向量数据库利用 pgvector 插件结合 pgSql 做了个1024维数据库，并做了一些优化 [[RAG检索优化]]，依靠 embedding 的非对称编码 加上 BM25 混合检索就可以满足我的技术文档库的需求


### LangGraph 和 LangChain 的关系

[[Langchain与LangGraph关系]]



