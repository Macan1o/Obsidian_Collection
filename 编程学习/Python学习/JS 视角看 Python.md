---
created: 2026-08-12T10:17
---


---
## 底层差异

### 块级作用域

JS用 `let` / `const` 声明的变量被限制在 `{}` 内。**Python 只有模块、类和函数作用域**，在 `if`、`for`、`while` 代码块中定义的变量，在块外部**依然可以正常访问和修改**。

### 强类型 vs 弱类型

JS 喜欢默默帮你做隐式转换，Python 是**强类型**，字符串和数字直接相加相减会直接抛出 `TypeError`，必须显式转换

### 空容器是 Falsy

在 JS 中，`[]` 和 `{}` 都是 Truthy（真值）。在 Python 中，**任何空的容器**——包括 `[]`（空列表）、`{}`（空字典/集合）、`()`（空元组）、`""`（空字符串）、`0` 和 `None`，在布尔判断时都是 **Falsy（假值）**！


---
## 核心语法映射

- 变量表达：大写字母代表常量
	
- 空值表达：只有一个None
	
- 相等比较：== 自带深度对比
	
- 字符串拼接： `f"Hello {name}"` f-string体验舒适，性能高
	
- 匿名函数：`lambda x: x * 2` lambda 极度克制，只能包含一条表达式
	
- 属性导入：Python 导入模块会自动执行该模块顶层代码，注意循环引用问题


## 数据结构差异

### 列表 (List) → 对应 JS Array

- **操作映射**：`arr.push(x)` → `lst.append(x)`；`arr.slice(0, 2)` → `lst[0:2]`。
	
- **超强切片 (Slice)**：Python 的切片语法为 `[start:stop:step]`。 `.reverse()`，直接用 `lst[::-1]` 即可。

### 字典 (Dictionary) → 对应 JS Object / Map

- Python 的字典默认就是**有序的**（插入顺序）
	
- **访问属性防坑**：不能像 JS 一样用点号 `dict.key` 访问，必须用 `dict["key"]`（如果没有该键会报错 `KeyError`），或者用安全的 `dict.get("key", default_val)`
	
- 字典的键可以是任何**不可变类型**（包括元组），比 JS 的 Object 只能用字符串/Symbol 做键强大得多。

### 元组 (Tuple) → JS 中无对应概念

- 语法：`t = (1, 2, "a")`。
	
- 可以把它当成**不可变的数组**。极其常用，多用于函数返回多个值（`return val1, val2`）、作为字典的键、或保护数据不被意外修改。

### 集合 (Set) → 对应 JS Set

	语法：`s = {1, 2, 3}`。直接支持数学逻辑运算：交集 `&`、并集 `|`、差集 `-`。


## 进阶Python特性

### 列表推导式

> 完全取代JS冗长的链式调用

```Python
# JS: const evensSquared = nums.filter(n => n % 2 === 0).map(n => n * n);

# Python: [表达式 for 元素 in 可迭代对象 if 条件]
evens_squared = [n * n for n in nums if n % 2 == 0]
```

### 解包与位置/关键字参数

对应 JS 的展开运算符 `...`，但 Python 对**位置参数**和**键值对参数**作了严格区分

- `*` 用于列表/元组解包（只解包值）。
    
- `**` 用于字典解包（解包键值对，类似 JS 合并对象 `...obj`）。

```Python
# 类似 JS: const merged = { ...defaultCfg, ...customCfg };
def merge_config(default_cfg, **custom_cfg):
    return {**default_cfg, **custom_cfg}
```

### 上下文管理器 (`with` 语句)

在 JS 中打开文件或连接数据库，常需写繁琐的 `try...finally` 确保调用 `.close()`。Python 的 `with` 会在代码块结束或抛出异常时**自动执行安全清理**

```Python
# 无论代码块里是否抛出异常，文件都会被自动且安全地关闭
with open("config.json", "r", encoding="utf-8") as f:
    data = f.read()
```

### 装饰器 (Decorators)

类似 JS / TS 中的实验性装饰器或高阶函数。在 Python 后端（FastAPI、Django、Flask）中无处不在，用于路由绑定、日志拦截、权限校验

```Python
# 定义一个打印执行时间的装饰器
def log_time(func):
    def wrapper(*args, **kwargs):
        print("开始执行...")
        return func(*args, **kwargs)
    return wrapper

@log_time
def fetch_data():
    pass
```

### 异步编程

`async / await` 在 Python 中语法几乎一模一样

```Python
import asyncio

async def fetch_user(user_id):
    await asyncio.sleep(1) # 类似 JS 的 await new Promise(...)
    return {"id": user_id}
```

- JS 引擎默认自带一个全天候运行的**事件循环 (Event Loop)**，你的所有异步代码都自然地跑在循环里。
    
- Python 默认是同步阻塞的。要启动异步代码，你必须显式启动事件循环，比如通过 `asyncio.run(fetch_user(1))`，或者在 Web 开发中交由 ASGI 服务器（如 Uvicorn）托管驱动。


---

## 协议机制

	`__xxx__` = Python 约定的"如果你实现了这个方法，我就会在特定时刻自动调用它"。你负责写逻辑，Python 负责在正确的时机触发。这就是 Python 的协议（Protocol）机制