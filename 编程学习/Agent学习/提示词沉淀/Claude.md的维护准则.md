---
created: 2026-08-12T10:17
---


---
> Claude Code每次启动时自动读取的**持久化指令文件**。

相当于写一份文档，告诉他我们用什么技术栈、代码风格怎么要求、测试怎么跑、哪些坑千万别踩。

为了兼容性通常会让 CLAUDE.md 去加载一下 AGENTS.md，省得一个项目维护两份规则。

### Claude Code的四层加载体系

#### 第一层：全局配置

用来放一些个人编码偏好、全局设置、通用指令

#### 第二层：项目层

路径是项目根目录下的CLAUDE.md，提交到git里，团队共享。这是用得最多的一层，写构建命令、代码规范、架构约定。

#### 第三层：目录层

主要是按需加载，负责模块说明、局部规则、特定约束等内容。当 Claude 读取到那个目录下的文件时，对应的 CLAUDE.md 才会被加载进上下文。涉及利于monorepo，前后端规则分离互不干扰

![[Pasted image 20260722162331.png|600]]

加载顺序是从文件系统根目录一路往下走到启动 Claude Code 的工作目录。所有文件的内容会**拼接在一起**。如果两个文件的指令冲突了，靠近工作目录的那个会因为“最后被读到”而有更高的优先级。

##### Claude code源码
```JS
核心逻辑在 src/utils/claudemd.ts 的 getMemoryFiles() 函数里：

// 第一步：加载 Managed（系统级）和 User（用户级）的 CLAUDE.md + rules
const managedClaudeMd = getMemoryPath('Managed')
result.push(...(await processMemoryFile(managedClaudeMd, 'Managed', ...)))
 
const userClaudeMd = getMemoryPath('User')
result.push(...(await processMemoryFile(userClaudeMd, 'User', ...)))
 
// 第二步：从当前目录一路往上走到根目录，收集所有路径
const dirs: string[] = []
let currentDir = getOriginalCwd()
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}
 
// 第三步：反转！从根目录往下走，依次加载每个目录的 CLAUDE.md
for (const dir of dirs.reverse()) {
  // 加载 CLAUDE.md（Project 类型）
  const projectPath = join(dir, 'CLAUDE.md')
  result.push(...(await processMemoryFile(projectPath, 'Project', ...)))
 
  // 加载 .claude/CLAUDE.md（也是 Project 类型）
  const dotClaudePath = join(dir, '.claude', 'CLAUDE.md')
  result.push(...(await processMemoryFile(dotClaudePath, 'Project', ...)))
 
  // 加载 .claude/rules/*.md
  const rulesDir = join(dir, '.claude', 'rules')
  result.push(...(await processMdRules({ rulesDir, type: 'Project', ... })))
 
  // 加载 CLAUDE.local.md（Local 类型）
  const localPath = join(dir, 'CLAUDE.local.md')
  result.push(...(await processMemoryFile(localPath, 'Local', ...)))
}
```

**不要让不同层级的 CLAUDE.md 互相矛盾**。这是维护的第一原则。


### LLM 指令预算

> CLAUDE.md 太长，Claude 会忽略一半，因为重要规则会被噪音淹没。

瓶颈在于模型的注意力分配不过来，本身Claude Code就内置了大量的指令，占据了很多指令位，进一步压缩了CLAUDE.md的有效空间

#### 怎么判断一条指令该不该放进 CLAUDE.md

- 如果不写这条，Claude 会不会搞错？如果 Claude 靠读代码就能推断出来，别写。
	
- 这条指令是不是每次会话都需要？如果只在特定场景下需要，放到 `rules/` 目录里做路径限定（后面讲），别塞在 CLAUDE.md 里浪费预算。

> **像维护代码一样维护 CLAUDE.md**。

定期 review，发现 Claude 没遵守的指令，加“重要”或“务必要执行”强调；发现 Claude 本来就会做对的指令，果断删掉。

### 什么样的规则会真正生效

如果要精简一点，要分清楚哪些规则有用、哪些是噪音。

##### 真实高效的规则
![[Pasted image 20260722170716.png|630]]

	每一条不说清楚就一定会搞错。

