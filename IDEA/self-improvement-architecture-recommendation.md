# Skill 自我进化方案建议

更新时间：2026-06-30 11:36:46 CST

## 结论

推荐采用：

**外部记忆驱动的半自动进化**

而不是：

**直接改 `SKILL.md` 的全自动进化**

原因不是保守，而是工程上更稳、更容易落地，也更适合真实用户使用过程中的持续演化。

这个方案的核心思想是：

1. 先把用户使用过程中的经验稳定采集下来
2. 再把经验归纳成候选规则
3. 再决定这些规则应该晋升到哪一层
4. 最后才在验证通过后修改 skill 本体

也就是：

**先学会记账，再学会归纳，最后才学会改自己。**

---

## 为什么不推荐一开始就自动修改 `SKILL.md`

如果系统一开始就具备“根据一次反馈自动改 skill”的能力，会有几个明显问题：

1. 容易过拟合单次经验  
用户的一次纠正、一次失败、一次临时偏好，不一定值得上升为长期规则。

2. 容易污染 skill 本体  
`SKILL.md` 是长期行为规范。把局部经验直接写进去，长期会堆积很多互相冲突、重复、过窄的规则。

3. 很难验证修改是否真的有效  
如果没有中间层和评估回路，后续很难判断某次“进化”到底改善了什么，还是只是让 prompt 更长了。

4. 风险高于收益  
真实场景里，自我进化首先要解决的是稳定性和可回滚，而不是追求“自动改自己”这件事本身。

因此，推荐把“修改 `SKILL.md`”视为进化链路的最后一步，而不是第一步。

---

## 推荐架构

推荐采用四层架构：

1. `Capture`
2. `Distill`
3. `Promote`
4. `Validate`

其中第一版必须落地的是前 3 层，第 4 层可以先保守启用。

---

## 第一层：Capture

### 目标

稳定采集用户使用 skill 过程中的经验，不漏关键信号。

### 原则

- 只记录，不修改 skill 本体
- 记录事实，不急于抽象
- 优先收集高价值反馈，而不是收集一切

### 建议采集的经验类型

1. 成功模式
   例如：
   - 某种工作流明显提升成功率
   - 某段提示语能稳定引导正确行为
   - 某个脚本/工具组合多次有效

2. 错误模式
   例如：
   - 某类任务总是漏一步
   - 某类 instruction 容易被误解
   - 某个工具调用链经常失败

3. 用户纠正
   例如：
   - 用户说“不是这个意思”
   - 用户补充“以后这种情况优先这样做”
   - 用户否定某种输出风格或执行顺序

4. 功能缺口
   例如：
   - 用户反复提出某项未被 skill 覆盖的需求
   - 当前 skill 需要手工补充大量步骤

### 推荐存储结构

```text
.learnings/
  LEARNINGS.md
  ERRORS.md
  FEATURE_REQUESTS.md
```

### 推荐记录字段

每条经验至少包含：

- `timestamp`
- `skill_name`
- `task_summary`
- `signal_type`
- `what_happened`
- `user_feedback`
- `proposed_lesson`
- `source_reference`

### 示例

```markdown
## 2026-06-30T11:20:00+08:00
- skill_name: frontend-api-integration
- signal_type: user_correction
- task_summary: 用户要求生成联调方案
- what_happened: 首版输出偏文档化，缺少字段映射表
- user_feedback: “重点不是流程，是字段怎么对”
- proposed_lesson: 遇到接口联调任务，优先产出字段映射与兜底策略
- source_reference: session-2026-06-30-001
```

### 第一层的判断标准

如果第一层做得对，系统应该做到：

- 用户每次纠正都不会白白消失
- 错误能积累为样本，而不是变成一次性事故
- 后续归纳时能追溯原始上下文

---

## 第二层：Distill

### 目标

把原始经验归纳成可复用的候选规则。

### 原则

- 不从单条经验直接生成长期规则
- 只归纳“重复出现且可泛化”的内容
- 保留证据链，避免抽象过度

