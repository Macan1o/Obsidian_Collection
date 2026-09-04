
---
# DOC模糊概念 / 情感 / 知识搜索

需要用到Vector（向量字段）数据库字段「1024维」

> 首先就是关于涉及到**语义搜索**的组件

HNSW 索引

```
[前端: 传数据] 
       │
       ▼
┌────────────────── 后端 API 内部 ──────────────────┐
│ 1. 拿到数据 ──> 通过 Embedding API 转为向量数据      │
│ 2. 拿到 1024 维的向量数据 [0.012, -0.34, ...]       │
│ 3. 带着向量去查 pgvector 数据库                     │
│ 4. 数据库通过 HNSW 索引，通过userId过滤用户并算出最相似的笔记  │
└───────────────────────────────────────────────────┘
       │
       ▼
[前端: 收到一个干净的 JSON 数组进行渲染
```

> AI 引入笔记系统并实现“生成 → 分类 → 入库”的自动化工作流

核心能力

### 完整链路

​[[AI Skill 完整数据链路]]

| 文件                       | 路径                                         | 说明                                            |
| ------------------------ | ------------------------------------------ | --------------------------------------------- |
| `schema.prisma`          | `prisma/schema.prisma`                     | 加 `extensions=[vector]` + `Note.embedding` 字段 |
| `migration.sql`          | `prisma/migrations/<ts>/`                  | 建扩展、加列、建 HNSW 索引                              |
| `embedding.ts`           | `lib/embedding.ts`                         | 纯工具层：API 调用、文本提取、upsert、搜索                    |
| `types.ts`               | `app/api/ai/chat/skills/types.ts`          | `enrichContext` 签名加 `userId?`                 |
| `doc.ts`                 | `app/api/ai/chat/skills/doc.ts`            | enrichContext 接入语义搜索                          |
| `ai-chat-route.ts`       | `app/api/ai/chat/route.ts`                 | `enrichContext(content, userId)` 传 userId     |
| `notes-userId-route.ts`  | `app/api/notes/[userId]/route.ts`          | POST 后触发向量化钩子                                 |
| `notes-noteId-route.ts`  | `app/api/notes/[userId]/[noteId]/route.ts` | PATCH title/content 时触发向量化钩子                  |
| `backfill-embeddings.ts` | `scripts/backfill-embeddings.ts`           | 存量笔记一次性补充向量                                   |
| `llm.ts`                 | `lib/llm.ts`                               | 轻量 DeepSeek 封装，供 agent 调用                     |

```
用户发送："帮我写一篇 React Hooks 的笔记，存到【前端】文件夹"
    │
    ├─ enrichContext(content, userId)
    │       └─ semanticSearchNotes → 已有相关笔记注入上下文
    │
    ├─ DeepSeek 流式生成笔记内容（Markdown）
    │
    └─ afterResponse(userId, userContent, aiContent)
            │
            ├─ ① 关键词预检（"存到" 命中）→ callLLM 意图识别
            │       └─ { shouldSave: true, title: "React Hooks 笔记", folderName: "前端" }
            │
            ├─ ② resolveFolderId("前端") → 精确/模糊匹配 → folderId
            │
            ├─ ③ markdownToTiptap(aiContent) → Tiptap JSON
            │
            ├─ ④ prisma.note.create(...)    → 笔记写入数据库
            │
            └─ ⑤ void upsertNoteEmbedding(...) → 向量化（不阻塞）
```

## 编辑器内容渲染失败问题

[[AI生成入库内容渲染失败]]

### 完整数据流

```
用户发送 "帮我写一篇...存到【前端】文件夹"
    ↓
chunk × N  → 流式显示 AI 回复
    ↓
note_created → useNotesStore.addNote() → 笔记列表实时出现新条目
    ↓
done → commitStreamingMessage() → AI 消息入库
```

## LLM识别防误触问题

> 设定笔记意图规则

- **关键词必须和「笔记」绑定才触发**（单独 “保存、写一篇” 不触发）
    
- **加入黑名单过滤**（文案、图片、桌面、朋友圈… 直接拦截）
    
- **LLM 意图识别更严格**（只有明确要存笔记才返回 true）


## Token优化策略

三重策略大大降低Token的使用量，保底降低70%以上。

​[[三重Token节省策略]]

## RAG检索优化 

​[[RAG检索优化]]

---

# Data

### 数据统计 / 过滤 / 排序

需要用到`Text-to-SQL / 智能函数调用`以及普通字段（Date, Integer等）

> 涉及到的组件有

`data.ts` 、`data-tools.ts`、`chat/route.ts`、`types.ts`、`schema.prisma`

此外在`data.ts` 中查询需要借助多个表，`DashboardData`、`DayRecord`、`ScheduledTask`三个表

### Recharts图表生成 

​[[Recharts图表生成]]

---

# Code

**Embedding + 向量库 API**（RAG 查历史对话）

**代码解析**：`Tree-sitter`

或者

- **文件系统 API**（读写代码）
    
- **代码执行沙箱 API**（运行代码）
    
- **Shell 命令 API**（安装依赖、运行项目）
    
- **Git API**（版本管理）
    
- **搜索 API**（查最新知识）


---

# Task

可以跟dashboard的后端结合，让AI把任务拆解之后，告诉AI自己的空闲时间安排规划，让AI自动生成最合理的规划，并且添加到日历上
```tandem-comments
// Schema: { "<id>": { anchor:{exact,prefix,suffix,pos?}, status:open|resolved, thread:[{author,ts,text}], suggestion?:{replacement,author,ts,result?} } }
// Anchor = quote from the prose. To locate: search for "exact", disambiguate via prefix/suffix.
{
  "1514": {
    "anchor": {
      "exact": "HNSW 索引",
      "pos": 82,
      "prefix": "就是关于涉及到**语义搜索**的组件\n\n",
      "suffix": "\n\n```\n[前端: 传数据] \n   "
    },
    "status": "open",
    "thread": [
      {
        "author": "apple",
        "ts": "2026-08-17T07:15:21.573Z",
        "text": "分层可导航小世界\n\n把数据库里所有的向量编织成了一张多层的网络\n\n查找路径是从顶层往下跳的算法缩短到毫秒级。\n\n缺点就是吃内存，因为要把全库所有向量的朋友圈关系（图结构）全部织好，还要分层，这张图必须全部常驻在服务器的内存中。"
      }
    ]
  },
  "8611": {
    "anchor": {
      "exact": "核心能力",
      "pos": 517,
      "prefix": "> 涉及到AI Agent搭建的组件\n\n",
      "suffix": "\n\n### 完整链路\n\n​[[AI Sk"
    },
    "status": "open",
    "thread": [
      {
        "author": "apple",
        "ts": "2026-08-17T07:17:32.373Z",
        "text": "理解用户指令：帮我在【工作/学习】文件夹生成一篇【React Hooks总结】的笔记\n调用大模型生成笔记内容\n自动写入数据库 + 更新向量索引（复用你现有的向量逻辑）\n自动归类到目标文件夹\n可查询、可检索（复用你现有的语义搜索）"
      }
    ]
  }
}
```
