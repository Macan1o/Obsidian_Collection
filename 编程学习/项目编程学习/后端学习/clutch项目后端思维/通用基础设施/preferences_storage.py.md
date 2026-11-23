---
created: 2026-08-12T10:17
---


---

> [!NOTE] 组件作用
> `preferences_storage.py` 是用户偏好的本机存盘层：把主题、语言、权限模式、联网开关等写成一个 JSON；前端 Settings / 顶栏 pill 经 API 读写它，Agent 执行时也会读其中的安全与联网旋钮。

### 偏好存储要求

> 聊天主链路不读 UI 状态，但会读这里的 `permission_mode`、`allow_network`。

1. 稳定落盘位置（挂在 `storage_helper` 大楼下的 `preferences/` 房间）
	
2. 校验 + 默认值（坏数据不能让 Settings 崩）
	
3. 给业务读的快捷 API（`load_permission_mode()` / `tr()` 等）


---

## 函数模块

1. 读全部：`load_preferences()`
	
2. 改某一项再保存：`save_theme`、`save_language`、`save_permission_mode`
	
3. 给别处用的快捷读取：
    - `load_permission_mode()` → Agent 能改多少东西
    - `load_allow_network()` → 能不能上网搜
    - `tr(英文, 中文)` → 按当前语言返回提示文案

### Agent相关链路

```
preferences.json
        ↓
load_permission_mode / load_allow_network
        ↓
拼提示词、决定能不能用 web_search 等
```