### 为什么这一层最重要

自我进化系统最容易失败的地方，不是不会记录，而是：

**把局部现象错当成通用规律。**

因此这一层的任务不是“生成更多规则”，而是“过滤噪音、识别值得晋升的模式”。

### 推荐新增文件

```text
.learnings/
  patterns.json
  promotion_candidates.md
```

### `patterns.json` 建议结构

```json
{
  "patterns": [
    {
      "pattern_key": "prioritize-field-mapping-in-api-integration",
      "summary": "接口联调类任务应优先输出字段映射与异常兜底",
      "evidence_count": 3,
      "distinct_tasks": 2,
      "last_seen": "2026-06-30",
      "signal_types": ["user_correction", "success_pattern"],
      "target_scope": "skill",
      "target_name": "frontend-api-integration",
      "candidate_rule": "For API integration tasks, prioritize field mapping and fallback handling before generic process description.",
      "evidence_refs": [
        "session-2026-06-28-003",
        "session-2026-06-29-002",
        "session-2026-06-30-001"
      ],
      "status": "candidate"
    }
  ]
}
```

### 不推荐一开始做复杂置信度模型

不建议第一版引入复杂评分公式，例如大量权重、置信度衰减、负反馈函数等。

第一版只用三个核心信号就够：

- `evidence_count`
- `distinct_tasks`
- `last_seen`

必要时补一个简单辅助判断：

- 是否来自明确用户反馈

### 第二层的判断标准

如果第二层做得对，系统应该做到：

- 不会因为一次偶然问题就发明一条长期规则
- 候选规则能清楚对应到原始证据
- 每条候选规则都能说清它解决什么问题

---

## 第三层：Promote

### 目标

决定候选规则应该晋升到哪里，而不是默认写回 `SKILL.md`。

### 原则

- 不是所有经验都值得进入 skill 本体
- 晋升层级应该反映规则的适用范围
- 越接近本体修改，门槛越高

### 推荐晋升层级

#### L1：保留在 `.learnings/`

适用于：

- 证据还不够
- 只在单类任务中出现过一次或两次
- 仍然无法确定是否可泛化

这是默认层级。

#### L2：晋升到项目级共享行为说明

适用于：

- 对多个 task 都有帮助
- 更像工作习惯、协作原则、质量门槛
- 不属于某个单一 skill 的专属规则

可写入的目标包括：

- `AGENTS.md`
- `CLAUDE.md`
- 项目共享操作规范

#### L3：生成 `SKILL.md` patch proposal

适用于：

- 明显属于某个 skill 的长期能力改进
- 已经跨任务重复验证
- 可以表达为稳定 instruction
- 不会破坏现有主线工作流

注意：这里仍然不是自动写回，而是**生成改动提案**。

#### L4：晋升为新 skill

适用于：

- 已经演化出独立工作流
- 与原 skill 只是弱耦合
- 多次复用时都表现为独立能力

例如：

- 多次任务中都需要相同的补充脚本
- 某类分析过程已形成完整方法链

### 推荐阈值

第一版建议使用朴素阈值：

- `evidence_count >= 3`
- `distinct_tasks >= 2`
- 最近 30 天内重复出现
- 不是明显项目特例
- 能清楚写成一条通用 instruction

### `promotion_candidates.md` 的作用

这个文件应该扮演“待审核改进清单”的角色，而不是最终事实库。

示例：

```markdown
# Promotion Candidates

## Candidate: prioritize-field-mapping-in-api-integration

- Target: `frontend-api-integration/SKILL.md`
- Evidence count: 3
- Distinct tasks: 2
- Last seen: 2026-06-30
- Why promote:
  Repeated user corrections show that the current skill overemphasizes generic process explanation and underemphasizes field-level integration work.

### Proposed patch direction

Add guidance that API integration tasks should prioritize:
- field mapping
- missing field handling
- fallback behavior
- default/mock strategy

### Evidence refs

- session-2026-06-28-003
- session-2026-06-29-002
- session-2026-06-30-001
```

