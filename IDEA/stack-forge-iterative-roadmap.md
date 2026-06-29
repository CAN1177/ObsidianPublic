# Stack Forge 迭代升级路线图

仓库: https://github.com/smartchaos/stack-forge

## 目标

把 Stack Forge 从“workflow scaffolding + prompt compiler”升级为“可验证、可恢复、可扩展的 workflow runtime”。

当前判断:

- 产品方向是对的: 用统一编排层连接已有 Claude Code provider。
- 当前最大短板不是 UI，不是命令数量，而是 runtime 不够硬。
- 目前很多关键行为主要靠生成的 Markdown skill 和模型自行遵守，程序本身缺少强约束和强验证。

## 当前优势

- CLI 入口和目录结构清晰，后续扩展成本不高。
- discovery / generator / state / validation 已经分层。
- 已有测试、文档、模板系统，适合持续演进。
- 已经抓住了真正有价值的编排对象: brainstorm / spec / plan / build / review / release。

## 核心问题

### 最大缺陷

编排主要存在于 prompt 模板里，而不是 runtime 里。

这会带来四个直接后果:

1. 阶段边界不硬，stage 是否真正完成缺少机器校验。
2. 状态推进不硬，`state.json` 更多像记录，不像受控状态机。
3. provider 健康检查不硬，现在更像“装了没有”，不是“能不能真的干活”。
4. 并行实现不硬，README 里强调 parallel implementation，但实际执行更多依赖模型遵守说明。

## 总体升级原则

1. 先补 runtime，再补 provider 数量。
2. 先补可验证性，再补自动化炫技。
3. 先让失败可恢复，再让成功更快。
4. 所有阶段都要有明确输入、输出、通过条件、失败条件。

## 版本路线

## V0.3: Runtime 收口版

目标: 让 Stack Forge 不再只是“生成 skill”，而是“程序主导 workflow 生命周期”。

### 要做什么

- 引入正式的 stage contract:
  - 每个 stage 定义 input artifacts、output artifacts、required checks。
- 把 stage 完成条件写进代码，而不是只写在模板里。
- `runWorkflow()` 不只打印“去 Claude Code 里运行 `/cforge`”，而是至少明确创建一次受控执行入口。
- `StateManager` 升级为显式状态机:
  - 只允许合法 stage transition。
  - 记录 started_at / completed_at / failed_at / retries。
- 为每个 stage 引入 artifact manifest:
  - 例如 `proposal.md`、`tasks.md`、`review.md` 是否存在、是否非空、是否满足基本格式。

### 建议新增目录

```text
src/runtime/
  contracts.ts
  transitions.ts
  artifacts.ts
  execution.ts
```

### 成功标准

- 用户执行 workflow 后，每个 stage 的进入、完成、失败都能被程序判定。
- 缺少必要 artifact 时不能进入下一个 stage。
- state 文件不再只是日志，而是 workflow 单一事实来源。

### 暂时不要做

- 不要急着支持很多新 workflow。
- 不要急着做复杂 UI。
- 不要急着接更多 provider。

## V0.4: Provider 健康检查升级版

目标: 把 healthcheck 从“存在性检查”升级为“最小能力验证”。

### 当前问题

- plugin installed
- skill exists
- mcp configured

这些只能证明“可能能用”，不能证明“真的能跑”。

### 要做什么

- 为每类 capability 定义 smoke task:
  - brainstorm: 能否根据一句需求产出结构化 proposal
  - specification: 能否根据 proposal 产出 spec 框架
  - review: 能否对一个小 diff 产出 review 结果
  - release: 能否识别 PR / branch / release 前置条件
- health record 增加:
  - `last_verified`
  - `verification_mode`
  - `sample_task`
  - `failure_reason`
- 区分三类状态:
  - installed
  - detectable
  - executable

### 成功标准

- healthcheck 输出能告诉用户:
  - provider 是否已安装
  - 是否可发现
  - 是否通过最小真实任务验证

## V0.5: 并行执行做实版

目标: 把 README 里的“parallel implementation”从承诺变成真实能力。

### 当前问题

- 任务依赖分析主要在模板说明里。
- 没有真正的任务图、冲突检测器、批次调度器。

### 要做什么

- 把 `tasks.md` 解析成结构化任务:
  - task id
  - summary
  - touched files
  - explicit dependencies
  - inferred dependencies
- 新增调度器:
  - 根据文件冲突和依赖关系生成 batch plan
