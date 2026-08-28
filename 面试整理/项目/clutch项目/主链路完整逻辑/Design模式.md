
---

> [!NOTE]
> 从用户在 Design 画布中输入自然语言提示词，到 LLM 两层流水线（spec → UI）、React Flow 画布渲染、迭代修改、最终 Send-to-Coding 交接的完整 API / 方法 / 数据流向

## Design模式链路

- 先走 PIL 颜色量化 + OCR 或 Playwright CSS 变量爬虫提取
	[[PIL 颜色量化和 Playwright CSS 爬虫]]
	
- 再通过LLM生成结构化的JSON文件，并生成标准格式的DESIGN，manifest 切至 `generating_ui
	
- 匹配布局最佳模式，few-shot HTML 模板和布局提示，拼接json + DESIGN + hint提交给LLM
	
- 经过三重安全网检测
	[[两层Design HTML防护机制]]
	
- 如果用户需要迭代，自动推断 modify/add 模式，覆盖HTML，用户可切换 Round Selector 对比历史版本。
	
- 确认后拼接JSON + DESIGN + prompt让Agent开始编码，完成视觉对齐

## 解决三个缺陷问题

- 缺陷一：PM 和开发者对页面认知不一致
	
- 缺陷二：LLM Coding 缺少视觉参考
	
- 缺陷三：Figma 门槛高、迭代慢

### Design模式解决方案

- 用户输入prompt + 参考图/URL
	
- LLM 生成 spec.json + HTML在画布上预览
	
- 对话迭代
	
- 一键 Send-to-Coding，0 返工，1 次对齐


---
## 完整链路追踪

```
用户输入 "做一个SaaS dashboard首页，参考Stripe的配色" 并按 Send
  │
  ▼
1. 前端: DesignWorkspace.tsx → DesignCanvasInner
   ├─ 用户填写 prompt + 选择 design_system="stripe"
   └─ WebSocket 发送 { action: "design_generate", prompt: "...", design_system: "stripe", device: "web" }
  │
  ▼
2. 后端: router.py → generate_session()
   ├─ 解析参数: user_prompt="做一个SaaS dashboard首页"
   │             device="web"
   │             design_system="stripe"
   │             reference_image=None
   │             reference_md=None
   │             reference_url=None
   │
   ├─ ensure_session_dir() → run_id = "rZh3xK2m"
   │   → .clutch/design/sessions/saas-dashboard-web__rZh3xK2m/
   │
   ├─ 写 manifest (phase="spec", status="crafting_spec")
   └─ 启动后台线程 _worker() → service.generate_session()
  │
  ▼
