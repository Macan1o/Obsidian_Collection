
---

> [!NOTE] 文件总览
> 该文件主要是包括有图时的降级策略，和落盘生成HTML的三重防护机制，分别是链路A和链路B

## 总链路

```
用户 brief +（可选）参考图 / md / URL
        │
        ▼
_build_ui_generation_prompt
  · 拼 brief、layout、spec、DESIGN.md…
  · 仅当本次真发图时 image_attached=True，才写「对齐截图」
        │
        ▼
_try_llm_complete_vision ──▶ _llm_complete_vision     【切入 A】
  · 有图先多模态；失败则本地读图 + 纯文本
        │
        ▼
抽 HTML → _coerce_ui_html（多轮默认不许离线模板） 【切入 B1】
        │
        ├─ 有内容且像 vision 报错且有图
        │     → 丢图重建 prompt → 再 LLM → coerce（可模板）【A2】
        │
        ├─ 有内容但意图不匹配
        │     → correction prompt → 再 LLM → coerce       【B2】
        │
        └─ 无可见内容
              → compact 重试 → 再失败再试一轮
              → 仍空且有图 → 丢图 + 可模板                 【A2 末】
        │
        ▼
（可选）_design_review_and_improve → 再 _coerce_ui_html
        │
        ▼
返回 (html, reasoning, usage, last_fail)
        │
        ▼
generate_session / iterate 写盘前
  · 仍无可见内容 + 模型可用 → 失败，不写
  · 仍无可见内容 + 无 Key   → _fallback_ui_html
  · inject_unified_tailwind_config
  · 写入 screens/{sid}.html
```

### A 链路 · Vision 降级（管「图」）

```
image_data_url 有值？
  │
  ├─ 无 ──▶ 直接当纯文本 _llm_complete，不进 A
  │
  └─ 有
        │
        ▼
【A3】组 prompt 时 image_attached=True
  （丢图重试时改为 False，且不再提「对齐截图」）
        │
        ▼
【A1】_llm_complete_vision
  │  router.chat 多模态发图
  │
  ├─ 返回正常 ──▶ 交给总链路抽 HTML
  │
  ├─ 返回内容像 vision 错（_looks_like_vision_error）
  │     或 API 抛 vision 错（looks_like_vision_api_error）
  │         │
  │         ▼
  │     本地 image_analysis_prompt_fragment（OCR/色板）
  │         │
  │         ▼
  │     prefix + prompt → _llm_complete 纯文本
  │         │
  │         ▼
  │     交给总链路抽 HTML
  │
  └─（进入总链路后）HTML 本身仍是报错文？
        │
        ▼
【A2】丢图立刻重试
  · has_image=False, image_attached=False, image_data_url=None
  · 再建 prompt → LLM → _coerce_ui_html（可模板）
        │
        └─ 若后面多轮仍空且当初有图
              → 末尾再丢图纯文本一次（可模板）→ 结束 A
```

### B 链路 · HTML 质量门（管「页」）

> Design 写盘前防护在 `generator.py` 中，位于 LLM 返回 HTML 之后、写入 `screens/screen-1.html` 之前。

```
LLM raw 文本
        │
        ▼
【B1】_coerce_ui_html
  │  无 <html> → _shell_html 套壳
  │  无 charset → 插入 meta utf-8
  │
  ├─ _html_has_visible_content(当前) 真 → 采用
  ├─ 否则 fallback_html 可见 → 退回上一轮/传入稿
  ├─ 否则 allow_template_fallback → _fallback_ui_html
  └─ 否则 → ""（交给后面重试）
        │
        ▼
【B3 判定器】_html_has_visible_content（贯穿，非独立关）
  · 空 / 无 body / style·script 未闭合 → 否
  · 剥 script/style/注释后空或纯空 div → 否
  · 可见字 ≥2 或有语义标签 → 是
        │
        ▼
分支（在 _generate_ui_html 内，B1 之后）
  │
  ├─ 有可见内容
  │     │
  │     ▼
  │  【B2】_html_matches_brief_intent
  │     · _prompt_intent(brief) → login/shop/dashboard/music/landing/generic
  │     · _detect_html_intent(html) → 四类具体意图或 None
  │     · landing + 检出 login → 不通过
  │     · landing/generic（已过上条）→ 通过
  │     · 否则 detected==expected 或 None → 通过
  │           │
  │           ├─ 通过 →（可选 Review）→ 返回
  │           └─ 不通过
  │                 → _build_ui_correction_prompt
  │                 → 再 LLM → _coerce_ui_html(fallback=当前html)
  │                 → 有可见内容则替换
  │
  └─ 无可见内容
        → compact prompt 重试 → 再失败加重试说明再调
        → 每轮结果都再走 B1（coerce）
        → 仍空交回总链路（可能再走 A2 丢图 / 写盘前模板）
```