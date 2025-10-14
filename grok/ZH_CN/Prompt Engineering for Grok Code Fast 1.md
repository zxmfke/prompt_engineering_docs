# Grok Code Fast 1 的提示工程

https://docs.x.ai/docs/guides/grok-code-prompt-engineering

## [对于使用代理编码工具的开发人员](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#for-developers-using-agentic-coding-tools)

`grok-code-fast-1` 是一个轻量级的代理模型，旨在作为您在最常见的编码工具中的结对程序员而表现出色。为了优化您的体验，我们提出了一些指导方针，以便您可以顺利完成日常编码任务。

### [提供必要的上下文](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#provide-the-necessary-context)

大多数编码工具会自行收集必要的上下文。但是，通过选择要用作上下文的特定代码来变得更具体通常会更好。这使得 `grok-code-fast-1` 能够专注于您的任务并防止不必要的偏差。尝试指定相关的文件路径、项目结构或依赖项，并避免提供不相关的上下文。

- 避免的无上下文提示

  > 改善错误处理

- 具有指定上下文的良好提示

  > 我的错误代码在 @errors.ts 中定义，您能否以此为参考，为我进行查询的 @sql.ts 添加适当的错误处理和错误代码

### [设定明确的目标和要求](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#set-explicit-goals-and-requirements)

清楚地定义您的目标以及您希望 `grok-code-fast-1` 解决的具体问题。详细而具体的查询可以带来更好的性能。尽量避免模糊或未指定的提示，因为它们可能导致次优的结果。

- 避免的模糊提示

  > 创建一个食物追踪器

- 良好、详细的提示

  > 创建一个食物追踪器，当我输入一个食物项目时，它会显示按不同营养素划分的每日卡路里消耗明细。使其既能让我看到概览，又能获得高层次的趋势。

### [不断完善您的提示](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#continually-refine-your-prompts)

`grok-code-fast-1` 是一种高效的模型，其速度是其他领先代理模型的 4 倍，成本是其 1/10。这使您能够以前所未有的速度和可负担性测试您的复杂想法。即使初始输出不完美，我们强烈建议利用独特的快速且经济高效的迭代来完善您的查询——使用上面的建议（例如，添加更多上下文）或通过引用第一次尝试中的特定失败。

- 具有改进的良好提示示例

  > 先前的方法没有考虑 IO 密集型进程，这会阻塞主线程，我们可能希望将其放在自己的线程循环中运行，这样它就不会阻塞事件循环，而不仅仅是使用异步库版本

### [分配代理任务](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#assign-agentic-tasks)

我们鼓励用户尝试将 `grok-code-fast-1` 用于代理式任务，而不是一次性查询。我们的 Grok 4 模型更适合一次性问答，而 `grok-code-fast-1` 是您使用工具在大量代码中导航以提供精确答案的理想伴侣。

一个很好的思考方式是：

- `grok-code-fast-1` 擅长快速、不知疲倦地为您找到答案或实施所需的更改。
- 当您预先提供所有必要的上下文时，Grok 4 最适合深入研究复杂的概念和棘手的调试。

------

## [对于通过 xAI API 构建编码代理的开发人员](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#for-developers-building-coding-agents-via-the-xai-api)

通过 `grok-code-fast-1`，我们希望将一个代理编码模型交到开发人员手中。除了我们的发布合作伙伴之外，我们欢迎所有开发人员在工具调用密集的领域中试用 `grok-code-fast-1`，因为其快速和低成本使其既高效又经济实惠，可以使用许多工具来找出正确的答案。

如博客文章中所述，`grok-code-fast-1` 是一个在其思考过程中穿插工具调用的推理模型。我们还通过与 OpenAI 兼容的 API 发送总结性思考，以提供更好的 UX 支持。更多 API 详细信息可以在 https://docs.x.ai/docs/guides/function-calling 找到。

### [推理内容](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#reasoning-content)

`grok-code-fast-1` 是一个推理模型，我们通过 `chunk.choices[0].delta.reasoning_content` 公开其思考轨迹。请注意，思考轨迹仅在使用流模式时才能访问。

### [使用原生工具调用](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#use-native-tool-calling)

`grok-code-fast-1` 提供对原生工具调用的一流支持，并且在设计时特别考虑了原生工具调用。我们鼓励您使用它，而不是基于 XML 的工具调用输出，这可能会损害性能。

### [提供详细的系统提示](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#give-a-detailed-system-prompt)

在您的系统提示中要详尽并提供许多细节。一个写得很好的系统提示，描述了任务、期望以及模型应该意识到的边缘情况，可以产生天壤之别的差异。有关更多灵感，请参阅上面的用户最佳实践。

### [向模型介绍上下文](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#introduce-context-to-the-model)

`grok-code-fast-1` 习惯于在初始用户提示中看到大量上下文。我们建议开发人员使用 XML 标签或 Markdown 格式的内容来标记上下文的各个部分，并为某些部分增加清晰度。描述性的 Markdown 标题/XML 标签及其相应的定义将使 `grok-code-fast-1` 能够更有效地使用上下文。

### [优化缓存命中](https://docs.x.ai/docs/guides/grok-code-prompt-engineering#optimize-for-cache-hits)

我们的缓存命中是 `grok-code-fast-1` 快速推理速度的一大贡献者。在模型按顺序使用多个工具的代理任务中，大部分前缀保持不变，因此会自动从缓存中检索以加快推理速度。我们建议不要更改或扩充提示历史记录，因为这可能导致缓存未命中，从而导致推理速度显着变慢。