3. Phase 1: crafting_spec
   ├─ 无 reference_image / reference_md / reference_url
   ├─ _normalize_preset_id("stripe") → "stripe"
   ├─ presets/stripe.spec.json 存在 ✅
   │
   ├─ resolve_builtin_spec("stripe", prompt, "web")
   │   → 加载 spec.json (颜色 #635BFF / #0A2540 / ..., 字体 Soehne, ...)
   │   → 加载 stripe.md → DESIGN.md
   │   → enrich_fallback_spec() → 补充 12 节结构
   │   → source = "builtin_preset"
   │
   ├─ 0 token 消耗 (未调用 LLM) ✅
   │
   ├─ 写 spec.json
   ├─ 写 DESIGN.md
   └─ manifest.phase = "generating_ui"
  │
  ▼
4. Phase 2: generating_ui
   ├─ detect_layout_pattern("SaaS dashboard首页", "web")
   │   → pattern = "dashboard"
   │
   ├─ _build_ui_generation_prompt():
   │   · user_prompt
   │   · spec JSON (Stripe 品牌色/字体)
   │   · layout_hint: "app dashboard: persistent sidebar (240px), top bar..."
   │   · fewshot HTML: dashboard 模板
   │   · design_md: stripe.md (截断 ≤ 8K 字符)
   │   → 完整 prompt 约 4K tokens
   │
   ├─ LLM 调用 (_try_llm_complete_vision):
   │   → 生成 HTML (超时 90s)
   │
   ├─ 安全网:
   │   · _html_matches_brief_intent(): prompt_intent="dashboard" ✅
   │   · _html_has_visible_content(): 有 nav/sidebar/cards ✅
   │   · CLUTCH_DESIGN_REVIEW: 未启用，跳过
   │
   ├─ 写 screens/screen-1.html
   ├─ 写 thumbnail.svg
   │
   ├─ manifest.phase = "ready"
   ├─ manifest.screens = [{ id:"screen-1", status:"accepted", ... }]
   └─ manifest.process_log 追加 "UI generation complete · 3.8k tokens"
  │
  ▼
5. 前端轮询 (2s 间隔):
   GET /api/design/sessions/rZh3xK2m
   → manifest.phase == "ready"
   → 渲染: AgentLogCardNode + SpecCardNode + UiCardNode(screen-1.html)
   → UiCardNode 内 iframe 加载 screen-1.html
  │
  ▼
6. 用户查看预览，说一句话迭代:
   "把统计卡片的数量改成4个"
   → POST /api/design/sessions/rZh3xK2m/iterate
     { instruction: "把统计卡片的数量改成4个", mode: "auto", target_kind: "ui", target_id: "screen-1" }
  │
  ▼
7. _infer_iterate_mode("把统计卡片的数量改成4个"):
   ├─ add_keys 无匹配
   ├─ modify_keys: "改" ✅
   └─ mode = "modify"
  │
  ▼
8. modify 执行:
   ├─ 备份: screens/_r1/screen-1.html
   ├─ _build_ui_iteration_prompt():
   │   "Transform the HTML: 把统计卡片的数量改成4个
   │    CURRENT HTML: {screen-1.html}"
   ├─ LLM 生成修改后 HTML
   ├─ _html_essentially_same() → False (确实改了) ✅
   ├─ 覆盖 screens/screen-1.html
   └─ manifest.round_history 追加 { round:1, mode:"modify", ... }
  │
  ▼
9. 前端收到 design_iterate_complete → iframe 刷新
   用户点 Round Selector → 可对比 Round 1 vs Current
  │
  ▼
10. 用户确认无误，点击 [Send to Coding]:
    POST /api/design/sessions/rZh3xK2m/send-to-coding
    → coding_handoff_payload(rZh3xK2m)
    → {
        design_spec: { colors, typography, components, ... },
        design_md: "## Stripe Design System\n...",
        screens: [{ html: "<html>...tailwind...</html>", ... }],
        preview_html: "...",
        prompt: "做一个SaaS dashboard首页，参考Stripe的配色",
        design_system_name: "Stripe",
        run_id: "rZh3xK2m"
      }
    → Coding Agent 接收 → system prompt 注入 spec
    → 开始按"Stripe 风格 + 4 统计卡片 dashboard"编码
```

## 全局单例与组件关系图

```
┌──────────────────────────────────────────────────────────────┐
│                 Design 模式核心组件关系                        │
│                                                              │
│  router.py (FastAPI 端点)                                     │
│    ├─ ensure_session()                                       │
│    ├─ generate_session() → 后台线程 _worker()                 │
│    ├─ iterate_session()                                      │
│    ├─ get_session() → 前端轮询                                │
│    └─ coding_handoff_payload() → send-to-coding              │
│                                                              │
│  generator.py (核心引擎, ~2600 行)                             │
│    ├─ generate_session() — 单次完整生成                        │
│    │   ├─ Phase 1: crafting_spec                             │
│    │   │   ├─ _normalize_preset_id()                         │
│    │   │   ├─ resolve_builtin_spec() → presets/{id}.spec.json│
│    │   │   └─ image_analysis.analyze_image_for_spec()        │
│    │   └─ Phase 2: generating_ui                             │
│    │       ├─ detect_layout_pattern() → 11 种布局             │
│    │       ├─ _build_ui_generation_prompt()                  │
│    │       ├─ _html_matches_brief_intent() (意图检测)         │
│    │       ├─ _html_has_visible_content() (空输出检测)         │
│    │       └─ _design_review_and_improve() (可选)             │
│    │                                                         │
│    ├─ iterate_session() — 迭代修改                            │
│    │   ├─ _infer_iterate_mode() (modify/add 自动推断)         │
│    │   ├─ _build_ui_iteration_prompt()                       │
│    │   └─ _html_essentially_same() (空迭代保护)               │
│    │                                                         │
│    └─ coding_handoff_payload() — 编码交接                     │
│                                                              │
│  依赖的模块:                                                  │
│    ├─ session_store.py — 文件 CRUD + 原子写入                 │
│    ├─ browser_extract.py — Playwright CSS 爬虫                │
│    ├─ image_analysis.py — PIL 颜色量化 + OCR                  │
│    ├─ layout_patterns.py — 11 种布局检测 + fewshot            │
│    ├─ layout.py — 画布坐标常量                                │
│    ├─ builtin_presets.py — 80+ 品牌预设                       │
│    └─ presets/*.md + presets/*.spec.json — 预设数据文件       │
│                                                              │
│  前端 (React):                                               │
│    ├─ DesignWorkspace.tsx — React Flow 画布主组件              │
│    └─ designWorkspaceUtils.ts — 布局/交互/类型定义             │
│                                                              │
│  6 种节点:                                                    │
│    AgentLogCardNode · SpecCardNode · UiCardNode               │
│    RefCardNode · MdDocCardNode · UrlCardNode                  │
└──────────────────────────────────────────────────────────────┘
```


