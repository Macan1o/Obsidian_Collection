
---
# n8n 工作流本地部署文档（Docker 部署版）

## 一、节点总览

|节点名称|类型|作用|
|---|---|---|
|`On form submission`|Form Trigger|第一步表单：填应用名称/组号/分支名/项目目录|
|`Project Init`|SSH|复制 `uniapp_base` 模板 → 项目目录|
|`Wait`|Wait|第二步表单：选视觉风格（19选N）+ 配色|
|`Message a model`|OpenAI LLM|用通义生成产品需求描述|
|`AI Agent1`|LangChain Agent|单一状态机：初始化项目 + 逐套生成 12 页面|
|`MCP Client1`|MCP Client|连接 Stitch MCP（create/list/generate）|
|`Memory`|Buffer Memory|Agent 对话记忆上下文（⚠️ 当前游离，未接入主流）|
|`DeepSeek Chat Model`|DeepSeek LLM|Agent 底层模型|
|`Code in JavaScript`|Code|解析 Agent JSON 输出，校验 N×12 个 screen_id|
|`Wait1`|Wait|第三步表单：选择用哪套风格|
|`Code in JavaScript1`|Code|根据用户选择提取对应风格数据|
|`Merge`|Merge|合并两路（Project Init 目录 + 风格数据）|
|`success`|SSH|触发 Cursor Agent 全量 UI 开发|
|`fail`|SSH|错误时调 Cursor Agent 自查修复|
|`hbuilder打包`|SSH|调用 HBuilderX CLI 发布 H5|

---

## 二、凭证清单

部署前需在 n8n **凭证（Credentials）** 中提前创建以下 4 个：

|凭证名称|类型|用途|
|---|---|---|
|`ali`|OpenAiApi|通义 API，调 Qwen3-Max|
|`SSH Password account`|sshPassword|SSH 到宿主机执行脚本|
|`DeepSeek account`|deepSeekApi|DeepSeek API，Agent 底层模型|
|`Header Auth account`|httpHeaderAuth|Stitch MCP 请求头认证|

---

## 三、Docker 部署

### 3.1 安装 Docker

**macOS：**

```
# 推荐 OrbStack（轻量高性能）
brew install --cask orbstack

# 或 Docker Desktop
brew install --cask docker
```

**Windows：**

```
# 1. 启用 WSL 2（管理员 PowerShell）
wsl --install

# 2. 下载 Docker Desktop
# https://www.docker.com/products/docker-desktop/
# 安装时勾选 "Use WSL 2 instead of Hyper-V"

# 3. 验证
docker --version
docker compose version
```

### 3.2 启动 n8n 容器

**macOS：**

```
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -v ~/n8n/data:/home/node/.n8n \
  -v ~/uniapp1/uniapp:/workspace/uniapp \
  -e N8N_HOST=0.0.0.0 \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e N8N_WEBHOOK_URL=http://<宿主机IP>:5678 \
  n8nio/n8n
```

**Windows（PowerShell）：**

```
# 创建宿主机挂载目录
New-Item -ItemType Directory -Path "C:\n8n\data" -Force
New-Item -ItemType Directory -Path "C:\uniapp1\uniapp" -Force

docker run -d `
  --name n8n `
  -p 5678:5678 `
  -v C:\n8n\data:C:\Users\ContainerAdministrator\AppData\Roaming\n8n `
  -v C:\uniapp1\uniapp:C:\workspace\uniapp `
  -e N8N_HOST=0.0.0.0 `
  -e N8N_PORT=5678 `
  -e N8N_PROTOCOL=http `
  -e N8N_WEBHOOK_URL=http://<宿主机IP>:5678 `
  n8nio/n8n
```

> 参数说明：
> 
> - -p 5678:5678：Web UI 端口映射
> - 数据卷映射：n8n 数据 + uniapp 项目目录，使 SSH 节点可访问宿主机代码
> - Webhook 和 Form Trigger 外部可访问，必须设置 N8N_WEBHOOK_URL

### 3.3 常用管理命令

```
# 查看运行状态
docker logs -f n8n

