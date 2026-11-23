---
created: 2026-08-12T10:17
---


---

> [!NOTE] 组件作用
> `storage_helper.py` 不是业务模块，而是整条「本地落盘」链路的总地址簿。只负责算出绝对路径，真正读写 JSON 的是上层各个 `XXX_storage.py`。

## 源码理解

```Python
def get_storage_dir() -> Path:
    # 函数名：拿到「存储根目录」
    # -> Path：返回值类型是 pathlib.Path（路径对象，不是普通字符串）

    """返回 Clutch 本机数据根目录（已 resolve）。
    # 文档字符串：说明这个函数干什么
    # 「已 resolve」= 变成绝对路径、展开软链等，避免相对路径歧义

    - DEV/TEST：`…/Application Support/clutch_dev`（macOS 示例）
    # 开发/测试：目录名用 clutch_dev，和正式用户数据分开

    - PROD（`sys.frozen`）：同路径下的 `clutch`
    # 打包后的生产程序：目录名用 clutch

    - Override：`CLUTCH_STORAGE_DIR` 绝对路径优先
    # 若设了环境变量，就用你指定的路径，上面两套默认全部作废
    """

    override = os.environ.get("CLUTCH_STORAGE_DIR")
    # 从环境变量里读 CLUTCH_STORAGE_DIR
    # 没设时 get 返回 None；设了就得到字符串路径

    if override:
        # 只要 override 非空（设过环境变量），就走「强制覆盖」分支
        return Path(override).expanduser().resolve()
        # Path(override)：把字符串变成 Path 对象
        # .expanduser()：把开头的 ~ 展开成用户主目录（如 ~/tmp → /Users/你/tmp）
        # .resolve()：转成绝对路径并规范化（处理 . / .. 等）
        # return：直接返回，后面平台判断都不再执行

    # PyInstaller 打包后 sys.frozen=True → 生产目录名
    # 上面这行是给人看的说明：为什么要用 sys.frozen

    sub_dir = "clutch" if getattr(sys, "frozen", False) else "clutch_dev"
    # 三元表达式：选子目录名
    # getattr(sys, "frozen", False)：
    #   - 若 sys 有 frozen 属性就取它的值
    #   - 没有就当 False（普通 python / uvicorn 开发态）
    # frozen=True → 打包产物 → 子目录 "clutch"
    # frozen=False → 开发态 → 子目录 "clutch_dev"

    if sys.platform == "darwin":
        # darwin = macOS
        return Path.home() / "Library" / "Application Support" / sub_dir
        # Path.home()：当前用户主目录，如 /Users/macando
        # / 是 Path 的路径拼接运算符（不是除法）
        # 最终类似：/Users/macando/Library/Application Support/clutch_dev

    if sys.platform == "win32":
        # win32 = Windows
        return Path(os.environ.get("APPDATA", Path.home())) / sub_dir
        # 优先用环境变量 APPDATA（常见如 C:\Users\你\AppData\Roaming）
        # 若没有 APPDATA，退回用户主目录 Path.home()
        # 再拼上 clutch 或 clutch_dev

    # Linux 等
    # 走到这里：既不是 macOS 也不是 Windows（主要是 Linux）
    return Path.home() / ".local" / "share" / sub_dir
    # 遵循 XDG 习惯：~/.local/share/clutch_dev（或 clutch）
```

	CLUTCH_STORAGE_DIR：强制指定 Clutch 本机数据根目录的绝对路径