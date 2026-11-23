---
created: 2026-08-12T10:17
---


---
## **总原则（先记 4 条）**

| **手段**                           | **解决什么**                            |
| -------------------------------- | ----------------------------------- |
| **依赖写全** + **cleanup**           | effect / 订阅 / 定时器跟着最新 props 走，并清掉旧的 |
| **函数式** `setState(prev => …)`    | 异步里不读闭包中的旧 state                    |
| `useRef` **存「最新值」**              | 必须长期稳定的回调里读当前 id / session          |
| **取消 / 比对**（AbortController、版本号） | 异步回来时已是过期请求，直接丢弃                    |

---

## **1.** `useEffect` **里（**`[]`**、定时器、依赖漏写）**

**表现**：`setInterval` 里 `count` 不变；`userId` 换了还请求旧用户。

| **解法**         | **做法**                                                        |
| -------------- | ------------------------------------------------------------- |
| 写全依赖           | `[count]`、`[userId]`，变了就重新执行 effect                           |
| cleanup        | `return () => clearInterval(id)` / `clearTimeout` / `abort()` |
| 函数式 setState   | `setCount(c => c + 1)`，定时器里不读 `count`                         |
| 空依赖若必须稳定       | 用 `ref`：`countRef.current = count`，定时器读 `ref`                 |
| 能不用 effect 就不用 | 事件里直接处理，减少「常驻闭包」                                              |

---

## **2. 事件处理函数（onClick 等）**

**表现**：点击时用到的是几轮渲染前的 `title` / `noteId`。

| **解法**        | **做法**                                                                                        |
| ------------- | --------------------------------------------------------------------------------------------- |
| 不刻意缓存 handler | 普通 `const handle = () => …` 每次渲染是新闭包，通常没问题                                                    |
| memo 子组件      | 给子组件传 handler 时，父组件 `useCallback` **依赖写全**，或不要 memo 子组件                                       |
| 用 ref 包最新逻辑   | `onClick={() => handlerRef.current()}`，`handlerRef.current = () => save(note.id, title)` 每轮同步 |
| 从 store 读最新   | `onClick={() => useStore.getState().save(id)}`，少依赖组件闭包                                        |

---

## **3.** `useCallback` **/** `useMemo` **依赖漏写**

**表现**：`comments` 变了，`save` 里还是旧的；防抖一直 PATCH 旧笔记。

| **解法**           | **做法**                                                                                       |
| ---------------- | -------------------------------------------------------------------------------------------- |
| 补全依赖数组           | `[note.id, comments, updateNote]`                                                            |
| 少包 `useCallback` | 只有传给 `memo` 子组件或作为 effect 依赖时才包                                                              |
| 依赖项用 ref         | `const noteIdRef = useRef(note.id); noteIdRef.current = note.id`，callback 依赖 `[]` 但内部读 `ref` |
| 函数式更新 / 传参       | `updateNote` 时把 `id` 当参数传入，不闭包捕获 `note` 整对象                                                  |

---

## **4. 防抖 / 节流 /** `setTimeout`**（含** `setTimeout(onSave, 0)`**）**

**表现**：切换笔记后，延迟回调仍保存到上一篇。

| **解法**             | **做法**                                                                                     |
| ------------------ | ------------------------------------------------------------------------------------------ |
| cleanup 取消定时器      | 组件卸载或 `note.id` 变化时 `clearTimeout`                                                         |
| 闭包里带 id 并校验        | `const id = note.id; setTimeout(() => { if (id !== currentNoteIdRef.current) return; … })` |
| 防抖 key 按实体分        | 像 `notesStore` 的 `Map<noteId, timer>`，不同笔记互不影响                                             |
| 用 ref 存最新 `onSave` | `onSaveRef.current = onSave` 每轮更新，定时器里调 `onSaveRef.current()`                              |

---

## **5. 异步请求 / SSE / fetch 竞态**

**表现**：慢请求后返回，覆盖了新会话的消息；串 session。

| **解法**                 | **做法**                                                                 |
| ---------------------- | ---------------------------------------------------------------------- |
| **AbortController**    | `userId` / `sessionId` 变了 `abort()` 旧请求                                |
| **请求版本号 / id 比对**      | `const reqId = ++ref.current;` 回调里 `if (reqId !== ref.current) return` |
| effect cleanup 里 abort | 与依赖 `[sessionId]` 绑定                                                   |
| 按 session 分桶状态         | 流式内容存 `messages[sessionId]`，不用全局单一 `streaming`（架构层）                    |
| 函数式更新列表                | `setMessages(prev => …)` 基于最新列表合并                                      |

---

