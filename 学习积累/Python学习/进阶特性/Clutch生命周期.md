
---
## 11. 装饰器三件套：`@lru_cache` / `@contextmanager` / `@asynccontextmanager`

装饰器 = 「用函数包函数」。本项目里三种都很经典。

### `@lru_cache` — 记住函数结果

Schema 文件只读一次，校验工作流时反复用。

```python 
@lru_cache(maxsize=1)
def _load_schema() -> dict[str, Any]:
    schema_path = _schema_path()
    if not schema_path.is_file():
        raise WorkflowValidationError(
            tr(
                f"Schema file not found: {schema_path}",
                f"未找到 Schema 文件：{schema_path}",
            ),
            [],
        )
    return json.loads(schema_path.read_text(encoding="utf-8"))
```

`maxsize=1`：只缓存一份。路径变了要 `cache_clear()`。

### `@contextmanager` + `yield` — 同步的「进入/退出」

文件锁：`with file_lock(f):` 保证无论成功失败都解锁。`yield` 前是获取，后是释放。

```python 
@contextmanager
def file_lock(handle: IO[str], *, exclusive: bool = True) -> Iterator[None]:
    handle.seek(0)
    # ...
    mode = fcntl.LOCK_EX if exclusive else fcntl.LOCK_SH
    fcntl.flock(handle.fileno(), mode)
    try:
        yield
    finally:
        fcntl.flock(handle.fileno(), fcntl.LOCK_UN)
```


### `@asynccontextmanager` — FastAPI 应用寿命

`yield` 之前启动后台任务，之后 cancel。这就是 Sidecar 进程的「开机/关机脚本」。

```python
@asynccontextmanager
async def _lifespan(app: FastAPI):
    """应用生命周期：yield 之前 = 启动准备；yield 之后 = 优雅关闭。"""
    # ... 启动清扫循环、定时任务 ...
    yield  # ← 服务在此期间正常接请求
    # ----- 关闭阶段：取消后台任务 -----
    sched_task.cancel()
    try:
        await sched_task
    except asyncio.CancelledError:
        pass
```

记住： 同步资源用 `@contextmanager`，async 资源用 `@asynccontextmanager`；`try/finally` 包住 `yield` 才能保证释放。


## 12. `async` / `await` / `asyncio.to_thread` / `create_task`

这是编排服务的骨架：HTTP/WebSocket 必须异步，但大量工作（子进程、读文件、调同步 LLM）是阻塞的。

|语法|作用|本项目场景|
|---|---|---|
|`async def` + `await`|协程：等 I/O 时把线程让给别人|所有 FastAPI 路由|
|`asyncio.to_thread(fn)`|把同步阻塞函数丢到线程池|工作流、扫 shell、读 keychain|
|`asyncio.create_task`|开火即忘的后台循环|60 秒清闲置、15 秒 tick 定时任务|
|`asyncio.CancelledError`|任务被 cancel 时抛出|关机时吞掉，正常退出|
|`asyncio.Lock` + `async with`|协程互斥|shell 池排队一次只 drain 一个|

```Python
    async def _sweep_shell_sessions() -> None:
        """每 60 秒扫一次：关掉闲置 Hybrid shell，并清理过期会话快照。"""
        manager = get_shell_session_manager()
        while True:
            await asyncio.sleep(60)
            try:
                terminated = await asyncio.to_thread(manager.sweep_idle)
                for run_id in terminated:
                    logger.info("shell_session idle sweep terminated run_id=%s", run_id)
```

排队 drain 用锁，避免两个协程同时从队列抢任务：

```Python
    async with _drain_lock:
        manager = get_shell_session_manager()
        while _queue:
            item = _queue[0]
            if not manager.pool_has_capacity(item.run_id):
                retry_needed = True
                break
            _queue.popleft()
```

易错点： 在 `async def` 里直接调用阻塞的 `subprocess.run` / `time.sleep` 会卡住整个事件循环。本项目的纪律是：阻塞 → `to_thread`。


## 16. 嵌套函数 / 闭包

作用： 内层函数能捕获外层变量（这里是 `loop`、manager），把「实现细节」关在 lifespan 里，不污染模块命名空间。

```Python
    async def _forward_interactive_pty_output(session_key: str, chunk: str) -> None:
        """终端里打出的字 → 找到对应 run 的 WebSocket → 推给前端 Terminal 面板。"""
        from src.terminal_orchestra import parse_pty_session_key

        parent_run_id, lane_id = parse_pty_session_key(session_key)
        from src.plain_chat_pool_queue import get_plain_chat_ws
        websocket = get_plain_chat_ws(parent_run_id)
        if websocket is not None:
            await _send_pty_output(websocket, parent_run_id, chunk, lane_id=lane_id)
```

同文件里 `_sweep_shell_sessions`、`_scheduled_tasks_loop` 也是闭包：只在进程生命周期内存在。

`plain_chat_pool_queue.py` 用 `global` 改模块级状态（事件循环、drain task）。这是「单例模块」写法，不是语法糖，但是生产里常见：一个进程一份队列。