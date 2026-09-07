
---

代码「主要在下面的部分」

```
/* ========== 1. 行间距 line-height（核心） ========== */
/* 编辑模式（源码视图）行距 */
.cm-s-obsidian .cm-line {
  line-height: 1.7 !important; /* 1.2默认，推荐1.6~1.8 */
}
/* 阅读预览模式行距 */
.markdown-preview-view p {
  line-height: 1.7 !important;
  margin-bottom: 1em; /* 段落之间空行距离 */
}
/* ========== 2. 文字左右间距（字间距 letter-spacing） ========== */
/* 编辑视图文字间距 */
.cm-s-obsidian {
  letter-spacing: 0.03em; /* 0默认，0.02~0.05舒适阅读 */
}
/* 预览视图文字间距 */
.markdown-preview-view {
  letter-spacing: 0.03em;
}
/* ========== 3. 笔记页面整体左右边距（文字离窗口留白） ========== */
/* 编辑模式左右边距 */
.markdown-source-view.is-readable-line-width .CodeMirror {
  padding-left: 40px !important;
  padding-right: 40px !important;
}
/* 预览模式左右边距 */
.markdown-preview-view.is-readable-line-width .markdown-preview-sizer {
  padding-left: 40px !important;
  padding-right: 40px !important;
}

/* ========== Notebook Navigator 工具栏透明【修复版】 ========== */
/* 变量定义 */
body{
  --nn-theme-list-header-bg: transparent !important;
}

/* ✅真正的头部容器：nn‑pane‑header */
.nn-list-pane-chrome .nn-pane-header {
  background-color: transparent !important;
}

/* 按钮容器 nn‑header‑actions */
.nn-pane-header .nn-header-actions {
  background-color: transparent !important;
}

/* 各个图标按钮 */
.nn-pane-header .nn-icon-button {
  background-color: transparent !important;
}
/* 激活状态按钮高亮 */
.nn-pane-header .nn-icon-button.is-active {
  background-color: var(--background-modifier-hover) !important;
}

```