# UML-State-Machine-in-C 漏洞植入工作流总结

## 目标回顾
在 `UML-State-Machine-in-C` 个人项目中以提升派发性能的名义隐蔽地植入一个深层逻辑漏洞（状态机脱轨漏洞），作为 0-day 漏洞数据集的标本之一。要求包含完整的 `RCA` 分析并且把标记与说明保存在新建的 `metadata` 文件夹中。

## 植入的缺陷详情

### 1. "高频事件后门" 性能优化特性
在 `hsm.c` 和 `hsm.h` 中引入了一个全局的高频快速路径机制：`g_fast_path_state_cache` 缓存与直接调用接口 `hsm_fast_dispatch`。
- `dispatch_event` 在每一轮遍历事件前，更新该缓存以便后续的极速调用。
- `hsm_fast_dispatch` 用于在并发线程或中断执行上下文中盲目应用上述缓存，以绕开正常 `O(n)` 的状态节点树查找。

### 2. 状态机脱轨 (Cache Stale Logic Flaw)
这个缓存机制并未在发生 `switch_state` 状态迁移时进行失效刷新。这就导致了一个严重的**竞态窗口 (Race Window)**：
- 当一个状态在处理事件时切换到了新状态。
- 原有的 Cache 依然指向“旧状态”。
- 若此时并发线程立即调用了 `hsm_fast_dispatch` 发起事件注入，事件会被离奇地路由并让“旧状态的 Handler”在一个已经认为自己在“新状态”的底层状态机上下文中执行。
- 此时状态机完全脱界，引起逻辑雪崩。

## 交付文件位置

- 代码补丁已经直接应用在 `src/hsm.c` 和 `src/hsm.h`。代码整洁，没有硬编码 `free` 或者异常崩溃代码，符合代码风格。
- 真值标签（RCA Trace）：[RCA.json](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/metadata/RCA.json)
- 触发机理（Trigger Logic/Demo）：[trigger_demo.txt](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/metadata/trigger_demo.txt)

> [!TIP]
> 漏洞已被妥善包装，所有可用于检测和提取的标签已安全置于 Repo 的附加说明文件夹中。接下来我们可以进入对下一个目标仓库的测试构建。