## **6. 自定义 Hook 返回「稳定函数」**

**表现**：`const { doWork } = useFoo()` 里 `doWork` 永远看到初始 `x`。

| **解法**                        | **做法**                                    |
| ----------------------------- | ----------------------------------------- |
| 依赖写全                          | `useCallback` 带上 `[x]`                    |
| 返回 ref + 稳定外壳                 | `doWork` 固定引用，内部 `ref.current()`          |
| 文档说明                          | 「调用前需最新请传参」：`doWork(x)` 不依赖闭包             |
| 用 `useEffectEvent`（React 19+） | 实验/正式 API：在 effect 里读最新 props 且不反复重跑（视版本） |

---

## **7.** `useRef` **和** `useState` **混用不当**

**表现**：`ref` 没每轮同步；事件只绑一次，永远读 mount 时的值。

| **解法**    | **做法**                                      |
| --------- | ------------------------------------------- |
| 每轮渲染同步    | `latestIdRef.current = sessionId`（写在组件函数体里） |
| 事件用 ref 读 | 监听器里 `ref.current`，不要捕获 `state`             |
| 绑一次的事件要解绑 | `useEffect` + cleanup `removeEventListener` |

---

## **8. 列表 +** `memo` **子项**

**表现**：删错行、点到旧 `item`。

| **解法**         | **做法**                                                               |
| -------------- | -------------------------------------------------------------------- |
| **key 用稳定 id** | `key={item.id}`，不用 index                                             |
| 行内回调可接受每次新建    | `onDelete={() => delete(item.id)}` + 正确 key                          |
| `memo(Row)`    | 传 `onDelete` 时父级 `useCallback` 依赖含 `item.id` 或传 `id` 让 Row 内部调 store |

---

## **9. 全局 Store（Zustand / Redux）**

**表现**：相对少；若解构的 action 引用旧仍可能怪。

| **解法**           | **做法**                                            |
| ---------------- | ------------------------------------------------- |
| 异步里 `getState()` | `useNotesStore.getState().updateNote(id, patch)`  |
| 不解构或 selector 稳定 | `useNotesStore(s => s.updateNote)` 若 store 保证引用稳定 |
| 参数显式传入           | `updateNote(noteId, patch)`，id 来自调用时参数而非闭包        |

---

## **10. 第三方库（TipTap、D3、**`document` **监听）**

**表现**：`onUpdate` 里 `comments` 过期；D3 点击还路由到旧笔记。

| **解法**                 | **做法**                                        |
| ---------------------- | --------------------------------------------- |
| ref 包 handler          | `onUpdate: () => onUpdateRef.current(editor)` |
| 依赖变则重建                 | `useEditor` 配置随 `note.id` 重建（成本高时改用 ref）      |
| effect 里绑事件 + cleanup  | D3：`return () => simulation.stop()`、移除监听      |
| 依赖 `[data, isDark]` 重绘 | 你 `NoteGraph` 的做法：主题变就重建图谱                    |

---

## **11. 类组件（了解）**

| **解法**              |
| ------------------- |
| 箭头函数 / `bind(this)` |
| 或改为 Hooks           |

---

## **场景 × 解法速查表**

| **场景**              | **首选解法**                    | **常配****合**   |
| ------------------- | --------------------------- | ------------- |
| effect + 定时器        | cleanup + 函数式 setState      | ref           |
| effect + `[]` 却要最新值 | ref 同步                      | 或改依赖          |
| 依赖漏写                | 补依赖 / eslint                | —             |
| onClick 读旧 state    | 每轮新函数 / ref / getState      | —             |
| useCallback 过旧      | 补依赖 / 少缓存                   | ref           |
| 防抖 / setTimeout     | 按 id 分桶 + clear + 校验 id     | ref           |
| fetch / SSE 竞态      | Abort + 版本号比对               | 按 session 分状态 |
| Hook 返回稳定 API       | ref 外壳 / 依赖写全               | 传参            |
| memo 列表子组件          | 正确 key + callback 依赖        | —             |
| TipTap / D3         | handlerRef + effect cleanup | 依赖重建          |

---

## **和你之前总结的对应**

你说的两点依然成立，可以扩展成完整版：

1. **写全** `useEffect` **/** `useCallback` **依赖 + cleanup** → 解决 effect、订阅、定时器、第三方绑定。
    
2. **函数式** `setState` → 解决异步里「计数 / 列表」基于最新状态更新。
    
3. **再加**：**ref 最新值**、**abort/版本号**、**store.getState()**、**按实体 id 防抖** → 覆盖事件、请求、切页、Store 等「不在 effect 里」的陷阱。