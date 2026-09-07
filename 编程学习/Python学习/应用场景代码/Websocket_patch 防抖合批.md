
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
