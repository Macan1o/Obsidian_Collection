
---
## 1. 池化复用 — `ShellSessionManager`

```
_sessions: dict[run_id → ShellSession]
get_or_create(run_id)：
    有且 alive → 直接返回（复用）
    没有/死了 → spawn bash，放进字典
```

同 `run_id` 反复用同一条 PTY，这就是「池化复用」。

## 2. LRU（更准确：挤最久未活动的空闲 session）

```
_evict_oldest_idle：
  只看状态是 IDLE/READY 的
  按 last_activity_at 选最老的
  close 并踢出字典
  腾出位置给新的
```

每次用/释放会 `touch()` 更新活动时间。另有空闲超时/最大存活时间清理。


## 3. 工作区互斥 — `workspace_cli_lock`

> 同一个工作区路径共用一把锁。谁先拿到谁跑，跑完释放

跑命令前先用`threading.Lock`将路径锁住，拿到这把路径对应的 `Lock` 对象，对它调用 `acquire()` / `release()`

```Python
# 内存字典：路径 → 锁
_locks: dict[str, threading.Lock] = {}

def _get_lock(workspace_path: str) -> threading.Lock:
    if workspace_path not in _locks:
        _locks[workspace_path] = threading.Lock()  # 第一次才创建
    return _locks[workspace_path]

# 跑 hybrid turn 时：
lock = _get_lock("/Users/me/proj")

lock.acquire(timeout=600)   # 锁住：别人同路径要等
try:
    往 PTY 写命令、等 marker、解析输出
finally:
    lock.release()           # 释放：下一个才能进来
```