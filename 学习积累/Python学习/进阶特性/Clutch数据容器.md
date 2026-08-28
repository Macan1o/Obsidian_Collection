
---
## 8. `@dataclass`：`frozen` / `slots` / `field` / `@property` / `@classmethod`

这是本仓库最常见的「高阶但日常」写法。

### `frozen=True` — 不可变名片

创建后不能改字段。适合「一次调用的结果」「模型规格」——改了会出诡异 bug 的值对象。

```Python
#cli_adapter.py lines 29-40

@dataclass(frozen=True)
class CliResult:
    """一次 CLI 调用的原始结果。"""

    command: list[str]
    exit_code: int
    stdout: str
    stderr: str

    @property
    def ok(self) -> bool:
        return self.exit_code == 0
```

`@property`：把方法当属性用，`result.ok` 而不是 `result.ok()`。派生值不要存成字段。

### `slots=True` — 省内存、禁多余属性

```python
#config.py lines 15-26

@dataclass(slots=True)
class VideoConfig:
    """Immutable configuration for a single video production run."""

    resolution: Resolution = "1080p"
    codec: Codec = "h264"
    output_format: OutputFormat = "mp4"
    fps: int = 30
    bitrate_mbps: float = 8.0
    output_dir: Path = field(default_factory=lambda: Path.cwd() / "outputs" / "videos")
```

### `field(default_factory=...)` — 可变默认值的正确写法

```python
#shell_session.py lines 167-178


@dataclass
class ShellSession:
    run_id: str
    workspace_path: str
    owner_node_id: str = "plain_chat"
    state: SessionState = SessionState.CREATING
    master_fd: object = -1
    pid: int = -1
    _proc: subprocess.Popen[bytes] | None = field(default=None, repr=False)
    created_at: float = field(default_factory=time.monotonic)
    last_activity_at: float = field(default_factory=time.monotonic)
    _lock: threading.RLock = field(default_factory=threading.RLock, repr=False)
```

`repr=False`：打印对象时隐藏内部锁 / 进程，避免日志泄露实现细节。`default_factory=time.monotonic` 传入的是函数本身，不是 `time.monotonic()` 的调用结果。

### `@classmethod` — 另一种构造方式

```python
# config.py line 42-51

    @classmethod
    def from_env(cls) -> VideoConfig:
        """Build config from environment variables (for CI / containerised builds)."""
        cfg = cls()  # picks up all dataclass defaults
        if (v := os.getenv("VIDEO_RESOLUTION")):
            cfg.resolution = v  # type: ignore[assignment]
```

记住： `frozen` 防改、`slots` 防乱挂、`default_factory` 防共享可变默认值。


## 9. `str, Enum` — 既是枚举又能当字符串

作用： 继承 `str` 后，`SessionState.READY == "ready"` 为真，JSON/日志里直接是字符串，又有一组合法值。

场景： 会话状态、运行策略、审批状态。

```python
# shell_session lines 51-58


class SessionState(str, Enum):
    CREATING = "creating"
    READY = "ready"
    BUSY = "busy"
    IDLE = "idle"
    DISCONNECTED = "disconnected"
    RECOVERING = "recovering"
    TERMINATED = "terminated"
```

```python
#runtime_stategy.py lines 7-12

class RuntimeStrategy(str, Enum):
    SHELL_EXEC = "shell_exec"
    INTERACTIVE_PTY = "interactive_pty"
    HTTP_DAEMON = "http_daemon"
    SDK_NATIVE = "sdk_native"
```


## 10. Pydantic `BaseModel` + `Field` — HTTP 边界上的校验

作用： FastAPI 把 JSON body 自动解析成这些类：缺字段、类型错、超出范围直接 422。业务函数拿到的已经是干净对象。Field 加默认值和范围

场景： 所有 `routes/*.py` 的请求体。内部状态用 TypedDict「内部流转的 dict」，对外 API 用 Pydantic「不可信的 HTTP JSON」

```python 
class SessionCreateRequest(BaseModel):
    run_id: str
    title: str = Field(default="New session")
    workflow_id: str = Field(default="")
    mode: str = Field(default="coding")
    status: str | None = None
# ...
class ForkSessionRequest(BaseModel):
    message_index: int = Field(ge=0)


class RewindFilesRequest(BaseModel):
    count: int = Field(default=1, ge=1, le=10)
```

`ge=0` / `le=10` 是约束，不是注释。装饰器 `@router.post(...)` 把函数注册成路由——这也是装饰器最典型的用法。