# 停止
docker stop n8n

# 重启
docker restart n8n

# 进入容器（调试用）
docker exec -it n8n /bin/sh

# Windows PowerShell 进入
docker exec -it n8n powershell
```

---

## 四、Webhook 与公网访问

### 4.1 配置 Webhook URL

Docker 部署时必须显式指定为宿主机 IP 或域名：

**macOS 获取本机 IP：**

`ifconfig | grep "inet " | grep -v 127.0.0.1 | awk '{print $2}'`

**Windows PowerShell 获取本机 IP：**

`ipconfig | findstr "IPv4"`

### 4.2 三个 Webhook 端点

|节点|Webhook ID|用途|
|---|---|---|
|`On form submission`|`96d2b4d6-ff09-4a7f-9f4b-5cbafeeb11ae`|第一次表单提交|
|`Wait`|`387ed10c-fbc3-4a91-8fec-d84bc9ecc188`|风格/配色表单（Resume）|
|`Wait1`|`7d11fd6a-b48a-40aa-8b21-8c932c253ef0`|风格选择表单（Resume）|

完整 Webhook URL 格式：

```
http://<宿主机IP>:5678/webhook/96d2b4d6-ff09-4a7f-9f4b-5cbafeeb11ae
```

### 4.3 内网穿透（可选）

**Windows 配合 frp：**

```
; frpc.ini
[common]
server_addr = <frps服务器>
server_port = 7000
token = <你的token>

[n8n_webhook]
type = tcp
local_ip = 127.0.0.1
local_port = 5678
remote_port = 5678
```

---

## 五、SSH 节点配置（宿主机访问）

### 5.1 macOS 宿主机

确保已开启**远程登录**：系统设置 → 通用 → 共享 → 远程登录 → 开启

Docker 容器内通过 `host.docker.internal` 连接宿主机：

|参数|值|
|---|---|
|主机|`host.docker.internal`|
|端口|`22`|
|用户名|宿主机用户名|
|认证方式|密码或 SSH Key|

### 5.2 Windows 宿主机

Windows 需额外配置 OpenSSH Server：

```
# 1. 安装 OpenSSH Server（管理员 PowerShell）
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 2. 启动服务
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# 3. 确认防火墙规则
Get-NetFirewallRule -Name *OpenSSH* | Select-Object Name, Enabled
```

SSH 连接参数：

|参数|值|
|---|---|
|主机|`host.docker.internal`（WSL2/Docker Desktop 模式）或宿主机实际局域网 IP|
|端口|`22`|
|用户名|Windows 用户名（如 `shihongwei`）|
|认证方式|密码或 SSH Key|

> ⚠️ Windows SSH 默认 shell 是 PowerShell，部分 bash 命令（如 cd ~）需改为 PowerShell 语法。

### 5.3 SSH Key 免密配置

**Windows 生成密钥（PowerShell）：**

```
ssh-keygen -t ed25519 -C "n8n-docker"
# 保存路径默认: C:\Users\<用户名>\.ssh\

# 将公钥加入 authorized_keys
cat $env:USERPROFILE\.ssh\id_ed25519.pub >> $env:USERPROFILE\.ssh\authorized_keys
```

**挂载私钥到容器：**

```
docker run -d `
  ... `
  -v C:\Users\<用户名>\.ssh:C:\home\node\.ssh:ro `
  n8nio/n8n
```

### 5.4 目录映射关系

**macOS：**

|宿主机路径|容器内路径|
|---|---|
|`/Users/shihongwei/n8n/uniapp`|`/workspace/n8n/uniapp`|
|`/Users/shihongwei/n8n/uniapp/uniapp_base`|`/workspace/n8n/uniapp/uniapp_base`|
|`/Users/shihongwei/.local/bin/agent`|需挂载或复制至容器 PATH|

**Windows：**

|宿主机路径|容器内路径|
|---|---|
|`C:\uniapp1\uniapp`|`C:\workspace\uniapp`|
|`C:\Users\<用户名>\.local\bin\agent`|`C:\workspace\.local\bin\agent`|

