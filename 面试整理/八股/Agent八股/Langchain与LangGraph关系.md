
---
#### 可以这样类比

> LangChain 是「工具箱」，LangGraph 是「流水线车间」。

## LangChain

	核心是把调用各种 AI 模型和外部工具的代码，标准化、简单化。

将所有模型的接口格式统一，更换模型只需要一行代码

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# 接口被统一了，换模型只需要换一行代码
llm = ChatOpenAI(model="gpt-4") 
# llm = ChatAnthropic(model="claude-3")
```

#### 因此其优势

	通过链把操作串起来：`Prompt -> LLM -> OutputParser`

- 模型抽象：统一不同大模型的调用接口。
	
- Prompt 模板管理：帮你方便地拼接字符串（比如把用户的历史聊天记录拼接到 Prompt 里）。
	
- 工具封装（Tools）：把 Python 函数包装成大模型能理解的格式（比如把 `read_file` 变成一个 Tool）。
	
- RAG（检索增强生成）：帮你切分文档、存入向量数据库、检索相似内容。

#### 其缺点同样明显

	复杂的业务逻辑，比如多节点传递、数据回传；危险工具人工干预；当前执行步骤状态持久化等复杂问题会导致代码逻辑比较混乱


## LangGraph

> 将任务抽象成了图「状态机」，只注重状态如何流转

#### 其优势在于

- 循环（Cycles）：天然支持打回重做（比如 Coder -> Reviewer -> Coder）。
	
- 多 Agent 协作：让不同的 Agent 负责图里的不同节点。
	
- 持久化与暂停（Checkpointer）：跑到一半可以把状态存进数据库，随时暂停（等人类审批），随时恢复。
	
- 可控性：比起让大模型自由发挥（容易失控），LangGraph 让你用代码画好边界，大模型只能在规定的节点里干活。


## 两者总结

- LangChain的优势在于数据预处理和准备工作，例如数据接入与处理、统一模型与工具的接口等。解决了Agent 怎么看懂数据、怎么使用工具处理逻辑的问题
	
	
- LangGraph则注重多Agent复杂业务逻辑流的逻辑链路运转，更加偏向工程层面。构建起更高效的中枢系统
