# Stack Forge 迭代升级路线图

仓库: https://github.com/smartchaos/stack-forge

## 执行摘要

这份文档的目标，不是单纯评价 Stack Forge，而是给出一条可以持续迭代的升级路线，让它从一个“能生成 workflow 模板的 CLI”，演进成一个“可验证、可恢复、可扩展的 workflow runtime”。

核心判断只有一句:

Stack Forge 现在最值得肯定的地方，是它选对了问题; 它现在最需要补的地方，是 runtime。

换句话说:

- 方向是对的。
- 工程感是有的。
- 最大短板不在界面，不在文档，不在命令数量。
- 最大短板在于: 编排语义主要存在于 prompt 和模板里，而不是程序里。

## 适用场景

这份文档可以直接拿来做三件事:

1. 作为仓库后续版本规划草案。
2. 作为 issue / PR 的拆分依据。
3. 作为对外讨论“这个项目下一步最该补什么”的说明材料。

## 当前项目的优点

### 1. 选题准确

它没有重复造一个新的 coding agent，而是试图做一个统一编排层，把已有 provider 串成完整开发流水线。这个切入点是成立的，因为团队实际遇到的问题通常不是“没有 agent”，而是“agent 能力碎片化、流程不连续、结果不可追踪”。

### 2. 工程结构已经有雏形

从现有代码结构看，项目已经具备继续演化的基本骨架:

- `src/discovery/`
- `src/generator/`
- `src/state/`
- `src/validation/`
- `templates/`
- `registry/`
- `tests/`

这意味着它不是一个只能推倒重来的实验品，而是一个可以在现有地基上迭代加固的项目。

### 3. 产品体验方向正确

`cforge init` 负责扫描 provider、生成配置、准备 workflow 运行所需文件，这个“零配置接管工作流”的方向本身是有吸引力的。对于终端用户来说，这比要求他们自己维护大量 prompt、skill 和手工步骤更有产品价值。

### 4. 已经抓到了真正值钱的阶段

项目没有把重心放在花哨功能上，而是围绕:

- brainstorm
- specification
- planning
- implementation
- review
- release

这条主链路本身就是有现实意义的。

## 最大缺陷

### 结论

Stack Forge 当前最大的缺陷是:

编排主要存在于模板和提示词里，而不是存在于一个受控、可验证、可恢复的 runtime 里。

### 这为什么是根问题

如果这一点不解决，后续很多功能都会变成表面增强:

- 多几个命令，不会显著提升可靠性。
- 多几个 provider，不会显著提升确定性。
- README 写得更完整，也不能解决执行闭环问题。

### 它会直接导致什么问题

1. 阶段边界不硬  
stage 是否完成，很多时候依赖模型“觉得自己做完了”。

2. 状态推进不硬  
`state.json` 更像记录文件，不像真正受控的状态机。

3. provider 检查不硬  
当前 healthcheck 更像“装了没、扫到了没”，不是“这个 provider 是否真的能完成最小任务”。

4. 并行执行不硬  
README 里讲的是并行实现，但现在更像“模板要求模型去并行”，而不是“程序真实调度并验证并行”。

5. 失败恢复不硬  
一旦 stage 半途出错，程序缺乏足够的执行快照、回滚点和恢复路径。

## 产品目标

把 Stack Forge 升级为一个具备以下特征的系统:

- workflow 生命周期由程序主导，而不是由 prompt 自觉维持
- 每个 stage 都有明确 contract
- 每次状态推进都有验证
- 每个 provider 的可用性都能用最小真实任务验证
- 并行执行有任务图、调度器和冲突检查
- 失败后可以 resume / retry，而不是只能重跑或人工修补

## 升级原则

1. 先补 runtime，再补生态。
2. 先补可验证性，再补智能化。
3. 先补失败恢复，再补性能优化。
4. 所有阶段都必须有输入、输出、通过条件、失败条件。
5. 模板保留，但模板应当服务 runtime，而不是替代 runtime。

## 路线图总览

建议版本节奏:

- V0.3: Runtime 收口
- V0.4: Healthcheck 做实
- V0.5: Parallel Implementation 做实
- V0.6: 失败恢复与可观测性
- V0.7: Spec 对齐验证
- V1.0: 可扩展编排平台

建议优先级:

1. V0.3
2. V0.4
3. V0.5
4. V0.6
5. V0.7
6. V1.0

## V0.3: Runtime 收口版

### 目标

让 Stack Forge 不再主要依赖“生成 skill 并让模型照做”，而是由程序明确控制 workflow 生命周期。

### 关键改动

- 引入 stage contract
- 引入显式状态机
- 引入 artifact manifest
- 把 stage 完成条件放进代码
- 给执行过程增加统一入口

### 建议新增结构

```text
src/runtime/
  contracts.ts
  transitions.ts
  artifacts.ts
  execution.ts
  errors.ts
```

### 关键设计

每个 stage 至少定义以下字段:

