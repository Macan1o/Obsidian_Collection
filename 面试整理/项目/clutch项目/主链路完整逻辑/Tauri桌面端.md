
---

> [!NOTE]
> Tauri：开窗口显示前端 + 拉起/看管 Sidecar；并补浏览器没有的能力（选目录、打开本地路径、把鉴权 token 给前端调 Sidecar）。

## Tauri 做桌面壳

- Rust 负责窗口、打包，以及启动/监督 Python Sidecar，并生成 loopback 鉴权 token。[[Sidecar_token环境变量]]
	 
- UI 是 React，跑在系统 WebView 里；
	
- 编排和 Agent 全在 FastAPI + LangGraph 的 Sidecar。  

前端和 Rust 只通过少量 `invoke` 交互（选目录、取 token、更新）；业务一律 HTTP/WebSocket 打本机 Sidecar。  
选 Tauri 而不是 Electron，主要是壳更轻，而且业务已经在 Python，不需要再在 Node 里做一层编排。


---
## `invoke` 是什么

```
React (JS)
   │  invoke('命令名', 参数)
   ▼
Tauri IPC 桥（安全白名单 capabilities）
   ▼
Rust #[tauri::command] 函数
   │  返回值序列化成 JSON
   ▼
再回到 React 的 Promise
```

`invoke` 是前端调 Rust 的 IPC，只用于壳能力：取 session token、选目录等

```
                    ┌──────────────────────────┐
                    │     Tauri 壳 (Rust)       │
                    │  · 窗口 / WebView         │
                    │  · spawn/kill Sidecar     │
                    │  · token 内存态           │
                    │  · command / 插件         │
                    └────────────┬─────────────┘
           ① invoke / 插件 IPC   │   ② spawn + 环境变量
                    │            │
         ┌──────────▼──────┐     │
         │  React 前端     │     │
         │  (WebView)      │     │
         └──────────┬──────┘     │
                    │            │
                    │ ③ HTTP/WS  │
                    │ Bearer     │
                    ▼            ▼
              ┌─────────────────────┐
              │ Python Sidecar      │
              │ FastAPI + LangGraph │
              └─────────────────────┘
```

> token 不是后端发给前端的登录接口，而是：壳启动时造一份，同时塞给 Python 和（经 invoke）给前端，两边对得上才能访问 loopback API。

### loopback API 

只监听在本机回环地址上的接口，HTTP/WebSocket 服务绑在 `127.0.0.1:端口`，给同一台机器上的其它进程用——在 Clutch 里就是 WebView 里的前端调 Python Sidecar。

1. 本地闭环  
    编排、对话、密钥、工作区都在本机进程间完成，不必先有一台云端服务器，也符合 Clutch「本地优先、源码不出机」的定位。
    
2. 前后端能拆开，又不用上公网  
    React 和 Python 是两个进程，用 HTTP/WS 通信最省事；绑在 loopback 上，等于「桌面 App 内部的私有总线」，不是对外开放的产品 API。
    
3. 桌面形态的自然选择  
    Tauri 负责窗口和拉起 Sidecar，UI 连 `127.0.0.1:8123/8124` 就能工作；用户装的是一个 App，不是「先部署后端再打开网页」。