一句话说清楚，不解释不废话，但信息密度很高。

#### 好规则的特征

- 第一，**一句话能写完**。如果一条规则需要三行才能说清楚，要么拆成三条，要么说明它本身就太复杂了，应该放到代码注释或文档里。
	
- 第二，**Claude 靠自己推断不出来**。能从 pom.xml、代码结构、配置文件推断的都不用写。
	
- 第三，**有明确的行动指导**。“注意安全”是空话，“PathGuard 限制在项目根目录，禁止绝对路径逃逸和符号链接穿越”才是规则。

### 规范的CLAUDE.md结构

- **Commands**：构建、测试、lint 的具体命令
	
- **What This Is**：项目是什么，一句话说清楚
	
- **How It Runs**：运行机制，不是文档式的介绍，而是“改代码之前必须知道的事”
	
- **Key Concepts**：核心概念，3-5 个要点
	
- **Things That Will Bite You**：踩坑清单，列出会让人踩雷的细节
	
- **Code Conventions**：代码约定，只写和默认不一样的部分

### rules/ 目录

rules 目录下的每个 `.md` 文件都是一份独立的指令集。没有 `paths` 前置字段的 rules 文件，和 CLAUDE.md 一样在启动时加载。带 `paths` 前置字段的，只在 Claude 操作匹配路径的文件时才加载。  


### /init 和 /memory

Claude Code 有一套自动记忆系统。每个项目在 `~/.claude/projects/` 目录下都有一个 `MEMORY.md` 文件，Claude 会把跨会话需要记住的信息自动写到这里。下次启动会话时，MEMORY.md 的前 200 行会被自动加载进上下文。  

新项目第一天，跑一下 `/init`，Claude 会扫描仓库结构、分析依赖、读 README，生成一份基础的 CLAUDE.md。包含构建命令、测试命令、项目基本描述。


### Claude Code 配置体系

- **settings.json 管“强制”。** 权限控制、环境变量、MCP 服务器配置，这些放在 `.claude/settings.json` 里。没有商量余地，硬性约束。
	
- **hooks 管“自动化”。** 如果某件事必须每次都执行，别在 CLAUDE.md 里写“请记得格式化代码”，直接配一个 hook，每次编辑文件后自动跑 Prettier。hooks 由 harness 执行，不依赖 Claude 的“记忆”。  
	
- **rules/ 管“精准投放”。** 前面详细讲过了，按路径限定加载规则，节省指令预算。

### 实战模板

```
# CLAUDE.md
 
## Commands
 
- 构建：mvn clean package -DskipTests
- 测试：mvn test
- 单个测试：mvn test -Dtest=XxxTest
- 代码检查：mvn spotbugs:check
- 格式化：mvn spotless:apply
 
## What This Is
 
一句话说清楚项目是什么。
比如：PaiCLI 是一个纯 Java 实现的 Agent CLI，28K 行代码，不依赖 Spring AI/LangGraph4J。
 
## Architecture
 
- 入口：Main.java → CliCommandParser 分发命令
- Agent 循环：AgentLoop.java，工具注册在 ToolRegistry
- 记忆系统：MemoryManager，基于文件持久化
- 不要动 agent/core/ 下的接口定义，下游工具全部依赖它们
 
## Things That Will Bite You
 
- search_code 是 RAG 辅助，不是主要代码定位方式，优先用 glob → grep → read
- 改了命令入口 → 必须同步 Main.java + CliCommandParser + 测试 + 文档
- FileUtils 的路径处理已经做了沙箱限制，不要绕过它自己拼路径
- 测试里的 API Key 全部用 mock，禁止提交真实 Key
 
## Code Conventions
 
- 日志用 SLF4J，不用 System.out
- 异常不要吞掉，至少 log.warn
- 所有 public API 返回统一的 Result 包装类
- 新工具必须实现 Tool 接口并在 ToolRegistry 注册
 
## Don't
 
- 不要在业务代码里直接 new Thread，用 ExecutorService
- 不要改 .env.example 的格式，CI 依赖它
- 不要往 CLAUDE.md 里加“保持代码整洁”这种废话

```
