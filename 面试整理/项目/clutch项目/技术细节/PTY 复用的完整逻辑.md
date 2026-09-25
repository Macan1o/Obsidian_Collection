
---

> [!NOTE] 基本复用逻辑
> 同一 run_id 的多个 turn 共用一条 Shell

### 创建过程

通过 `pty.openpty()` 开 主/从两端，slave 端给 bash，master 端留在 Python 手里——**写命令 = 模拟用户敲键盘，读输出 = 读终端回显**。

#### 复用三层环境的体现

- **进程层**：bash 跨进程跨 turn 不退出，将 cwd 锁在用户指定的 workspace 
	
- **文件系统层**: 上一个 turn 创建的文件、装好的 venv 、export 变量下一个 turn 还存在，**CLI 是真的在这棵文件树上改代码、跑脚本**，环境连续性 = 任务连续性。
	
- **CLI 会话层**：`cli_session_id` 通过 `--resume` / `--session-id` / `--conversation` 传给 CLI（具体旗标由 `conversation_mode` 决定），让 **CLI 自己的上下文也跨 turn 续接**——不只是 shell 活着，是模型的对话记忆也活着。

