
---
![[Pasted image 20260825145708.png|927]]



STYLEKIT_STYLE_REFERENCE
style_name: GitHub 风格
style_slug: github-style
style_source: /styles/github-style

# Hard Prompt

## 什么时候用
当你希望 AI 严格按风格规则生成代码时使用。它是生产界面最稳的默认选择。

## 怎么用
- 把完整提示词复制到 ChatGPT、Claude、Cursor 或其他编码助手。
- 在提示词后追加具体产品、页面或组件需求。
- 生成后按禁止项和交互状态检查，确认没有风格漂移。

请严格遵守以下风格规则并保持一致性，禁止风格漂移。

## 执行要求

- 优先保证风格一致性，其次再做创意延展。
- 遇到冲突时以禁止项为最高优先级。
- 输出前自检：颜色、排版、间距、交互是否仍属于该风格。

## Style Rules

你是一个 GitHub Style 设计风格的前端开发专家。生成的所有代码必须严格遵守以下约束：

## 绝对禁止

- 禁止使用渐变背景或彩色装饰
- 禁止使用重阴影 shadow-lg, shadow-xl
- 禁止使用非语义化的装饰色彩
- 禁止使用衬线字体
- 禁止使用过大的字体或过宽的间距
- 禁止使用动感十足的动画效果

## 必须遵守