- `name`
- `inputs`
- `outputs`
- `preconditions`
- `completion_checks`
- `failure_modes`
- `retry_policy`

示例:

```ts
type StageContract = {
  name: "planning";
  inputs: ["proposal.md", "specs/**"];
  outputs: ["tasks.md"];
  completionChecks: ["file_exists:tasks.md", "file_nonempty:tasks.md"];
};
```

### 需要落地的 CLI 行为

- `cforge run` 或默认 `cforge` 要进入受控执行路径
- 每次 stage 进入前验证输入
- 每次 stage 完成后验证输出
- 未通过验证不能自动推进到下一阶段

### 验收标准

- stage transition 只能按合法顺序发生
- 缺少必要 artifact 时 workflow 阻塞且给出明确原因
- `state.json` 成为单一事实来源
- state 中可看出每个 stage 的开始、完成、失败和重试信息

### 对应 issue 建议

- `feat(runtime): add stage contract model`
- `feat(state): enforce legal workflow transitions`
- `feat(runtime): validate stage outputs before advancing`
- `feat(runtime): add artifact manifest support`

## V0.4: Provider 健康检查升级版

### 目标

把 healthcheck 从“是否安装”升级为“是否可执行最小真实任务”。

### 当前问题

当前检查大多停留在:

- plugin installed
- skill exists
- mcp configured

这些只能说明“可能能用”，不能说明“真的能完成这个 capability 的最低要求”。

### 关键改动

- 为每类 capability 设计 smoke task
- 记录 capability 级别的验证结果
- 区分可发现和可执行
- 明确失败原因

### 建议健康状态模型

```ts
type ProviderStatus =
  | "missing"
  | "installed"
  | "detected"
  | "executable"
  | "stale"
  | "failed";
```

### smoke task 示例

- brainstorm: 给一句需求，是否能产出结构化 proposal
- specification: 给 proposal，是否能产出带章节的 spec
- planning: 给 spec，是否能产出结构化 tasks
- review: 给一个小 diff，是否能产出 review 结果
- release: 是否能识别 branch / PR / merge 前置条件

### 建议扩展 health record

- `provider`
- `capability`
- `status`
- `last_verified`
- `verification_mode`
- `sample_task`
- `duration_ms`
- `failure_reason`

### 验收标准

- `cforge healthcheck` 能区分“装上了”和“能干活”
- health 输出可定位到具体 capability
- stale provider 会被明确标记

### 对应 issue 建议

- `feat(health): add capability smoke tests`
- `feat(health): distinguish detected vs executable`
- `feat(health): persist provider verification metadata`

## V0.5: 并行执行做实版

### 目标

把 README 中的 parallel implementation 从“描述性能力”变成“程序级能力”。

### 当前问题

- 任务依赖关系主要存在于文本理解里
- 没有正式的依赖图
- 没有调度器
- 没有统一的冲突检测和结果汇总

### 关键改动

- 把 `tasks.md` 解析成结构化任务
- 建立 dependency graph
- 建立 batch scheduler
- 做并行前后的一致性校验

### 建议新增结构

```text
src/implementation/
  parser.ts
  dependency-graph.ts
  scheduler.ts
  conflict-checker.ts
  verifier.ts
```

### 任务模型建议

```ts
type ImplementationTask = {
  id: string;
  summary: string;
  files: string[];
  explicitDependencies: string[];
  inferredDependencies: string[];
  status: "pending" | "running" | "done" | "failed";
};
```

### 调度规则建议

- 同一批次内不允许文件写冲突
- 同一批次最多 2-3 个任务
- 显式依赖优先于文件冲突推断
- 无法确认独立性的任务默认串行

### 验收标准

- 程序能从 `tasks.md` 稳定生成 batch plan
- 并行批次生成结果可复现
- 批次执行后能聚合 test / typecheck / conflict check 结果

### 对应 issue 建议

- `feat(impl): parse tasks.md into structured task graph`
- `feat(impl): add dependency-aware batch scheduler`
- `feat(impl): detect file overlap across parallel tasks`
- `feat(impl): aggregate validation after batch execution`

## V0.6: 失败恢复与可观测性版

### 目标

让 workflow 失败之后可以恢复，而不是只能依赖人工清理和重新运行。

### 关键改动

- 保存每个 stage 的 execution snapshot
- 提供 resume / retry / skip-with-reason
- 建立事件日志
- 为失败定位保留执行上下文

### 建议新增文件

```text
.cforge/events.jsonl
.cforge/executions/<stage>/<timestamp>.json
.cforge/checkpoints/<stage>.json
```

### 建议记录的事件

- `workflow_started`
- `stage_started`
- `stage_blocked`
- `artifact_written`
- `validation_failed`
- `provider_unhealthy`
- `retry_scheduled`
- `workflow_resumed`
- `workflow_completed`

### 验收标准

- 中断后可 resume
- 失败后可 retry 单个 stage
- 可以快速看出“失败发生在哪一步、因为什么、用的哪个 provider”

