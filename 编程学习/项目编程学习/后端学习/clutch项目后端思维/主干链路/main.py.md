---
created: 2026-08-12T10:17
---


---

> [!NOTE]
> 编排 Sidecar 的 FastAPI 入口，本机后端进程从这里启动

# 前置引用
## `import asyncio`

Python 官方异步 IO 核心库；FastAPI 是异步框架，`async def` 异步函数、任务并发、休眠、异步锁、异步事件调度全都依赖它。


## `import logging`

Python 官方日志模块，用来打印分级日志（info/warn/error/debug）。

```Python
logger = logging.getLogger(__name__)

# __name__ 是当前模块文件名
# 拿到专属日志实例，后续 `logger.info()`、`logger.error()` 打印运行日志，替代零散 `print()`，线上可以配置日志文件、日志级别、格式化输出。
```


## `from contextlib import asynccontextmanager`

专门写**异步的上下文管理器**，项目启动前执行初始化逻辑、项目关闭前执行销毁清理逻辑（连接池初始化、数据库连接、后台任务启停、资源释放）。类似于已经写好的初始化 + 收尾的语法糖装饰器


## FastAPI 核心包

```Python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
```

后端 Web 服务入口，对标 Node.js 的 Express、NestJS。负责创建服务实例、注册路由、请求参数解析、自动接口文档、类型校验、依赖注入等。

`CORSMiddleware` 跨域中间件，解决浏览器**跨域请求**问题


## starlette（FastAPI 的底层基座）

> FastAPI 本身是基于 Starlette 封装的，Starlette 是轻量级异步 Web 框架

```Python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
```

### BaseHTTPMiddleware

自定义全局 HTTP 中间件基类。可以拦截每一条进来的请求、统一做前置处理（鉴权、日志打印、限流、请求耗时统计）、后置统一修改返回结果。对标 Express 的全局中间件。

### Request

封装单次 HTTP 请求的全部信息：请求头、Cookie、请求参数、body、客户端 IP、路径等，接口函数接收这个对象读取原始请求数据。

### JSONResponse

手动构造 JSON 格式的返回响应，设置状态码、响应头、返回 JSON 数据。 FastAPI 默认返回字典会自动包装成 JSONResponse，需要精细控制响应时手动使用。


## 项目内部自定义模块

### 1. `src.release_hardening`

`api_docs_enabled`：布尔值开关，控制**FastAPI 自动生成的 Swagger 接口文档是否开启**。 线上环境为了安全，经常关闭接口文档，测试环境打开。

### 2. `src.sidecar_auth` 鉴权相关整套工具（接口权限校验）

1. `public_http_paths`：白名单路径列表，不需要登录鉴权就能直接访问的接口（登录接口、健康检查接口）；
2. `validate_bearer`：校验请求头里 `Authorization: Bearer xxx` 格式的令牌；
3. `validate_token`：解析、校验 token 合法性（验签、判断是否过期、解析出用户 / 会话信息）；
4. `auth_required`：鉴权装饰器 / 依赖项，加到接口上强制要求登录校验，未携带合法 token 直接拒绝访问。


---

# 代码逻辑

## `_lifespan`

> 进程启动时挂后台任务（清闲置 shell、定时任务），关闭时 cancel。只做进程级开机接线，服务医生只跑一次。把「PTY有输出时谁找谁」注册好

1. `yield` **之前**：服务启动阶段，所有初始化、注册绑定、启动后台常驻任务；
	
2. `yield` 这一行卡住：正式对外提供 HTTP/WebSocket 接口，接收前端请求；
	
3. `yield` **之后**：服务收到关闭信号（`ctrl+c`、容器销毁、重启），优雅关停后台异步任务，释放资源。

#### 关于为什么`_lifespan`只有后端到前端的输出的解释

