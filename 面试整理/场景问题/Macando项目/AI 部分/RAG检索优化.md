
---

> [!NOTE] 逻辑预览
> query/document 分离 + 归一化 + 相似度阈值的体现

## 完整链路

- 用户原始 query
	 
- 预处理：`normalizeText(query)`得到归一化 query 字符串
	
- **两路并行检索**
    
    - A 路【向量检索，原有逻辑不变】
        
        - 归一化 query 调用`getEmbedding(..., text_type="query")`得到 query 向量
	        
        - pgvector：余弦相似度 >0.28 阈值过滤，取 top‑10 向量结果列表
        
    - B 路【新增 BM25 关键词检索】
        
        - **原始 query 做中文分词**（不要 normalize 清洗过度，保护专有名词）
	        
        - BM25 索引计算关键词得分，返回 top‑10 关键词匹配结果列表
     
    
- **RRF 倒数排名融合** <mark style="background:#d3f8b6">「RRF 是根据两种检索链路的排名，排名越靠前分数越高，最后通过统一得分体系得出最新的文档名次」</mark>
    
    - 输入向量排名列表 + BM25 排名列表
	    
    - 不依赖绝对分数，只靠文档排名重新算分、合并去重，输出最终候选笔记
    
- 将融合后的笔记，组装 XML 上下文注入 LLM，生成答案


## **1. query / document 分离**

百炼 `text-embedding-v4` 支持 `parameters.text_type`，同一套模型对「长文档」和「短查询」用不同特征空间，RAG 里常见做法是 **入库用 document、检索用 query**。这是典型的「**非对称 Embedding** RAG」，模型会根据不同模式进行编码向量，用来弥合【用户问句】和【知识库切片】的向量分布偏移。

> 后续如果成为一个生产级项目的话则需要引入下列业务逻辑

- 文档加载、分块、清洗
	
- Query 改写、多 query 生成、HyDE、子问题拆解
	
- 向量库写入、ANN 检索、混合检索
	
- Rerank 重排、结果过滤

### BM25 混合检索算法

[[BM25 混合检索算法]]


### **入库：**`document`**（默认）**

```javascript
embedding.tsLines 199-203

// ── 调用 Embedding API ────────────────────────────────────────
// 哈希用 normalized（忽略格式），Embedding 用原始文本（保留完整语义）
const embedText = rawText.trim() || title.trim() || "untitled";

// 这里的 getEmbedding 默认不传第二个参数，即自动使用 "document" 模式
const vector  = await getEmbedding(embedText);

// getEmbedding 第二个参数默认是 "document"，请求体里会带上：
embedding.tsLines 107-111
    body: JSON.stringify({
model: EMBEDDING_MODEL,
input: { texts: [text.slice(0, 8000) || 'empty'] },
parameters: { text_type: type }, // 动态匹配百炼推荐的特征提取类型
}),
```

笔记 PATCH/创建、doc 自动保存后 `upsertNoteEmbedding`，走的都是这条 **document** 路径。

### **检索：**`query`

用户一句话（doc Skill 里就是 `content`）先归一化，再显式传 `"query"`，和库里 **document 向量** 做相似度比较

### **业务入口**

```JavaScript
doc.tsLines 344-347

async enrichContext(content: string, userId?: string): Promise<string> {
if (!userId || !content.trim()) return "";
    try {
const notes = await semanticSearchNotes(userId, content, TOP_K, SIMILARITY_THRESHOLD);
```

`content` 是用户当前问题，在 `semanticSearchNotes` 内部走 **query** 向量化；库里的 `Note.embedding` 是 **document** 向量化，二者在 pgvector 里用 `<=>`（余弦距离）对齐。

#### **两个向量不是同一套生成模式，但处在同一个向量空间，可以直接算余弦距离**。

---

## **2. 归一化（normalizeText）**

> `normalizeText` 在 RAG 里主要服务**检索侧查询**，文本预处理，字符串层面清洗。并和入库侧的 **哈希/缓存** 共用同一套规则。

```javascript
embedding.tsLines 77-84

export function normalizeText(raw: string): string {
return raw
.replace(/[，。！？、；：""''【】《》（）「」…—·～]/g, " ")  // 中文标点
.replace(/[^\u4e00-\u9fff\w\s]/g, " ")                       // 其余非语义字符
.replace(/\s+/g, " ")
.trim()
.toLowerCase();
}
```

### **在检索里的体现**

```javascript
embedding.tsLines 243-252

export async function semanticSearchNotes(
userId:    string,
query:     string,
  ...

): Promise<SemanticNote[]> {
const normalized = normalizeText(query);
const vector     = await getEmbedding(normalized, "query");
```

作用：用户输入「帮我总结一下 **项目架构？**」和「项目架构」在标点、空格、大小写上更一致，减少 **query 向量抖动**。

### **在入库里的体现（和 RAG 相关但不完全相同）**

入库时：

- **调 API 做向量**：用 `rawText`（`buildNoteText` 的原始标题+正文），保留完整语义；
    