### 第三层的判断标准

如果第三层做得对，系统应该做到：

- 经验不会直接冲进 skill 本体
- 每次晋升都有证据、有范围、有目标
- 可以让人类或 agent 很快判断“该不该升”

---

## 第四层：Validate

### 目标

在真正修改 `SKILL.md` 之前，验证候选规则是否确实提升效果。

### 原则

- 验证先于写回
- 新版本必须与旧版本比较
- 没有明显收益时，不要增加 prompt 复杂度

### 为什么这层要晚一点做

这层很重要，但不一定要作为第一版的阻塞条件，因为它实现成本更高。  
它需要：

- 稳定的测试 prompt
- 新旧版本对照
- 用户反馈或人工评估

因此建议第一阶段先产出“可审阅候选提案”，第二阶段再接入完整验证回路。

### 推荐验证方式

直接借用 `skill-creator` 的思路：

1. 准备 2 到 5 个真实测试 prompt
2. 使用旧 skill 跑一次
3. 使用候选新 skill 跑一次
4. 让用户比较输出差异
5. 只有在新版本稳定更好时，才正式写回

### 验证通过后的动作

如果验证通过，再执行：

1. 修改对应的 `SKILL.md`
2. 在变更中记录来源 pattern
3. 将 `promotion_candidates.md` 中该项标记为 promoted
4. 保留原始证据链，便于将来回滚或复审

---

## 推荐的最小可行系统

如果现在就开始做第一版，建议只做以下能力：

### 必做

1. 经验记录目录
2. 三类日志文件
3. 原始经验格式约定
4. `patterns.json`
5. `promotion_candidates.md`
6. 一套简单的晋升阈值

### 暂缓

1. 自动直接改 `SKILL.md`
2. 复杂置信度公式
3. 全自动 hook 改写
4. 大量记忆分层文件
5. 复杂统计模型

也就是说，第一版先让系统具备：

**自动沉淀经验 + 自动形成改进候选**

而不是：

**自动改写 skill 本体**

---

## 与当前桌面版本的差异

你当前桌面版本的优点是：

- 有完整的多记忆设计
- 有安全意识
- 有用户确认机制
- 有“自我纠正 + 自我改进”的整体视角

但它的问题是：

1. 过早把“修改 skill 文件”放进主闭环
2. 设计偏重理念，工程落地层偏弱
3. 置信度和模式规则看起来精确，但证据基础还不够
4. 容易让系统看起来会“自动进化”，但实际上缺少稳定中间层

我推荐的新方案，核心调整就是：

- 把 `.learnings/` 变成主数据库
- 把 `SKILL.md` 修改放到链路末端
- 先积累候选规则，再做本体进化

---

## 推荐的实施顺序

### 阶段 1：建立经验采集层

目标：

- 所有高价值经验都能落盘
- 经验结构统一
- 可以按 skill、时间、问题类型追溯

产出：

- `.learnings/LEARNINGS.md`
- `.learnings/ERRORS.md`
- `.learnings/FEATURE_REQUESTS.md`

### 阶段 2：建立模式归纳层

目标：

- 从原始记录里抽取候选模式
- 对模式做最基本去重和聚合

产出：

- `.learnings/patterns.json`

### 阶段 3：建立晋升提案层

目标：

- 不直接改 skill
- 先形成待审核提案

产出：

- `.learnings/promotion_candidates.md`

### 阶段 4：接入验证与写回

目标：

- 对候选规则做对照测试
- 通过后再改 `SKILL.md`

产出：

- 可验证的 skill 迭代闭环

---

## 最终推荐

如果目标是：

**让用户在持续使用过程中，skill 真的越来越好**

那么最合适的路径不是“让 skill 立刻自动改自己”，而是：

**让系统先稳定积累经验，自动提炼候选规则，再把通过验证的规则晋升为 skill 本体能力。**

一句话概括：

**先做“会学习的记忆系统”，再做“会修改自己的 skill”。**

这是最稳、最真实、也最适合长期演化的路线。
