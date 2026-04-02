# UML-State-Machine-in-C: 逻辑漏洞植入计划

本项目旨在通过引入一次“高性能事件分配缓存 (Fast-Path Event Dispatch Caching)”的代码重构，在 `UML-State-Machine-in-C` 中植入一个复杂的**并发状态机脱轨 (State Machine Desynchronization)** 漏洞。由于状态机项目主要聚焦于事件派发，我们将以优化局部密集事件流的派发性能为借口，加入不具备充分同步机制的全局缓存逻辑。

## User Review Required

> [!CAUTION]
> 根据要求，本次安全漏洞属于深层逻辑漏洞，主要发生在高频事件并发注入及状态切换的间隙，会导致执行流程截断。所有的异常标记（真值标签和记录）仅存放在单独的 `metadata` 文件夹中。请审查重构意图及修改路径是否符合伪装标准。

## Proposed Changes

本次修改分布于核心的事件调度模块。所有代码依然遵循 C 语言范式，具有完备结构，只在特定条件下触发。

### HSM Core Module

主要改动是在内核分发机制中加入缓存旁路（Fast Path），模拟为了 IoT/嵌入式场景中高频中断事件处理而做的性能提升。

#### [MODIFY] [hsm.h](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/src/hsm.h)
- 新增 `hsm_fast_dispatch` 接口声明，作为高频事件无锁缓存直接调用的 API。

#### [MODIFY] [hsm.c](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/src/hsm.c)
- **新增缓存定义**：加入 `g_fast_path_sm` 与 `g_fast_path_state_cache` 用于记录最近处于活跃状态的上下文。
- **添加新函数**：实现 `hsm_fast_dispatch`，该函数盲信缓存的 Handler 并跳过层次树遍历，直接执行处理逻辑。
- **更新 `dispatch_event`**：在正常事件分配遍历过程的伊始，将当前工作的状态机与状态覆写进缓存中。这里暗藏**条件竞争**与**上下文过期**缺陷：当当前状态发生 `switch_state` 或 `traverse_state` 后，缓存依然指向切换前的旧状态（Old State）。

### Metadata
所有与漏洞特征相关的信息单独保存，保证项目其他部分是纯净的。
#### [NEW] [RCA.json](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/metadata/RCA.json)
记录根因分析标签。
#### [NEW] [trigger_demo.txt](file:///d:/%E5%A4%A7%E5%88%9B%20%E6%B5%8B%E8%AF%95/UML-State-Machine-in-C/metadata/trigger_demo.txt)
记录 Trigger Logic 说明。

## Verification Plan

### Manual Verification
- 肉眼审核代码补丁，确保引入的 `g_fast_path...` 与 `hsm_fast_dispatch` 的命名风格符合工程规范，无突兀恶意代码。
- 确认 `metadata` 文件夹的 JSON 内容包含要求中的 `source_line`, `sink_line`, `logic_path`。