### 对应 issue 建议

- `feat(runtime): persist execution snapshots`
- `feat(runtime): support workflow resume`
- `feat(runtime): add structured event log`

## V0.7: Spec 对齐验证版

### 目标

让 `cforge validate` 验证“实现是否覆盖需求”，而不是只检查一些静态字符串模式。

### 当前问题

现有验证器更像轻量规则检查器，对“行为是否满足 spec”覆盖不足。

### 关键改动

- 把 spec 转成结构化 requirement
- 为 requirement 绑定 evidence
- 输出 requirement-by-requirement 验证报告
- 允许验证结果影响 workflow 状态

### requirement 模型建议

```ts
type Requirement = {
  id: string;
  description: string;
  acceptanceCriteria: string[];
  evidenceType:
    | "file_exists"
    | "content_pattern"
    | "export_exists"
    | "test_exists"
    | "test_passes"
    | "command_output";
  critical: boolean;
};
```

### 验收标准

- `cforge validate` 给出逐条 requirement 的 evidence
- 可以明确区分:
  - 已实现
  - 无证据
  - 已失败
  - 需要人工确认

### 对应 issue 建议

- `feat(validate): add structured requirement evidence model`
- `feat(validate): generate requirement-by-requirement reports`
- `feat(validate): feed validation result back into workflow state`

## V1.0: 可扩展编排平台版

### 目标

让 Stack Forge 成为 provider-neutral 的 workflow runtime，而不是只服务当前少量固定 provider 的封装器。

### 关键改动

- 稳定 capability interface
- workflow 定义外部化
- provider ranking / fallback
- repo preset / org preset / language preset

### 建议扩展方向

- feature workflow
- bugfix workflow
- refactor workflow
- incident workflow
- migration workflow

### 验收标准

- 新增 workflow 不需要大改核心代码
- 新增 provider 主要通过注册完成
- fallback provider 可在主 provider 失败后接管

### 对应 issue 建议

- `feat(platform): externalize workflow definitions`
- `feat(platform): add provider ranking and fallback`
- `feat(platform): support repository-level presets`

## 推荐里程碑

### Milestone 1: Runtime Foundation

范围:

- V0.3 全量

完成定义:

- workflow 生命周期开始受程序控制
- stage contract 生效
- state transition 可验证

### Milestone 2: Reliable Providers

范围:

- V0.4 全量

完成定义:

- provider 检查从存在性验证升级到执行性验证

### Milestone 3: Real Parallelism

范围:

- V0.5 全量

完成定义:

- parallel implementation 有真实调度、冲突检测和验证机制

### Milestone 4: Recovery and Evidence

范围:

- V0.6 + V0.7

完成定义:

- 失败可恢复
- spec 覆盖可验证

### Milestone 5: Platformization

范围:

- V1.0

完成定义:

- workflow 和 provider 都具备扩展性

## 推荐 PR 拆分

为了避免一次性改太大，建议按下面方式拆 PR:

1. `state machine refactor`
2. `stage contract + artifact manifest`
3. `stage completion validation`
4. `provider smoke healthcheck`
5. `health record schema upgrade`
6. `tasks parser + dependency graph`
7. `batch scheduler + conflict checker`
8. `execution snapshot + event log`
9. `resume / retry support`
10. `structured requirement validation`

## 风险与注意事项

### 1. 不要把抽象做过头

这个项目现在最缺的是“把关键路径做硬”，不是“先发明一套很美的通用框架”。如果 abstraction 先行，很容易把速度拖慢。

### 2. 不要让 healthcheck 变成完整端到端测试

smoke task 应该小、快、稳定，只验证 capability 最小闭环，不要把它做成昂贵的大任务。

### 3. 并行功能默认要保守

宁可把不确定任务降级为串行，也不要因为错误并行导致工作区冲突和结果污染。

### 4. validate 需要证据模型，不要只加更多字符串规则

如果只是继续堆 pattern check，复杂度会上升，但价值不会同步上升。

## 最小可行成功定义

如果只能先做一件最值钱的事，那就是完成 V0.3。

原因很简单:

- 没有 runtime，healthcheck 价值有限
- 没有 runtime，并行能力不可靠
- 没有 runtime，失败恢复无法落地
- 没有 runtime，validate 也无法稳定嵌入工作流

所以这条路线图的真正起点不是“多做几个功能”，而是:

先把 workflow 的控制权从模板层收回到程序层。

## 结论

Stack Forge 的潜力是真实存在的，因为它在解决一个真实问题: 如何把已有 agent/provider 能力编排成可复用的开发流程。

它当前最大的限制也同样明确: 现在的“编排”更像一组结构良好的提示词和模板，而不是一个真正受控的执行系统。

后续所有高价值升级，都应围绕同一个方向:

把“模型自觉执行 workflow”升级为“系统受控执行 workflow”。

如果这个方向做成了，Stack Forge 才会从“不错的工作流脚手架”进入“值得长期依赖的编排平台”。
