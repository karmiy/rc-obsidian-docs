1. Discovery：从 specs 发现尚未完成的任务列表，创建临时文件 STATES.md 记录任务列表与状态，进入 Plan 流程
2. Plan：计划任务如何进行（优先级、串行 or 并行），可以同时 plan 一个或多个任务，根据任务的依赖性、独立性按如下规则收集 plan tasks，**并行**进入 Execute 流程，同时更新 STATES.md 的任务排序与状态
	1. 如果任务 A 的实现是其他任务的前提，则提到最高优先级，本轮视为 plan task
	2. 如果任务 B 的实现相对独立，没有外部依赖，本轮视为 plan task
	3. 如果任务 C 的实现依赖别的任务，有后置依赖，本轮不安排
3. Execute：进入当前流程的 plan task 应该起**独立的 sub agent** 进入开发，并遵循如下规则，完成后 sub agent 将产物交付给 Verify 流程 ：
	1. memory ：
		1. 始终以 specs 的设计为根源需求文档上下文记忆
		2. 开发过程中可以在 STATES.md 实时更新实现的细节记忆便于跟踪
	2. work rule：
		1. 开发前先确认理解 specs 下对应的内容要做什么
		2. 开发完成后再次 review 对比 specs 和实现的代码是否一致，如果不一致需要重新根据 spec 方案对齐代码实现，对齐结束后继续回到当前步骤 review， 直到完全一致
	3. coding：代码**必须**确保先思考如何设计、如何封装再开始实现
		1. 先确认当前实现应该放在 FE、BE，再确认应该放在什么目录层，最后确认应该在原文件里扩展还是隔离独立职责构建新文件
		2. 不论是否多出复用，独立的职责都应该抽离独立的组件/class/function/hook，拒绝代码堆叠，把可拆分的逻辑耦合在一份文件内导致一个文件数百上千行
	4. block：只根据 spec 的方案实现，开发过程中发现 block 问题应该停止 loop，输出 block point 重新讨论，严格拒绝为了解决问题而做不符合 specs 方案的 fallback 处理
	5. production business：开发的同时优先保证不影响原业务需求
4. Verify：Execute 流程中完成的任务需要单独安排一位验证员 sub agent 进行 1 对 1 验证，下述任何不满足的都需要拒绝并打回对应 Execute sub agent 进行修复，通过过进入下一次 loop 回到 Plan 流程
	1. 代码实现与 specs 设计不一致
	2. 代码实现没有做到 Execute 的 coding、production business 规则
	3. UT/IT/E2E/contract test 不通过
	4. 若需要，可通过 @computer plugin 在浏览器模拟真人测试