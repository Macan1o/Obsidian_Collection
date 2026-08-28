
---

> [!NOTE]
> 整个链路时间线

```
时间线 ──────────────────────────────────────────────────────────────►

┌─ 阶段 A：请求前 (~200ms) ───────────────────────────────────────┐
│                                                                  │
│  route.ts 第 45-49 行：三路并行                                   │
│                                                                  │
│  ① loadMemories(userId)    → 长期记忆文本                         │
│  ② prisma.aiSession        → 查/建会话                            │
│  ③ skill.enrichContext()   → Skill 专属上下文注入                  │
│     ├─ search: Tavily 联网搜索                                    │
│     ├─ doc:    pgvector 语义搜索笔记 (semanticSearchNotes)         │
│     ├─ data:   DeepSeek 工具轮 → Prisma 查仪表盘                  │
│     ├─ task:   DeepSeek 工具轮 → Prisma 查任务/日历               │
│     └─ code:   Tree-sitter 自动解析 + 工具轮 + 历史消息 RAG       │
│                                                                  │
│  结果合并为 enrichContext 字符串                                   │
└──────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─ 阶段 B：拼装 messages ─────────────────────────────────────────┐
│                                                                  │
│  messages = [                                                    │
│    { role: "system", content: skill.prompt + memories }            │
│    ...history (从 AiMessage 表加载的历史对话)                     │
│    { role: "system", content: enrichContext }  ← 注入的上下文     │
│    { role: "user",   content: enrichedContent }  ← 含附件的用户消息 │
│  ]                                                               │
└──────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─ 阶段 C：DeepSeek SSE 流式 ─────────────────────────────────────┐
│                                                                  │
│  fetch(https://api.deepseek.com/chat/completions, { stream: true })│
│                                                                  │
│  每一帧 delta：                                                   │
│    reasoning_content → SSE { type: "thinking", text }             │
│    content          → SSE { type: "chunk",   text }              │
│                                                                  │
│  首帧还会发送：                                                   │
│    SSE { type: "session", sessionId }     ← 新会话时前端拿 ID     │
│    SSE { type: "searching", done: true }  ← 上下文已注入完毕      │
└──────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─ 阶段 D：保存 assistant 消息 ───────────────────────────────────┐
│                                                                  │
│  prisma.aiMessage.create({ role: "assistant", content, thinking })│
│  (code Skill 还会 fire-and-forget upsertAiMessageEmbedding)      │
│                                                                  │
│  prisma.aiSession.update({ updatedAt })                          │
└──────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─ 阶段 E：afterResponse 后置钩子 ────────────────────────────────┐
│                                                                  │
│  result = await skill.afterResponse(userId, userContent, aiContent)│
│                                                                  │
│  ┌─ doc Skill ──────────────────────────────────────────┐       │
│  │ detectSaveIntent() → LLM 意图识别                      │       │
│  │   三层过滤: 触发词 + 笔记名词 + 黑名单                  │       │
│  │     ↓ shouldSave=true                                  │       │
│  │ markdownToTiptap() → prisma.note.create()               │       │
│  │     ↓                                                  │       │
│  │ SSE { type: "note_created", note, notice }              │       │
│  │     ↓ 前端 useAiChat 收到后：                           │       │
│  │ useNotesStore.getState().addNote(note)  ← 直接插 Store  │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
│  ┌─ data Skill ─────────────────────────────────────────┐       │
│  │ parseChartBlock() → 提取 ```chart JSON 块              │       │
│  │     ↓                                                  │       │
│  │ prisma.aiMessage.update({ chartConfig })               │       │
│  │     ↓                                                  │       │
│  │ SSE { type: "chart", config }                          │       │
│  │     ↓ 前端: setStreamingChart(config)                  │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
│  ┌─ task Skill ─────────────────────────────────────────┐       │
│  │ parseTaskPlanBlock() → 提取 ```task_plan JSON          │       │
│  │     ↓                                                  │       │
│  │ applyTaskPlan() → 去重 → prisma.task.createMany()      │       │
│  │                → prisma.scheduledTask.createMany()      │       │
│  │     ↓                                                  │       │
│  │ SSE { type: "dashboard_sync", tasks, scheduledTasks }   │       │
│  │     ↓ 前端:                                            │       │
│  │ useDashboardStore.getState().syncFromAI({ tasks, ... }) │       │
│  └──────────────────────────────────────────────────────┘       │
│                                                                  │
│  SSE { type: "done" }  ← 所有后置处理完成                        │
└──────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─ 阶段 F：fire-and-forget ───────────────────────────────────────┐
│                                                                  │
│  void extractAndSaveMemories(userId, skillId, content, reply)    │
│    非阻塞，不延迟响应，静默失败                                    │
│    DeepSeek JSON 提取 → prisma.userMemory (最多 20 条)            │
└──────────────────────────────────────────────────────────────────┘
```

## SSE的职能

> 流式输出

`thinking → chunk → chunk → chunk → ...`

前端 `useAiChat` 逐字渲染，用户体验流畅。

> **后置事件（指令式同步）**

```
session → ... → done
                  │
         note_created / dashboard_sync / chart
                  │
             前端收到后立即调用 Zustand Store
             （不刷新页面，不重新 fetch，直接插入内存）
```

这些后置事件一定在 `done` 之前发出，前端用 `done` 作为"本轮完整结束"的信号来 `commitStreamingMessage`（把流式内容提交到消息列表）

## 链路总结

先通过 `enrichContext` 将文本内容注入上下文，然后再对上下文进行处理后拿到AI结果后通过后置钩子来对内容进行处理，通过SSE控制时序，打通前后端数据链路