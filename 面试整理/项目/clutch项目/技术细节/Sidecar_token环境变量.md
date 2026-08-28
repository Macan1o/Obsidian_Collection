
---

> [!NOTE]
> 注入发生在Tauri spawn Sidebar子进程时，给新进程单独带上环境变量，由 Rust 语法将鉴权 token 给前端发送请求到后端的时候携带完成鉴权

### 应用`setup`的时候

- 生成 `UUID：uuid::Uuid::new_v4().to_string()`
	
- 存进壳内 SidecarAuthState（给前端 invoke 用）
	
- 调用 `spawn_sidecar(..., &token)` —— 注入发生在这里

### 从OS层语义了解注入

Rust 的 std::process::Command（以及 Tauri shell sidecar API）在 spawn 前调用 .env(key, value)，子进程一启动就能用 `os.environ.get("CLUTCH_SIDECAR_TOKEN", "").strip()` 读到

#### 具体链路如下

```
Tauri 进程                          Sidecar 进程 (uvicorn / orchestrator)
─────────                          ──────────────────────────────────
生成 token
  │
  ├─ 存 SidecarAuthState            （前端 invoke 取这把）
  │
  └─ Command.spawn(
       env = { ..., CLUTCH_SIDECAR_TOKEN: token }
     ) ──────────────────────────►  继承这份 env
                                    os.environ["CLUTCH_SIDECAR_TOKEN"]
                                    = 同一把 token
```

就是当Tauri启动的时候会自动生成token，当Tauri新建并启动新进程比如FastAPI Sidecar的时候会自动将token注入到进程中，然后通过os.environ.get获取到token进行比对