- 新增执行日志:
  - 每批任务谁执行
  - 哪些任务并行
  - 哪些任务因冲突降级为串行
- 并行前后加一致性校验:
  - git diff overlap check
  - test result aggregation
  - typecheck aggregation

### 建议新增目录

```text
src/implementation/
  parser.ts
  dependency-graph.ts
  scheduler.ts
  verifier.ts
```

### 成功标准

- 给定一个 `tasks.md`，程序能稳定产出 batch plan。
- 不再只靠模型自己理解哪些任务能并行。

## V0.6: 失败恢复和可观测性版

目标: workflow 失败后可恢复，而不是只能手工处理。

### 要做什么

- 为每个 stage 保存 execution snapshot。
- 引入 resume / retry / skip-with-reason 机制。
- 记录每一步失败原因、最近一次成功 artifact、最近一次 provider。
- 增加事件日志:
  - stage_started
  - artifact_written
  - validation_failed
  - provider_unhealthy
  - retry_scheduled

### 建议新增文件

```text
.cforge/events.jsonl
.cforge/executions/<stage>/<timestamp>.json
```

### 成功标准

- 任意 stage 中断后，用户可以执行 resume。
- 故障定位不再依赖读一堆 Markdown 和猜测。

## V0.7: Spec 对齐验证版

目标: 让 `validate` 真正验证“实现是否覆盖需求”，而不是只做静态字符串检查。

### 当前问题

现在的 requirement validation 更像规则检查器，不足以覆盖“行为是否满足 spec”。

### 要做什么

- 将 spec 拆成结构化 requirement:
  - requirement id
  - description
  - acceptance criteria
  - evidence type
- 为每条 requirement 绑定验证方式:
  - file exists
  - content pattern
  - exported symbol
  - test exists
  - test passes
  - command output match
- 允许 validation report 回写 state。

### 成功标准

- `cforge validate` 产出 requirement-by-requirement 报告。
- 可以明确指出“哪些需求已实现，哪些没有证据”。

## V1.0: 可扩展编排平台版

目标: 让 Stack Forge 成为 provider-neutral 的 workflow runtime，而不只是当前几种 provider 的打包器。

### 要做什么

- provider capability interface 稳定化。
- workflow 定义外部化:
  - feature workflow
  - bugfix workflow
  - refactor workflow
  - incident workflow
- 支持 provider ranking / fallback:
  - 主 provider 失败时自动切备用 provider
- 支持团队级配置:
  - repo preset
  - org preset
  - language preset

### 成功标准

- 新增 workflow 不需要改核心代码太多。
- 新增 provider 主要靠注册 capability 和 contract。

## 优先级排序

### 必须先做

1. V0.3 Runtime 收口
2. V0.4 健康检查升级
3. V0.5 并行执行做实

### 第二梯队

1. V0.6 失败恢复
2. V0.7 Spec 对齐验证

### 最后做

1. V1.0 平台化扩展

## 建议的实现顺序

### 第一周

- 定义 stage contract
- 重构 state transition
- 增加 artifact manifest

### 第二周

- provider smoke check
- health record 扩展
- health CLI 输出重构

### 第三周

- task parser
- dependency graph
- batch scheduler

### 第四周

- execution snapshot
- resume / retry
- event log

## 关键设计建议

### 1. 不要把“模板生成”误当成“执行引擎”

模板可以保留，但它应该是 runtime 的附属层，不应该是 runtime 本身。

### 2. 不要让 state 只是 JSON 文件

state 可以继续落盘为 JSON，但在代码层必须是一个严格状态机。

### 3. 不要让 validate 停留在字符串层

真正有价值的是 requirement evidence，而不是文件里有没有某句话。

### 4. 并行能力一定要结构化

并行不是 prompt 里写一句“请并行执行”就算完成，必须有:

- 任务图
- 冲突规则
- 批次规划
- 汇总验证

## 一个现实判断

如果只做表层增强，比如:

- 再加几个命令
- 再支持几个 provider
- 再写漂亮一点的 README

那这个项目会继续停留在“看起来很完整”的阶段。

真正能把它拉开差距的，是把 runtime 做硬，把验证做硬，把恢复做硬。

## 结论

Stack Forge 最值得肯定的地方，是它选对了问题: “如何把已有 agent 能力编排成开发流水线”。

它目前最大的缺陷，是“编排语义主要存在于 prompt，而不是程序”。

后续所有高价值迭代，基本都应该围绕这一个核心目标展开:

把“模型自觉执行 workflow”升级为“系统受控执行 workflow”。