- **哈希 /** `plainText` **缓存**：用 `normalizeText(rawText)`。


```javascript
embedding.tsLines 162-163

const normalized   = normalizeText(rawText) || title.trim().slice(0, 100);
const newHash      = hashText(normalized);
```

注释也写明了分工：

```
embedding.tsLines 199-201

// 哈希用 normalized（忽略格式），Embedding 用原始文本（保留完整语义）
```

所以面试时可以这样说：**归一化主要用于检索 query 的稳定化 + 入库变更检测「省token」；真正写入向量的文档文本仍用原始内容，避免过度清洗损失语义。**

---

## **3. 相似度阈值（threshold）**

阈值在 **两层** 出现：函数默认值 + doc Skill 业务覆盖。

### **SQL 层：过滤低相关结果**

```JavaScript
embedding.tsLines 255-268

const rows = await prisma.$queryRaw<SemanticNote[]>`
    SELECT
      ...

      (1 - (embedding <=> ${vecLit}))::float AS similarity
    FROM   "Note"
    WHERE  "userId"   = ${userId}
      AND  embedding  IS NOT NULL
      AND  (1 - (embedding <=> ${vecLit})) > ${threshold}
    ORDER  BY embedding <=> ${vecLit}
    LIMIT  ${topK}

`;
```

- `<=>`：pgvector 余弦**距离**；
    
- `1 - distance`：转成 **相似度**（0～1）；
    
- `> threshold`：只保留相似度高于阈值的笔记；
    
- `ORDER BY ... LIMIT topK`：在通过阈值的结果里取 Top-K。
    

`semanticSearchNotes` 默认 `threshold = 0.25`。

### **业务层：doc Skill 提高到 0.28**

```JavaScript
doc.tsLines 32-33

const SIMILARITY_THRESHOLD = 0.28;
const TOP_K = 10;
doc.tsLines 347-347
const notes = await semanticSearchNotes(userId, content, TOP_K, SIMILARITY_THRESHOLD);
```

比库函数默认 **0.25 更严**，少注入弱相关笔记，降低 LLM 上下文噪声。命中结果还会在日志和注入 XML 里带上 `similarity`：

```JavaScript
doc.tsLines 44-44

<note index="${i + 1}" id="${n.id}" similarity="${n.similarity.toFixed(2)}">
```

---

## **实际应用场景：用户问一句 doc 问题时发生了什么**

```
chat/route.ts → docSkill.enrichContext(content, userId)
    ↓
semanticSearchNotes(userId, content, 10, 0.28):
    ├─ 向量支路（原有不变）
    │   ├─ normalizeText(content)
    │   ├─ getEmbedding(..., "query")
    │   └─ pgvector SQL similarity >0.28，取top10 → vectorResultList
    │
    ├─ BM25支路【并行】
    │   ├─ 原始content中文分词（不normalize）
    │   └─ 查询内存BM25索引 top10 → bm25ResultList
    │
    ├─ RRF倒数排名融合(vectorResultList, bm25ResultList)
    └─ 返回融合后的笔记列表

enrichContext 将结果格式化为 <notes>…</notes> 送入DeepSeek

入库：upsertNoteEmbedding → getEmbedding(embedText, document) 写入pgvector
```

---

## **面试一句话版**

> _「入库和检索用了百炼的_ **_document/query 非对称 Embedding_**_；检索前对 query 做_ **_标点/空白归一化_**_；在 pgvector 上用_ **_余弦相似度阈值 0.28 + Top10_** _过滤弱相关笔记再注入 RAG，避免把无关笔记塞进上下文。」

若面试官追问「阈值怎么定的」，可答：当前是 **经验常量**（文档 0.28、函数默认 0.25），可结合日志 `[doc.enrichContext] matched=… top=0.xxx` 做离线调参；代码里 `similarity` 已回传到注入块，便于后续做可观测性。
```tandem-comments
// Schema: { "<id>": { anchor:{exact,prefix,suffix,pos?}, status:open|resolved, thread:[{author,ts,text}], suggestion?:{replacement,author,ts,result?} } }
// Anchor = quote from the prose. To locate: search for "exact", disambiguate via prefix/suffix.
{
  "0703": {
    "anchor": {
      "exact": "document",
      "pos": 582,
      "prefix": "空间，RAG 里常见做法是 **入库用 ",
      "suffix": "、检索用 query**。这是典型的「*"
    },
    "status": "open",
    "thread": [
      {
        "author": "apple",
        "ts": "2026-08-17T07:23:16.138Z",
        "text": "被检索的「资料」（你的笔记正文）"
      }
    ]
  },
  "d187": {
    "anchor": {
      "exact": "query",
      "pos": 595,
      "prefix": " **入库用 document、检索用 ",
      "suffix": "**。这是典型的「**非对称 Embed"
    },
    "status": "open",
    "thread": [
      {
        "author": "apple",
        "ts": "2026-08-17T07:23:26.945Z",
        "text": "用来搜资料的「问题」（用户在 AI 里问的那句话）"
      }
    ]
  }
}
```
