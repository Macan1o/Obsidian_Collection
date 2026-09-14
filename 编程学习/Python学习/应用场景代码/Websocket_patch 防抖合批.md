
---

> [!NOTE] 项目路径
> Python_Study/websocket_patch防抖合批

## 使用验证方式

	npx tsx demo.ts

|文件|作用|
|---|---|
|`types.ts`|`StatePatch` / `RunState` 契约|
|`mergePatches.ts`|合批规则：标量覆盖、messages 按 id、terminal append/replace、hybrid 浅合并|
|`PatchStore.ts`|核心：校验 `run_id` → pending → 重置 timer → flush → apply → emit|
|`usePatchStore.ts`|`useSyncExternalStore` 订阅|
|`RunView.example.tsx`|React + WebSocket 接线示意|
|`demo.ts`|无 React 可跑演示|

## 链路逻辑

> [!NOTE] 核心需求场景
> 在 WebSocket 高频推送 `state_patch` 时，先积攒部分，再合并统一更新UI，保证聊天流式输出、终端日志、执行状态都流畅，又不卡死主线程。

### 调用链关系图

![[Pasted image 20260914155207.png|836]]

#### 初始化阶段

- 先用当前 run_id `useMemo` 出一个 `PatchStore` 初始化状态，消息/日志
	
- `usePatchStore(store)` 调用 `useSyncExternalStore` 保证响应能力
	
- 通过 useEffect 给 `ws` 事件挂上 `message` 监听

#### 事件到达阶段

- 服务端推送事件时触发 `ws` 的 `message` 
	
- `RunView` 预处理检查类型并且进行去重
	
- `pendingPatches.push(patch)` 数据放入队列

#### 防抖阶段

此时在规定时间内如果又来了新的事件，再 `push` 进 `pendingPatches` 清掉上轮 timer 重新计时

#### flush 阶段

- `snapshot = pendingPatches` 后立刻 `pendingPatches = []`、`timer = null` 完成这一批 patch
	
- `merged = mergePatches(snapshot)` 完成字段 merge ，其中：标量后者覆盖、messages 按 id 合、终端按 append/replace、hybrid 按 id 浅合。
	
- `this.state = applyMergedToState(旧 state, merged)`  → 这里才第一次改完整状态。
	
- `emit()`：遍历 `listeners`，把「数据变了」通知出去。

#### 重渲染阶段

- emit 后 React 调 `getSnapshot()` → 拿到已经合批后的 `RunState`
	
- 触发 `RunView` 重渲染。
	
- 完成页面渲染
```tandem-comments
// Schema: { "<id>": { anchor:{exact,prefix,suffix,pos?}, status:open|resolved, thread:[{author,ts,text}], suggestion?:{replacement,author,ts,result?} } }
// Anchor = quote from the prose. To locate: search for "exact", disambiguate via prefix/suffix.
{
  "b926": {
    "anchor": {
      "exact": "字段 merge",
      "pos": 1078,
      "prefix": "atches(snapshot)` 完成",
      "suffix": " ，其中：标量后者覆盖、messages"
    },
    "status": "open",
    "thread": [
      {
        "author": "macando",
        "ts": "2026-09-14T08:15:02.548Z",
        "text": "四种字段合并规则"
      },
      {
        "author": "macando",
        "ts": "2026-09-14T08:15:34.351Z",
        "text": "1. 标量后者覆盖\n\n单个值字段，如 status、phase、error（不是数组、不是对象列表）\n\n同一安静窗口里来了多条 patch，只认最后一次写到的值。"
      },
      {
        "author": "macando",
        "ts": "2026-09-14T08:16:15.974Z",
        "text": "2. messages 按 id 合\n\n每条消息有 id；同一个 id 就合并成一条，字段用后来的盖前面的；新 id 就追加。"
      },
      {
        "author": "macando",
        "ts": "2026-09-14T08:17:27.422Z",
        "text": "3. 终端按 append / replace\n\n终端日志带 mode，根据 mode append / replace\n\nappend = 日记本往后写\nreplace = 撕掉重抄一整页"
      },
      {
        "author": "macando",
        "ts": "2026-09-14T08:18:26.644Z",
        "text": "4. hybrid 按 id 浅合\n\nhybrid_executions 是一堆「执行任务」，每个有 id。\n\n任务卡片按编号更新；只改你这次带来的字段，没写的字段原样留着。"
      }
    ]
  }
}
```
