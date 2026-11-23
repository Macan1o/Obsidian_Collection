---
created: 2026-09-02T14:40
---


---

> [!NOTE] 代码路径
> 详情代码在 Python_Study / 本地MCP通信


> [!NOTE] 归纳
> 本地 MCP 就是「起子进程 + 一行 JSON 写 stdin + 一行 JSON 读 stdout」，项目中的 McpClient 在此基础上多了超时、OAuth、HTTP 远程等生产逻辑。

### 运行脚本

	python stdio_mcp_client_demo.py

### 输出期望

```
initialize => {'protocolVersion': '2024-11-05', ...}
tools/list => {'tools': [{'name': 'echo', ...}]}
tools/call => {'content': [{'type': 'text', 'text': 'echo: hello stdio'}]}
```

### 运转示意图

```
父进程(客户端)                         子进程(MCP server)
     │                                      │
     │  stdin  ──►  {"method":"initialize"} │
     │  stdout ◄──  {"result":{...}}        │
     │                                      │
     │  stdin  ──►  {"method":"tools/list"} │
     │  stdout ◄──  {"result":{"tools":[...]}} │
     │                                      │
     │  stdin  ──►  {"method":"tools/call"} │
     │  stdout ◄──  {"result":{...}}        │
```

