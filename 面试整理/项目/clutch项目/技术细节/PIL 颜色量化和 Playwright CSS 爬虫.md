
---

	逻辑主要在image_analysis.py中

## 完整链路

	入口函数是 analyze_image_for_spec(image_data_url)

### 解码图片

```Python
# 第 242-250 行
_, b64 = image_data_url.split(",", 1)       # 拆掉 "data:image/png;base64," 前缀
raw_bytes = base64.b64decode(b64)            # base64 → 原始字节
img = PILImage.open(io.BytesIO(raw_bytes))   # 字节 → PIL Image 对象
```

### 颜色量化核心 — `_extract_dominant_colors()`

```Python
# 第 109-118 行
thumb = img.convert("RGB").resize((150, 150), PILImage.LANCZOS)   # ① 缩到 150×150
quantized = thumb.quantize(                                       # ② 中位切分量化
    colors=12,                                                     #    压缩到 12 种颜色
    method=PILImage.Quantize.MEDIANCUT,                           #    算法：Median Cut
    dither=0                                                       #    不抖动，纯色块
)
palette_raw = quantized.getpalette()                               # ③ 取调色板 [R,G,B, R,G,B, ...]
```

> **Median Cut 算法原理**：把 RGB 三维颜色空间反复沿最长轴一劈两半，直到切出 12 个颜色盒子，每个盒子的中位数颜色就是该盒的代表色。比 K-Means 快，效果也不差。


### 统计覆盖率 & 分类

```Python
# 第 121-148 行
pixels = quantized.getdata()
# 数每个调色板索引出现多少次
for px in pixels:
    counts[px] = counts.get(px, 0) + 1

# 对每个颜色：
for idx, cnt in sorted(counts.items(), key=lambda kv: -kv[1]):  # 按覆盖率降序
    r, g, b = palette_raw[idx*3], palette_raw[idx*3+1], palette_raw[idx*3+2]
    pct = round(cnt / total * 100, 1)                            # 像素占比
    bucket, name = _classify_color(r, g, b)                      # 分类
```


### 颜色分类逻辑 — `_classify_color()`

```Python
# 第 83-100 行
h, s, v = colorsys.rgb_to_hsv(r/255, g/255, b/255)

if sat_pct < 12:           # 饱和度 < 12% → 无彩色
    if val_pct > 85:  return "neutral", "white"    # 极亮 → 白
    if val_pct < 18:  return "neutral", "black"    # 极暗 → 黑
    return "neutral", "grey"                        # 其他 → 灰

if sat_pct > 45 and val_pct > 30:
    return "primary", color_name                     # 高饱和 + 中高亮 → 主色
return "accent", color_name                          # 其他 → 强调色
```

### 分桶输出

```Python
# _build_palette() 第 157-175 行
return {
    "primary":   [hex, ...][:3],    # 最多 3 个主色
    "secondary": [hex, ...][:3],    # 剩余主色 + 前 2 个 accent
    "accent":    [hex, ...][:3],    # 最多 3 个强调色
    "neutral":   [hex, ...][:5],    # 最多 5 个中性色（白/黑/灰）
}
```



## Playwright CSS 变量爬虫

> [!NOTE] 核心
> 本质是利用 Playwright API打开真实网页，把页面里的 CSS 变量（`--primary`、`--spacing-sm` 这类）批量读出来，当成设计 token 用。解决静态抓包看不到最终值的问题

| API                                            | 用途                |
| ---------------------------------------------- | ----------------- |
| `chromium.launch()` / `browser.newPage()`      | 起浏览器、开页           |
| `page.goto(url, { waitUntil: 'networkidle' })` | 等样式/主题 JS 跑完      |
| `page.evaluate(...)`                           | 关键：在浏览器里执行 JS，读变量 |

	入口是 _extract_with_playwright(url)，CSS 变量提取逻辑在注入页面的 _EXTRACT_SCRIPT JS 脚本中，分两层

### 从样式表 CSSRule 直接读

```Python
// 第 95-106 行
for (const sheet of document.styleSheets) {
    for (const rule of sheet.cssRules) {
        // 只匹配 :root、html、body 这三个选择器
        if (rule.selectorText === ':root' || rule.selectorText === 'html'
            || rule.selectorText === 'body') {
            for (let i = 0; i < rule.style.length; i++) {
                const prop = rule.style[i];
                if (prop.startsWith('--'))                     // 只要 CSS 变量
                    cssVars[prop] = rule.style.getPropertyValue(prop).trim();
            }
        }
    }
}
```

	这层能抓到源代码里直接写的，比如 :root { --brand: #3B82F6; }。但如果 CSS 是跨域加载的（CDN 上的 Tailwind 等），sheet.cssRules 会抛异常被 catch(e) {} 吞掉，进入下一层。


### 从 computed style 兜底

```Python
// 第 108-122 行
const collectFromEl = (el) => {
    const style = getComputedStyle(el);
    for (let i = 0; i < style.length; i++) {
        const prop = style[i];
        if (prop.startsWith('--'))
            cssVars[prop] = style.getPropertyValue(prop).trim();
    }
};
collectFromEl(document.documentElement);   // <html>
collectFromEl(document.body);              // <body>
// 再往下探两层：body 的直接子元素和孙子元素
document.querySelectorAll('body > *, body > * > *').forEach(collectFromEl);
```

	`getComputedStyle()` 拿的是浏览器计算后的最终值，不管变量是写在源 CSS 里还是 Tailwind 生成的还是跨域 CDN 的，浏览器已经全部解析完毕了。

> Python 侧拿到 `cssVars` 后还要过滤一遍

```Python
# browser_extract.py 第 232-240 行
def is_brand_var(k):
    # 排除 Tailwind 工具类变量，只保留品牌/主题变量
    for prefix in ('--tw-', '--font-', '--leading-', '--container-',
                   '--animate-', '--blur-', '--tracking-', '--radius-',
                   '--ease-', '--text-'):
        if k.startswith(prefix):
            return False
    return True

filtered_vars = {k: v for k, v in cssVars.items() if is_brand_var(k)}
```


---

## 总结对比

![[Pasted image 20260723161916.png|783]]

	两个模块最终都汇入 LLM 的 system prompt，告诉模型"参考网站长这样、用这些颜色和变量，你生成的设计 spec 要匹配它"。