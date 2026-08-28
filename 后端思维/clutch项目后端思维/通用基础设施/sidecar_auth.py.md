
---

> [!NOTE]
> 这是门禁规则表，负责HTTP、WS的鉴权，HTTP 带 Header，WS/图片带 URL 参数

## 运行链路

```
Tauri 造钥匙
   ↓ 塞进环境变量
Sidecar 启动
   ↓
前端要钥匙（IPC）→ 请求时带上
   ↓
HTTP：main.py 中间件查 Bearer
WS：chat_runner 查 ?token=
   ↓
都去问 sidecar_auth：「过不过？」
```


## 五个函数组件

> 只写规则，不拦请求。拦截逻辑在别的组件

### `auth_required`

1. 如果存在E2E 沙箱，即环境里有 `CLUTCH_E2E_SANDBOX` 则返回false「常跑自动化测试」
	
2. 查环境中有没有 `CLUTCH_SIDECAR_TOKEN` 「Tauri spwan进程注入环境中的」


### `expected_token`

> 从环境变量中读出正确的`CLUTCH_SIDECAR_TOKEN` 给 `validate_token` 用来跟请求带来的token进行对比 



### `validate_token`

> 判断token是否一致

- `auth_required()` 为 False → 直接 `True`
	
- 若为True，则取出正确钥匙 `expected_token`
	
- 正确钥匙或对方 token 缺一个 → `False`
	
- 两边都有 → 用 `compare_digest` 「Python标准库」 安全比对，一样才 `True`



### `validate_bearer`

> 对请求头header的Authorization进行检查并预处理后交给`validate_token` 进行比对

1. `auth_required()` 为 False   → ` True`，放行
2. 没有 Header，或不是以 `Bearer` 开头 → `False`
3. 有 `Bearer xxx` → 去掉前面的 `Bearer` （前 7 个字符），取出 `xxx`，交给 `validate_token` 去比对



### `public_http_paths`

> 给 `main.py` 中间件用，请求路径如果在这个集合里 → 直接放行，不查 token


永远放行：`/health`（探活，看服务是否alive）