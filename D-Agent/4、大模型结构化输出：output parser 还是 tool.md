# 大模型结构化输出：output parser 还是 tool

## 1. 这类能力是为了解决什么问题

大模型默认输出的是自然语言，但很多实际场景并不希望得到“看起来像对的文本”，而是希望拿到可以直接被程序消费的结构化结果，例如：

- JSON 对象
- 固定字段的表单数据
- XML 片段
- 工具调用参数

这时就需要对模型输出施加“格式约束”。

## 2. 两条主路线

做结构化输出，通常有两种路线：

### 路线一：`output parser`

它的核心思路是：

1. 在提示词中明确告诉模型应该输出什么格式
2. 模型先生成文本
3. 再由 parser 对这段文本做解析和校验

常见类型包括：

- `JsonOutputParser`
- `StructuredOutputParser`
- `XMLOutputParser`
- `JsonOutputToolsParser`

可以把它理解为：

> 先靠提示词“约束格式”，再靠解析器“把文本转成结构化数据”。

### 路线二：`withStructuredOutput`

它更像一个统一封装好的高级 API。

它的核心价值是：

- 你只需要声明自己要的结构
- 底层自动根据模型能力选择实现方式
- 可能使用 tool call
- 也可能退回到 output parser

因此，在多数“只想要稳定结构化结果”的场景里，`withStructuredOutput` 是更直接的选择。

## 3. 为什么 tool call 通常更可靠

如果模型支持 tool call，那么结构化输出通常优先考虑它，而不是单纯依赖 parser。

原因在于：

- tool call 是模型能力的一部分，不只是提示词技巧
- 模型在训练阶段就学过“按指定 schema 产出参数”
- 输出不是自由文本，而是更接近受约束的参数对象
- 结构稳定性通常比“先生成文本再 parse”更高

所以从工程实践上说：

> 如果目标只是得到结构化数据，优先考虑 `withStructuredOutput`。

## 4. `withStructuredOutput` 为什么常常是首选

可以把它看成一个“默认推荐入口”。

它适合的原因：

- API 更统一，使用成本低
- 不需要自己手动拼很多格式提示词
- 能自动利用模型的 tool call 能力
- 当底层模型能力不同的时候，适配逻辑由框架处理

一句话总结：

> 你关心“我要什么结构”，框架关心“怎么让模型产出来”。

## 5. `withStructuredOutput` 不适合的两个典型场景

虽然 `withStructuredOutput` 很方便，但并不是所有场景都最优。

### 场景一：需要流式打印

如果你希望模型一边生成、一边把内容实时输出给前端或终端，那么通常需要 `output parser`。

原因是：

- tool call 往往更偏“等参数整体成型后再返回”
- 流式过程中，很多时候你想逐步看到内容
- parser 更适合对流式文本边接收边处理

因此：

> 只要你的重点是“流式展示”，通常应优先考虑 `output parser`。

### 场景二：目标格式不是 JSON，而是 XML 等其他格式

tool call 的天然目标通常是结构化参数对象，本质上更接近 JSON / schema。

如果你明确需要：

- XML
- 自定义标记格式
- 某种固定模板文本

那么 `output parser` 更灵活，例如：

- `XMLOutputParser`

因此：

> 只要目标不是 JSON 风格结构，而是 XML 或其他文本格式，应优先考虑 `output parser`。

## 6. 特殊情况：流式拿到 `tool_calls` 参数

有一个比较细的场景容易混淆：

你虽然在使用工具调用思路，但在“流式输出 tool 参数”的过程中，希望实时拿到 `tool_calls` 对应的 JSON 对象，并尽快触发工具执行。

这时可以使用：

- `JsonOutputToolsParser`

它适合的场景是：

- 输出本质和工具调用相关
- 你不只是想等最终结果
- 而是想在流式过程中尽早拿到可解析的工具参数

这个点可以理解为：

> 它是“工具调用场景下，偏流式解析”的补充方案。

## 7. 应该怎么选

最实用的判断方法可以直接记成下面这张表。

| 场景 | 推荐方案 |
| --- | --- |
| 只想稳定拿到结构化 JSON / schema 数据 | `withStructuredOutput` |
| 模型支持 tool call，追求可靠性 | `withStructuredOutput` |
| 需要流式打印内容 | `output parser` |
| 需要 XML 等非 JSON 格式 | `output parser` |
| 需要在流式过程中提早拿到 `tool_calls` JSON | `JsonOutputToolsParser` |

## 8. 一句话决策法

可以记成下面这套非常实用的规则：

- 默认做结构化输出：先用 `withStructuredOutput`
- 需要流式输出：改用 `output parser`
- 需要 XML 或其他非 JSON 格式：改用 `output parser`
- 需要流式解析工具参数：用 `JsonOutputToolsParser`

## 9. 你可以怎么理解这两者的关系

不要把它们理解成互相替代得很绝对的两套体系，而应该理解成：

- `withStructuredOutput` 是更推荐的工程化入口
- `output parser` 是更灵活的底层控制手段

也就是说：

- 当你追求“少操心、够稳定、直接拿结构”，选 `withStructuredOutput`
- 当你追求“流式、定制格式、解析过程可控”，选 `output parser`

## 10. 最终结论

这段内容可以压缩成一句最重要的话：

> 做大模型结构化输出时，通常在 `withStructuredOutput` 和 `output parser` 之间二选一：默认优先 `withStructuredOutput`，遇到流式输出或非 JSON 格式时再用 `output parser`。

## 11. 复习版速记

### 速记一

- 想要结构化结果：先想 `withStructuredOutput`
- 想要流式过程：再想 `output parser`
- 想要 XML：用 `XMLOutputParser`
- 想在流式中拿工具参数：用 `JsonOutputToolsParser`

### 速记二

`withStructuredOutput = 默认首选`

`output parser = 流式 / XML / 自定义格式时使用`

### 速记三

最重要的不是“有没有 parser”，而是先判断你的目标到底是：

- 稳定结构化结果
- 还是流式、可控、非 JSON 的输出过程

判断清楚这个问题，选型就不会乱。