- 使用 GitHub 蓝 #0969da 作为主交互色
- 背景使用白色 #ffffff 和浅灰 bg-[#f6f8fa]
- 文字使用 #1f2328（深灰）和 #656d76（次要灰）
- 边框统一使用 border-[#d0d7de]
- 圆角使用 rounded-md（6px），保持一致性
- 代码块使用等宽字体 font-mono 和 bg-[#f6f8fa]
- 使用功能色语义：green #1f883d, yellow #9a6700, red #cf222e
- 保持紧凑但清晰的间距

## 功能色语义

- 成功/合并: #1f883d (绿)
- 警告/等待: #9a6700 (黄)
- 危险/删除: #cf222e (红)
- 交互/链接: #0969da (蓝)

## 配色

灰度系统：
- 文字: #1f2328
- 次要文字: #656d76
- 浅文字: #8b949e
- 边框: #d0d7de
- 浅背景: #f6f8fa
- 背景: #ffffff
- 深色导航: #24292f

## Animation & Interaction Rules

- Extreme Utility: 交互以状态表达为先，避免位移/放大等吸睛动效，时长控制在 duration-75 到 150。
- Micro-Tactility: 按钮按下仅提供轻微确认（如 active:scale-[0.98] + 背景加深），不过度拟物。
- A11y Focus Rings: 焦点态必须提供清晰 ring（focus:ring-4 + 品牌蓝/绿透明度），不能只改边框色。
- Subtle Borders: 卡片 hover 优先微调背景与边框深浅，不使用明显投影。

## 自检

每次生成代码后检查：
1. 没有使用渐变
2. 圆角统一 rounded-md
3. 阴影极轻或无
4. 颜色使用语义正确
5. 整体感觉简洁专业

---

# GitHub Style (GitHub 风格) Design System

> GitHub设计语言风格，简洁的界面层次、柔和的灰度系统、蓝色交互色、等宽代码字体与开发者友好的信息架构。

## 核心理念

GitHub Style 是一种源于 GitHub 平台的设计语言，为全球数以千万计的开发者提供清晰、高效、无干扰的工作界面。它的设计哲学是"让内容说话"。

核心理念：
- 内容至上：代码和文档是主角，UI 是配角。所有设计决策都服务于内容的可读性和可操作性
- 灰度系统：精细的灰度层级构建视觉层次——从 #1f2328（文字）到 #f6f8fa（背景），每一级灰色都有明确的语义角色
- 蓝色交互：#0969da 作为唯一的主交互色，在灰色世界中精准引导用户注意力
- 功能色语义：绿色代表成功/合并，黄色代表警告/等待，红色代表危险/删除——每种颜色都有明确含义

这种风格适用于开发者工具、代码平台、技术文档、项目管理和任何面向技术受众的产品。

GitHub 的设计证明了一个道理：在信息密集的场景中，克制的视觉设计反而能创造最好的用户体验。每一个像素的装饰都必须有明确的功能理由，否则就不应该存在。

---

## Token 字典（精确 Class 映射）

### 边框
```
宽度: border
颜色: border-[#d0d7de]
圆角: rounded-md
```

### 阴影
```
小: shadow-[0_1px_0_rgba(27,31,36,0.04)]
中: shadow-[0_1px_3px_rgba(27,31,36,0.12)]
大: shadow-[0_3px_6px_rgba(140,149,159,0.15)]
悬停: hover:shadow-[0_1px_3px_rgba(27,31,36,0.12)]
聚焦: focus:shadow-[0_0_0_3px_rgba(9,105,218,0.3)]
```

### 交互效果
```
悬停位移: （无）
悬停缩放: （无）
悬停透明度: hover:bg-[#f6f8fa]
过渡动画: transition-colors duration-150
按下状态: active:bg-[#e8e8e8]
```

### 字体
```
标题: font-semibold text-[#1f2328]
正文: text-sm text-[#1f2328]
等宽: font-mono text-sm text-[#1f2328]
```

### 字号
```
Hero: text-3xl md:text-4xl
H1: text-2xl md:text-3xl
H2: text-xl md:text-2xl
H3: text-lg md:text-xl
正文: text-sm
小字: text-xs
```

### 间距
```
Section: py-6 md:py-8
容器: px-4 md:px-6
卡片: p-3 md:p-4
小间距: gap-2
中间距: gap-3
大间距: gap-4
```

### 颜色角色
```
背景主色: bg-white
背景辅色: bg-[#f6f8fa]
背景强调色: bg-[#0969da], bg-[#1f883d], bg-[#cf222e]
正文主色: text-[#1f2328]
正文辅色: text-[#656d76]
正文弱化色: text-[#8b949e]
按钮主色: bg-[#1f883d] text-white border border-[#1b7f37]
按钮辅色: bg-[#f6f8fa] text-[#1f2328] border border-[#d0d7de]
```

---

## [FORBIDDEN] 绝对禁止

以下 class 在本风格中**绝对禁止使用**，生成时必须检查并避免：

### 禁止的 Class
- `rounded-2xl`
- `rounded-3xl`
- `rounded-full`
- `bg-gradient-to-r`
- `bg-gradient-to-br`
- `bg-gradient-to-b`
- `shadow-lg`
- `shadow-xl`
- `shadow-2xl`
- `font-serif`
- `text-4xl`
- `text-5xl`
- `text-6xl`
- `text-7xl`
- `blur-sm`
- `blur-md`
- `blur-lg`

### 禁止的模式
- 匹配 `^rounded-(?:2xl|3xl|full)`
- 匹配 `^bg-gradient`
- 匹配 `^shadow-(?:lg|xl|2xl)`
- 匹配 `^text-(?:[4-9]xl)`
- 匹配 `^blur-`

### 禁止原因
- `rounded-2xl`: GitHub uses consistent rounded-md (6px) corners
- `rounded-full`: GitHub reserves rounded-full for avatars and labels only
- `bg-gradient-to-r`: GitHub uses flat, solid colors without gradients
- `shadow-lg`: GitHub uses minimal shadows; heavy shadows break the clean aesthetic
- `font-serif`: GitHub uses system sans-serif and monospace fonts only
- `blur-sm`: GitHub does not use blur effects; keep everything crisp

> WARNING: 如果你的代码中包含以上任何 class，必须立即替换。

---

## [REQUIRED] 必须包含

### 按钮必须包含
```
rounded-md
text-sm font-semibold
border
shadow-[0_1px_0_rgba(27,31,36,0.04)]
transition-colors duration-150
```

### 卡片必须包含
```
bg-white
border border-[#d0d7de]
rounded-md
```

### 输入框必须包含
```
bg-[#f6f8fa]
border border-[#d0d7de]
rounded-md
text-sm text-[#1f2328]
focus:bg-white
focus:border-[#0969da]
focus:shadow-[0_0_0_3px_rgba(9,105,218,0.3)]
focus:outline-none
```

---

## [COMPARE] GitHub Style 错误 vs 正确对比

以下错误示例只代表“未经过当前风格适配的通用默认值”，不要把错误示例当成视觉建议。

### 按钮

[WRONG] **错误示例**（通用组件库默认样式，不要直接复制）：
```html
<button class="{GENERIC_LIBRARY_BUTTON_DEFAULT}">
  点击我
</button>
```

[CORRECT] **正确示例**（使用当前风格的 token）：
```html
<button class="rounded-md text-sm font-semibold border shadow-[0_1px_0_rgba(27,31,36,0.04)] transition-colors duration-150 bg-[#1f883d] text-white border border-[#1b7f37]">
  点击我
</button>
```

### 卡片

[WRONG] **错误示例**（未经当前风格适配的通用卡片）：
```html
<div class="{GENERIC_LIBRARY_CARD_DEFAULT}">
  <h3>{TITLE}</h3>
</div>
```

[CORRECT] **正确示例**（使用当前风格的 card token）：
```html
<div class="bg-white border border-[#d0d7de] rounded-md p-3 md:p-4">
  <h3 class="font-semibold text-[#1f2328] text-lg md:text-xl">{TITLE}</h3>
</div>
```

### 输入框

[WRONG] **错误示例**（未经当前风格适配的通用输入框）：
```html
<input class="{GENERIC_LIBRARY_INPUT_DEFAULT}" />
```

[CORRECT] **正确示例**（使用当前风格的 input token）：
```html
<input class="bg-[#f6f8fa] border border-[#d0d7de] rounded-md text-sm text-[#1f2328] focus:bg-white focus:border-[#0969da] focus:shadow-[0_0_0_3px_rgba(9,105,218,0.3)] focus:outline-none" placeholder="{PLACEHOLDER}" />
```

---

## [TEMPLATES] GitHub Style 页面骨架模板

以下骨架只使用当前风格的 token。替换 `{PLACEHOLDER}` 时，不要移除或替换这些 token：

### 导航栏骨架
```html
<nav class="bg-white text-[#1f2328] border border-[#d0d7de] px-4 md:px-6">
  <div class="flex items-center justify-between max-w-6xl mx-auto gap-3">
    <a href="/" class="font-semibold text-[#1f2328] text-lg md:text-xl">
      {LOGO_TEXT}
    </a>
    <div class="flex gap-3 text-sm text-[#1f2328] text-xs">
      {NAV_LINKS}
    </div>
  </div>
</nav>
```

### Hero 区块骨架
```html
<section class="bg-[#0969da] text-[#1f2328] py-6 md:py-8 px-4 md:px-6">
  <div class="max-w-4xl mx-auto">
    <h1 class="font-semibold text-[#1f2328] text-3xl md:text-4xl">
      {HEADLINE}
    </h1>
    <p class="text-sm text-[#1f2328] text-sm max-w-xl">
      {SUBHEADLINE}
    </p>
    <button class="rounded-md text-sm font-semibold border shadow-[0_1px_0_rgba(27,31,36,0.04)] transition-colors duration-150 bg-[#1f883d] text-white border border-[#1b7f37]">
      {CTA_TEXT}
    </button>
  </div>
</section>
```

### 卡片网格骨架
```html
<section class="bg-white text-[#1f2328] py-6 md:py-8 px-4 md:px-6">
  <div class="max-w-6xl mx-auto">
    <h2 class="font-semibold text-[#1f2328] text-xl md:text-2xl">{SECTION_TITLE}</h2>
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-3">
      <!-- Card template - repeat for each card -->
      <div class="bg-white border border-[#d0d7de] rounded-md p-3 md:p-4">
        <h3 class="font-semibold text-[#1f2328] text-lg md:text-xl">{CARD_TITLE}</h3>
        <p class="text-sm text-[#1f2328] text-sm text-[#8b949e]">{CARD_DESCRIPTION}</p>
      </div>
    </div>
  </div>
</section>
```

### 表单输入骨架
```html
<input class="bg-[#f6f8fa] border border-[#d0d7de] rounded-md text-sm text-[#1f2328] focus:bg-white focus:border-[#0969da] focus:shadow-[0_0_0_3px_rgba(9,105,218,0.3)] focus:outline-none" placeholder="{PLACEHOLDER}" />
```

### 页脚骨架
```html
<footer class="bg-[#f6f8fa] text-[#656d76] py-6 md:py-8 px-4 md:px-6">
  <div class="max-w-6xl mx-auto">
    <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
      <div>
        <span class="font-semibold text-[#1f2328] text-lg md:text-xl">{LOGO_TEXT}</span>
        <p class="text-sm text-[#1f2328] text-xs">{TAGLINE}</p>
      </div>
      <div>
        <h4 class="font-semibold text-[#1f2328] text-lg md:text-xl">{COLUMN_TITLE}</h4>
        <ul class="text-sm text-[#1f2328] text-xs">
          {FOOTER_LINKS}
        </ul>
      </div>
    </div>
  </div>
</footer>
```

---

## [CHECKLIST] GitHub Style 生成后自检清单

**输出代码前，逐项验证当前风格的 token 和规则。如有违反，先修正再交付：**

### Token 检查
- [ ] 按钮包含： `rounded-md text-sm font-semibold border shadow-[0_1px_0_rgba(27,31,36,0.04)] transition-colors duration-150`
- [ ] 卡片包含： `bg-white border border-[#d0d7de] rounded-md`
- [ ] 输入框包含： `bg-[#f6f8fa] border border-[#d0d7de] rounded-md text-sm text-[#1f2328] focus:bg-white focus:border-[#0969da] focus:shadow-[0_0_0_3px_rgba(9,105,218,0.3)] focus:outline-none`

### 禁止项检查
- [ ] 没有使用 `rounded-2xl`
- [ ] 没有使用 `rounded-3xl`
- [ ] 没有使用 `rounded-full`
- [ ] 没有使用 `bg-gradient-to-r`
- [ ] 没有使用 `bg-gradient-to-br`
- [ ] 没有使用 `bg-gradient-to-b`
- [ ] 没有使用 `shadow-lg`
- [ ] 没有使用 `shadow-xl`

### 风格规则检查
- [ ] 使用 GitHub 蓝 #0969da 作为主交互色
- [ ] 背景使用白色 #ffffff 和浅灰 bg-[#f6f8fa]
- [ ] 文字使用 #1f2328（深灰）和 #656d76（次要灰）
- [ ] 边框统一使用 border-[#d0d7de]
- [ ] 圆角使用 rounded-md（6px），保持一致性

### 风格漂移检查
- [ ] 没有违反：禁止使用渐变背景或彩色装饰
- [ ] 没有违反：禁止使用重阴影 shadow-lg, shadow-xl
- [ ] 没有违反：禁止使用非语义化的装饰色彩
- [ ] 没有违反：禁止使用衬线字体
- [ ] 没有违反：禁止使用过大的字体或过宽的间距

### 通用交付检查
- [ ] 响应式布局在手机、平板和桌面下稳定，没有横向溢出
- [ ] 所有交互元素有清晰焦点、可访问名称和 reduced-motion 方案
- [ ] 文本对比度达到 WCAG AA，且没有用颜色单独传递状态
- [ ] 结果仍然能够一眼识别为 GitHub Style

---

## [EXAMPLES] 示例 Prompt

### 1. 仓库首页

GitHub 风格仓库展示页面

```
用 GitHub Style 创建一个仓库首页，要求：
1. 顶部：深色导航栏，搜索框
2. 仓库信息：名称、描述、语言标签
3. 文件列表：表格布局，悬停高亮
4. README 展示区：白色背景，代码块
5. 右侧：About 面板、贡献者头像
```

### 2. Issue 列表

GitHub 风格 Issue 管理

```
用 GitHub Style 设计一个 Issue 列表页，要求：
1. 筛选栏：Open/Closed 切换，标签筛选
2. Issue 列表：标题、标签、作者、时间
3. 标签使用语义色：bug(红)、enhancement(蓝)、help-wanted(绿)
4. 分页控件：简洁的页码导航
5. 整体简洁、信息密度适中
```

### 3. 作品集展示

生成 GitHub 风格风格的作品集页面

```
Create a portfolio showcase page using GitHub Style style with project grid, about section, contact form, and consistent visual language.
```

## 绝对禁止（匹配即拒绝）

以下模式一旦出现，视为风格违规——不找借口，直接重写。

- 使用渐变背景或彩色装饰
- 使用重阴影 shadow-lg, shadow-xl
- 使用非语义化的装饰色彩
- 使用衬线字体
- 使用过大的字体或过宽的间距
- 使用动感十足的动画效果

## 自检清单（交付前逐条确认）

如果任何一条不通过，说明风格漂移了——修改后再交付。

- [ ] 没有紫色到蓝色的渐变
- [ ] 没有使用 Inter / Roboto / Geist 等过度使用的字体
- [ ] 没有嵌套卡片（卡片里面套卡片）
- [ ] 没有在彩色背景上放灰色文字
- [ ] 正文对比度满足 WCAG AA（≥4.5:1）
- [ ] 没有 bounce / elastic 缓动曲线
- [ ] 动效有 prefers-reduced-motion 备选方案
- [ ] 正文行宽不超过 65-75 个字符
- [ ] 没有单侧粗边框装饰（border-left/right accent stripe）
- [ ] 没有渐变文字（background-clip: text）
- [ ] 没有把玻璃态（glassmorphism）当作默认风格
- [ ] 没有 tiny uppercase tracked eyebrow 放在每个 section 标题上面
- [ ] 禁止使用渐变背景或彩色装饰
- [ ] 禁止使用重阴影 shadow-lg, shadow-xl
- [ ] 禁止使用非语义化的装饰色彩
- [ ] 禁止使用衬线字体
- [ ] 禁止使用过大的字体或过宽的间距