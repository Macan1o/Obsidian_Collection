
---

> [!NOTE]
> Hybrid = 不每个请求新起一个 CLI 进程，而是按 `run_id` 保活一个 bash PTY，在里面反复跑 `claude -p …` 这类命令；跑完用 marker 截输出。



`cli_session_id` 解决的是，同一个 run_id 里，多轮去调外部 CLI 时，怎么接着CLI自己的那条会话。cli_session_id 只是一把钥匙，外部CLI的会话存储中已经存了完整对话，只需要通过这个找到对应的对话即可



## 基本链路

```
get_or_create(run_id) → 长驻 bash
        ↓
拼命令：claude -p "..." ; echo __CLUTCH_DONE_xxx__
        ↓
write 进 PTY → read 直到 marker
        ↓
解析输出 → 返回文本
```

实现位置：`shell_session.py`（保活 PTY）+ `shell_exec_runtime.py`（写命令/等标记/解析）


## 逻辑总结


- 首次执行到 CLI 分支的时候先起 bash 进程
	
- 然后用 PTY 将 Python 和 bash 接上
	
- 利用 master_fd ，把「pid + master_fd」装进 ShellSession 中
	
- 按 run_id 将 ShellSession 完成映射。
	
- 后续同 `run_id` 再来就直接从字典取 `ShellSession`，用已有的 `master_fd` 继续写命令。


## 逻辑补充

### 前置概念

- `master_fd` = PTY 主端的文件描述符；跟 bash 对话的通道；
	
- `ShellSession` 就是这一条长驻 bash 的会话对象「上下文」

```
Python ←→ master_fd
              ↕ 同一根线
bash   ←→ slave
```

创建时：

1. `openpty()` 得到 master + slave
	
2. bash 的 stdin/stdout/stderr 接到 slave
	
3. Python 只留 master
	
4. slave 在父进程里关掉（bash 子进程已经拿着了）

> 平时读写只用 `master_fd`


### 兜底逻辑

> session.alive() 方法根据返回值判断是否失活

每轮执行完后通过 `upsert_hybrid_turn_snapshot` 更新 JSON存在磁盘的JSON上，后续检查如果旧 shell 失活就利用`load_snapshot` → `format_handoff_prefix` 垫到下一轮 prompt 前面


### 失活判断逻辑

`alive()` 的判断逻辑

```Python
def alive(self) -> bool:
    # 1. 有子进程对象时：poll() 为 None = 还在跑；有返回码 = 已退出
    if self._proc is not None:
        return self._proc.poll() is None

    # 2. 否则用 pid 探活：能 kill(pid, 0) 说明进程存在
    if self.pid <= 0:
        return False
    try:
        os.kill(self.pid, 0)   # 信号 0 = 只检查，不真杀
        return True
    except OSError:
        return False           # 找不到进程 = 失活
```

