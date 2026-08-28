
---

> [!NOTE] 组件作用
> **pytest 测试模块**，专门测 `src/workflow_validator.py`（校验器），负责## Schema 层和图结构校验，涉及一个正向测试和六个反向测试

> [!NOTE] 概括汇总
> 走到对应的测试函数后，构造对应的反向测试的结构，然后输入给校验函数，分别读出报错，将报错存入exc_info，并通过assert进行原因针对排查。若符合则通过，用例 PASSED，然后继续走下一个反向测试，一直走完将结果汇总

## 反向测试逻辑

```Python
with pytest.raises(WorkflowValidationError) as exc_info:
    validate_workflow({...故意造的坏数据...})

assert "某个关键词" in exc_info.value.message   # 或检查 errors 列表

```

- `pytest.raises()` 作用是等抛异常的监视器，本质是个 `with` 上下文管理器。提前预设好这段代码会抛出什么错误，若没抛出则测试失败，抛出后通过 `as exc_info` 将异常对象存到`exc_info` 里，方便后面检查。
	
- `assert` 是质量关键，具体报错的原因，防止其他原因干扰。比如 `assert "Schema" in exc_info.value.message` 检查报错是不是由 Schema 引起的

## 七个测试逻辑

### 正向测试

#### **`test_builtin_video_production_passes_schema

加载真实的 `video-production` 模板，断言返回值是 dict、id 对得上、节点数 ≥ 1。这是"全链路能过"的基线。

### 反向测试

#### **1. ****`test_validate_workflow_rejects_missing_required_fields`**

只传 `{"id": "incomplete"}`，缺 `name/version/nodes/edges`。断言：抛异常，且 message 含 `"Schema"`、`errors` 非空。

	打中 `validate_workflow` 里 `Draft202012Validator.iter_errors` 那段（Schema 层报错）。


#### **2. `test_validate_workflow_rejects_invalid_node_type`**

先 `json.loads` 读出**真实文件**，再把 `nodes[0]["type"]` 改成 `"unknown_type"`，然后校验。断言 errors 里有一条提到 `"unknown_type"` 或 `"enum"`。

	打中 Schema 的 `enum` 枚举校验（节点类型不在 5 种之内）


#### **3. `test_load_unknown_workflow_template`**

传一个不存在的 id `"does-not-exist"`。断言 message 含 `"未找到工作流模板"`。

	打中 load_and_validate_workflow 的"文件不存在"分支。


#### **4. `test_invalid_graph_missing_end_node`**

手搓一个工作流：只有 1 个 agent 节点 + 一条 `start→n1` 边，**没有 end 节点**。断言 errors 含 `"缺少结束节点"`。

	打中 `validate_workflow_graph` 里的 `if not has_end`

  
#### 5. `test_invalid_graph_missing_start_edge`

工作流有 agent 节点和 end 节点，但唯一一条边是 `n1→end`，**没有任何边从 start 出发**。断言含 `"缺少开始节点连接"`。

	打中 validate_workflow_graph 里的 if not any(edge["source"] == "start")


#### **6. `test_invalid_graph_isolated_node`**

三个节点：n1、n2（孤立）、end；边只连 `start→n1` 和 `n1→end`，**n2 谁都没连**。注意它调用的是 `validate_workflow_graph(workflow)`（直接打第二关），断言含 `"孤立节点"`。

	 打中 validate_workflow_graph 里那段 BFS：从 start 出发能走到 n1、end，走不到 n2，于是 node_ids - reachable = {n2}，被报成孤立节点。


## 完整链路

#### 输入脚本

```shell
cd services/orchestrator
pytest tests/test_workflow_validator.py::test_invalid_graph_isolated_node -v
```

```
pytest 启动并收集用例
   → 命中上述测试函数，准备执行
   → 进入测试函数体：构造 workflow（nodes=[n1,n2,end]，edges=[start→n1, n1→end]，n2 谁都没连）
   → 调用 validate_workflow_graph(workflow)        【落点：workflow_validator.py:92】
   → 提取 node_ids={n1,n2,end}
   → 检查 has_end：有 end → 不报错（过）
   → 检查 start 边：e1 的 source 是 "start" → 不报错（过）
   → 建邻接表 adjacency = {start:[n1], n1:[end], n2:[], end:[]}
   → BFS 从 "start" 扩散：走到 n1、再走到 end → reachable={n1, end}
   → 计算差集 node_ids - reachable = {n2}（非空，说明 n2 走不到）
   → 收集错误：errors = ["孤立节点：n2 未与开始节点连通"]
   → if errors → raise WorkflowValidationError("工作流图结构无效", errors)   【结果：抛出异常】
   → pytest.raises(WorkflowValidationError) 接住：类型匹配 → 吞掉异常，存入 exc_info，with 块正常结束
   → assert any("孤立节点" in err ...) → 在 errors 里命中 → 成立（过）
   → 测试函数无异常返回
   → 该用例标记 PASSED ✅
   → pytest 汇总打印 "1 passed"，进程退出码 0   【完成测试】
```