> ⚠️ SSH 节点中所有硬编码路径需改为容器内映射后的路径。

---

## 六、宿主机目录结构（示例）

**macOS：**

```
/Users/shihongwei/
├── uniapp1/uniapp/          # uniapp 项目根目录
│   ├── pages/
│   ├── static/stitch/
│   └── .git
├── .local/bin/agent          # Cursor AI CLI
└── n8n/data/                # n8n 数据卷备份
    └── .n8n/
```

**Windows：**

```
C:\
├── uniapp1\
│   └── uniapp\              # uniapp 项目根目录
│       ├── pages\
│       ├── static\stitch\
│       └── .git
├── n8n\
│   └── data\                # n8n 数据卷
│       └── .n8n\
└── Users\<用户名>\
    └── .local\bin\
        └── agent.exe        # Cursor AI CLI
```

---

## 七、执行流程（完整链路）

```
用户 → 填表单1(appName/组号/分支)
     → Git Branch Manager 切分支
     → 填表单2(风格列表/配色)
     → LLM 生成产品需求
     → AI Agent 调 Stitch MCP 生成12页面
     → Code 校验 JSON 结构
     → 填表单3(选第几套风格)
     → Code 提取选中风格数据
     → Merge 合并两路数据
     → SSH → Cursor Agent 全量UI开发
     → SSH → HBuilderX 打包发布
```

---

## 八、导入工作流

### 方式一：UI 导入

1. 浏览器访问 `http://localhost:5678`
2. 左上角 `+` → `Import from JSON`
3. 粘贴 `test5(文件）.json` 内容

### 方式二：docker exec 导入

**macOS/Linux：**

`docker exec -it n8n n8n import:workflow --input=/workspace/n8n项目/test5(文件）.json`

**Windows（PowerShell）：**

```
# 先将文件复制进容器
docker cp C:\n8n项目\test5(文件）.json n8n:C:\workspace\test5.json
docker exec -it n8n n8n import:workflow --input=C:\workspace\test5.json
```

### 方式三：docker cp 传入

```
docker cp <源路径> n8n:/tmp/test5(文件）.json
docker exec -it n8n n8n import:workflow --input=/tmp/test5(文件）.json
```

---

## 九、关键注意事项

1. **路径映射**：SSH 节点中的本地路径在 Docker 环境下对应容器内的映射路径（macOS: `/workspace`，Windows: `C:\workspace`），所有路径需同步调整。
2. **Webhook URL**：必须设置 `N8N_WEBHOOK_URL`，否则 Form 和 Wait Resume 链接生成错误。
3. **HBuilderX 打包**：容器内无法直接调 GUI 应用。如需自动打包，建议在宿主机安装 HBuilderX CLI，SSH 节点调用宿主机 CLI。
4. **Windows SSH shell 差异**：Windows 默认 shell 为 PowerShell，`git`、`cd ~` 等 bash 语法需替换为 PowerShell 等价命令。
5. **MCP 凭证**：`Header Auth account` 中的认证头需与 Stitch MCP 服务端一致。
6. **Agent 重试**：`AI Agent1` 设置了 `retryOnFail: true，maxTries: 5`，Stitch 生成 12 页面耗时长，需关注 n8n 执行超时设置。
7. **项目目录隔离**：`Project Init` 通过复制 `uniapp_base` 模板目录实现项目隔离，不再依赖 git 分支。项目目录名由表单 `project_dir` 字段指定，安全校验正则 `^[a-z0-9_-]+$` 不允许 `/`。
8. **Memory 节点游离**：`test5(文件）` 中 `Memory` 节点已脱离主流，处于空闲状态，不影响流程但可忽略。
9. **单一状态机 Agent**：`AI Agent1` 整合了项目初始化和多风格页面生成，需等待 Agent 完成所有风格（串行）后才能进入下一阶段，耗时长，建议关注 n8n 执行超时。
10. **Cursor Agent**：`success` 节点通过 `osascript`（macOS）或 `start`（Windows）唤起 AI CLI 执行任务，Windows 节点需改用对应 Windows 命令。