```
前端 Terminal 敲键 / 开终端
        │  WebSocket 发 JSON
        ▼
ws_run（chat_runner）收 action
  pty_attach / pty_input / pty_resize / …
        │
        ▼
interactive_pty_manager 写进真正的 PTY 进程
        │
        │  进程自己往 stdout 吐字（异步、持续）
        ▼
output_handler（lifespan 里挂的那根线）
        │  WebSocket event: pty_output
        ▼
前端 Terminal 面板渲染
```

只负责将水龙头接到水管上，具体业务不涉及到，相当于一个调度中心

将对应的功能交给对应的函数处理链路，做一些初始化链路配置

# 创建 FastAPI 应用

先创建FastAPI应用并传入基础配置「绑定生命周期、生产加固开关」

## CORS中间件

桌面 WebView / Vite 开发页跑在 3000 端口，需要跨域访问本机 8124/8123。因此需要配置allow_origin_regex放行本机相关源。因为Sidebar本就不该对外网开放，CORS方便「同源策略下本机前端」调API

## SidebarAuthMiddleware

鉴权中间件，不是公开路径就要携带 Tauri 注入的 Bearer token，防止本机别的程序乱掉编排接口。完整链路

```
Tauri 启动
  → 生成 UUID
  → 注入 Python: CLUTCH_SIDECAR_TOKEN
  → 内存: SidecarAuthState

前端第一次调 API
  → invoke('clutch_sidecar_token')
  → Authorization: Bearer <同一 UUID>
  → SidecarAuthMiddleware 比对 env → 放行

前端连 Chat WS
  → ws://...?token=<同一 UUID>
  → ws_run 里 validate_token → accept

本机其它程序乱 curl 无 token
  → 401（只要 env 里设了 token）
```

```Python
class SidecarAuthMiddleware(BaseHTTPMiddleware):

# dispatch： 中间件固定异步入口函数，每一条进入服务的 HTTP 请求都会先走这个方法；
# request: Request：FastAPI 封装的当前完整请求对象（请求头、路径、参数、方法、Cookie 等）；
# call_next：放行钩子，执行它代表把当前请求交给后续路由、业务接口继续处理；
# async：异步写法，适配 FastAPI 异步运行模式，不会阻塞事件循环。

    async def dispatch(self, request: Request, call_next):
        if request.method == "OPTIONS":  # 浏览器跨域预检请求，前端发起跨域接口调用前浏览器自动发送
            return await call_next(request)
        path = request.url.path  # 提取当前请求的接口路径
        if path in public_http_paths():  # 白名单公开接口直接放行（健康检查、静态资源、登录页等）
            return await call_next(request)
        if not auth_required():  # 全局配置开关函数，主要区别线上和线下开发环境
            return await call_next(request)
        if not validate_bearer(request.headers.get("authorization")):  # 自定义校验函数
            # 媒体资源偶发用 query ?token=（img src 不好带 Header）把 Token 拼在 URL 参数里。
            if path == "/api/workspace/media" and validate_token(request.query_params.get("token")):
                return await call_next(request)
            return JSONResponse(
                status_code=401,
                content={
                    "detail": {
                        "message": "Unauthorized sidecar request",
                        "message_zh": "未授权的 Sidecar 请求",
                    }
                },
            )
        return await call_next(request)

app.add_middleware(SidecarAuthMiddleware)
```

#### 关于`validate_bearer` 自定义校验函数「在sidecar_auth.py里」

1. auth_required() 为假时，也就是测试环境，任何请求都通过
	
2. Header缺失或格式不对：没有authorization、空字符串等格式都会失败
	
3. 抽出token比对，去掉字符再`.strip()` 清洗格式比对 [[Sidecar_token环境变量]]


# 路由分组挂载到全局状态

> `app.include_router(xxx_router)`把整个项目按业务拆成一个个独立小模块，每个模块一套路由，最后统一注册进主服务，挂载之后对应的接口才会对外生效。

**所有这些路由下的接口，全部都会先走全局鉴权中间件校验 Token**，除非路径放进了白名单 `public_http_paths`。










