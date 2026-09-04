
---
```
用户输入：帮我分析近 14 天专注趋势，画个图
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
## 阶段 ①：工具调用获取真实数据（后端）                           │
│                                                                 │
│ data.ts: enrichContext()                                         │
│   1. 把用户问题 + 4 个工具定义发送给 DeepSeek                    │
│      - get_dashboard      → 今日总览                             │
│      - query_day_records  → 历史专注记录                         │
│      - query_scheduled_tasks → 预定任务                          │
│      - aggregate_day_records → 聚合统计                          │
│   2. DeepSeek 自主决定调用哪个工具、传什么参数                    │
│   3. executeTool() 查 Prisma，返回 JSON 结果                     │
│   4. 最多 3 轮工具调用，数据注入 system prompt                    │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼ "【以下是从数据库查询到的用户真实数据】[query_day_records] {...}"
       │
┌─────────────────────────────────────────────────────────────────┐
## 阶段 ②：DeepSeek 生成分析 + 图表 JSON（后端）                  │
│                                                                 │
│ route.ts:                                                       │
│   把 enriched 上下文 + system prompt 发送 DeepSeek               │
│   system prompt 里定义了 ```chart 代码块的格式规范                │
│                                                                 │
│   大模型输出：                                                    │
│   ```                                                            │
│   你近14天专注呈上升趋势...（文字分析）                            │
│                                                                 │
│   ```chart                                                      │
│   {                                                             │
│     "type": "line",                                             │
│     "title": "近14天专注时长趋势",                                │
│     "xAxis": ["06-03","06-04","06-05",...],                     │
│     "series": [{"name":"实际专注","data":[82,65,90,...]}]        │
│   }                                                             │
│   ```                                                            │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼  SSE 流返回前端（文字逐字推送）
       │
┌─────────────────────────────────────────────────────────────────┐
## 阶段 ③：afterResponse 解析图表（后端）                         │
│                                                                 │
│ data.ts: afterResponse()                                        │
│   parseChartBlock(aiContent)                                     │
│   1. 正则提取 ```chart ...``` 代码块                             │
│   2. JSON.parse → ChartConfig                                    │
│   3. 校验：type ∈ {line,bar,area}、必须有 title/xAxis/series     │
│   4. 返回 { chartCreated: chart }                                │
│                                                                 │
│ route.ts:                                                       │
│   → 写入 AiMessage.chartConfig（持久化到数据库）                  │
│   → send({ type: "chart", config })  通过 SSE 推给前端           │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼  SSE event: { type: "chart", config: {...} }
       │
┌─────────────────────────────────────────────────────────────────┐
## 阶段 ④：前端接收 → 状态更新 → 渲染                              │
│                                                                 │
│ useAiChat.ts:                                                   │
│   if (event.type === "chart") setStreamingChart(event.config)   │
│                                                                 │
│ ChatMessage.tsx:                                                │
│   chartConfig = msg.chartConfig ?? streamingChart                │
│   {chartConfig && <ChartBlock config={chartConfig} />}           │
│                                                                 │
│ ChartBlock.tsx:                                                 │
│   switch (type) {                                               │
│     "line" → <LineChart>  <Line .../>  </LineChart>             │
│     "bar"  → <BarChart>   <Bar .../>   </BarChart>              │
│     "area" → <AreaChart>  <Area .../>  </AreaChart>             │
│   }                                                             │
│   自动配色（PALETTE 5 色循环）、深色模式适配、                    │
│   自定义 Tooltip、响应式容器                                     │
└─────────────────────────────────────────────────────────────────┘
       │
       ▼
[用户看到：一篇带交互式图表的 AI 分析]
```

## 关键设计点

| **环节**                | **做什么**                              | **为什么**                        |
| --------------------- | ------------------------------------ | ------------------------------ |
| **数据不走 LLM 幻觉**       | `enrichContext` 先查数据库，把真实 JSON 注入上下文 | LLM 不知道你的专注数据，必须靠 tool call 获取 |
| **图表格式用代码块承载**        | ` ```chart ` + JSON                  | 不与正文 Markdown 冲突，正则提取干净        |
| **afterResponse 持久化** | 图保存进 `AiMessage.chartConfig` 字段      | 刷新页面后图表不丢                      |
| **SSE 实时推送**          | chart 事件和文字流同一个 SSE 连接               | 图表在文字完成后立刻出现                   |
| **流式状态下残留清理**         | 切会话/新建会话时 `setStreamingChart(null)`  | 避免旧图闪现到新会话                     |