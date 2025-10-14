# 提示缓存

提示缓存是一项强大的功能，通过允许从提示中的特定前缀恢复来优化您的 API 使用。这种方法显著减少了重复性任务或具有一致元素的提示的处理时间和成本。

以下是如何使用 `cache_control` 块通过 Messages API 实现提示缓存的示例：

  ```python
  import anthropic

  client = anthropic.Anthropic()

  response = client.messages.create(
      model="claude-sonnet-4-5",
      max_tokens=1024,
      system=[
        {
          "type": "text",
          "text": "你是一个人工智能助手，任务是分析文学作品。你的目标是就主题、人物和写作风格提供富有见地的评论。\n",
        },
        {
          "type": "text",
          "text": "<《傲慢与偏见》的全部内容>",
          "cache_control": {"type": "ephemeral"}
        }
      ],
      messages=[{"role": "user", "content": "分析《傲慢与偏见》中的主要主题。"}],
  )
  print(response.usage.model_dump_json())

  # 再次使用相同的输入调用模型，直到缓存检查点
  response = client.messages.create(.....)
  print(response.usage.model_dump_json())
  ```

```JSON JSON
{"cache_creation_input_tokens":188086,"cache_read_input_tokens":0,"input_tokens":21,"output_tokens":393}
{"cache_creation_input_tokens":0,"cache_read_input_tokens":188086,"input_tokens":21,"output_tokens":393}
```

在此示例中，使用 `cache_control` 参数缓存了《傲慢与偏见》的全部文本。这使得可以在多个 API 调用中重用此大文本，而无需每次都重新处理。仅更改用户消息即可让您在利用缓存内容的同时提出有关该书的各种问题，从而加快响应速度并提高效率。

***

## 提示缓存的工作原理

当您发送启用提示缓存的请求时：

1. 系统会检查提示前缀（直到指定的缓存断点）是否已从最近的查询中缓存。
2. 如果找到，它将使用缓存的版本，从而减少处理时间和成本。
3. 否则，它将处理完整的提示，并在响应开始后缓存前缀。

这对于以下情况特别有用：

* 包含许多示例的提示
* 大量上下文或背景信息
* 具有一致指令的重复性任务
* 长时间的多轮对话

默认情况下，缓存的生命周期为 5 分钟。每次使用缓存内容时，缓存都会免费刷新。

<Note>
  如果您发现 5 分钟太短，Anthropic 还提供 1 小时的缓存持续时间。

  有关更多信息，请参阅[1 小时缓存持续时间](#1-小时缓存持续时间)。
</Note>

<Tip>
  **提示缓存会缓存完整的前缀**

  提示缓存会引用整个提示——`tools`、`system` 和 `messages`（按此顺序），直到并包括使用 `cache_control` 指定的块。
</Tip>

***

## 定价

提示缓存引入了新的定价结构。下表显示了每百万个 token 对每个支持的模型的定价：

| 模型 | 基本输入 Token | 5 分钟缓存写入 | 1 小时缓存写入 | 缓存命中和刷新 | 输出 Token |
| --- | --- | --- | --- | --- | --- |
| Claude Opus 4.1 | $15 / MTok | $18.75 / MTok | $30 / MTok | $1.50 / MTok | $75 / MTok |
| Claude Opus 4 | $15 / MTok | $18.75 / MTok | $30 / MTok | $1.50 / MTok | $75 / MTok |
| Claude Sonnet 4.5 | $3 / MTok | $3.75 / MTok | $6 / MTok | $0.30 / MTok | $15 / MTok |
| Claude Sonnet 4 | $3 / MTok | $3.75 / MTok | $6 / MTok | $0.30 / MTok | $15 / MTok |
| Claude Sonnet 3.7 | $3 / MTok | $3.75 / MTok | $6 / MTok | $0.30 / MTok | $15 / MTok |
| Claude Sonnet 3.5 ([已弃用](/en/docs/about-claude/model-deprecations)) | $3 / MTok | $3.75 / MTok | $6 / MTok | $0.30 / MTok | $15 / MTok |
| Claude Haiku 3.5 | $0.80 / MTok | $1 / MTok | $1.6 / MTok | $0.08 / MTok | $4 / MTok |
| Claude Opus 3 ([已弃用](/en/docs/about-claude/model-deprecations)) | $15 / MTok | $18.75 / MTok | $30 / MTok | $1.50 / MTok | $75 / MTok |
| Claude Haiku 3 | $0.25 / MTok | $0.30 / MTok | $0.50 / MTok | $0.03 / MTok | $1.25 / MTok |

<Note>
  上表反映了提示缓存的以下定价乘数：

  * 5 分钟缓存写入 token 是基本输入 token 价格的 1.25 倍
  * 1 小时缓存写入 token 是基本输入 token 价格的 2 倍
  * 缓存读取 token 是基本输入 token 价格的 0.1 倍
</Note>

***

## 如何实现提示缓存

### 支持的模型

目前支持在以下模型上进行提示缓存：

* Claude Opus 4.1
* Claude Opus 4
* Claude Sonnet 4
* Claude Sonnet 3.7
* Claude Sonnet 3.5 ([已弃用](/en/docs/about-claude/model-deprecations))
* Claude Haiku 3.5
* Claude Haiku 3
* Claude Opus 3 ([已弃用](/en/docs/about-claude/model-deprecations))

### 构建您的提示

将静态内容（工具定义、系统指令、上下文、示例）放在提示的开头。使用 `cache_control` 参数标记要缓存的可重用内容的结尾。

缓存前缀按以下顺序创建：`tools`、`system`，然后是 `messages`。此顺序形成一个层次结构，其中每个级别都建立在前一个级别的基础上。

#### 自动前缀检查的工作原理

**您只需在静态内容的末尾使用一个缓存断点，系统就会自动找到最长的匹配前缀。** 其工作原理如下：

* 当您添加 `cache_control` 断点时，系统会自动检查所有先前内容块边界（直到您显式断点之前的大约 20 个块）的缓存命中情况
* 如果这些先前位置中的任何一个与早期请求的缓存内容匹配，系统将使用最长的匹配前缀
* 这意味着您不需要多个断点来启用缓存——末尾的一个就足够了

#### 何时使用多个断点

如果您想执行以下操作，最多可以定义 4 个缓存断点：

* 缓存更改频率不同的不同部分（例如，工具很少更改，但上下文每天更新）
* 对缓存的内容有更多的控制权
* 确保在最终断点之前的 20 多个块的内容可以缓存

<Note>
  **重要限制**：自动前缀检查仅从每个显式断点向后查找大约 20 个内容块。如果您的提示在缓存断点之前有超过 20 个内容块，则除非您添加其他断点，否则不会检查该内容之前的缓存命中情况。
</Note>

### 缓存限制

可缓存的最小提示长度为：

* Claude Opus 4.1、Claude Opus 4、Claude Sonnet 4.5、Claude Sonnet 4、Claude Sonnet 3.7、Claude Sonnet 3.5（[已弃用](/en/docs/about-claude/model-deprecations)）和 Claude Opus 3（[已弃用](/en/docs/about-claude/model-deprecations)）为 1024 个 token
* Claude Haiku 3.5 和 Claude Haiku 3 为 2048 个 token

较短的提示无法缓存，即使标记了 `cache_control`。任何缓存少于此 token 数的请求都将在不缓存的情况下处理。要查看提示是否已缓存，请参阅响应使用情况[字段](/en/docs/build-with-claude/prompt-caching#tracking-cache-performance)。

对于并发请求，请注意，缓存条目仅在第一个响应开始后才可用。如果您需要并行请求的缓存命中，请在发送后续请求之前等待第一个响应。

目前，“ephemeral”是唯一支持的缓存类型，默认生命周期为 5 分钟。

### 理解缓存断点成本

**缓存断点本身不会增加任何成本。** 您只需为以下内容付费：

* **缓存写入**：当新内容写入缓存时（对于 5 分钟 TTL，比基本输入 token 多 25%）
* **缓存读取**：使用缓存内容时（基本输入 token 价格的 10%）
* **常规输入 token**：对于任何未缓存的内容

添加更多 `cache_control` 断点不会增加您的成本——您仍然根据实际缓存和读取的内容支付相同的金额。断点只是让您可以控制哪些部分可以独立缓存。

### 什么可以被缓存

请求中的大多数块都可以通过 `cache_control` 指定用于缓存。这包括：

* 工具：`tools` 数组中的工具定义
* 系统消息：`system` 数组中的内容块
* 文本消息：`messages.content` 数组中的内容块，适用于用户和助手轮次
* 图像和文档：`messages.content` 数组中的内容块，在用户轮次中
* 工具使用和工具结果：`messages.content` 数组中的内容块，在用户和助手轮次中

这些元素中的每一个都可以用 `cache_control` 标记，以启用该部分请求的缓存。

### 什么不能被缓存

虽然大多数请求块都可以被缓存，但也有一些例外：

* 思维块不能直接用 `cache_control` 缓存。但是，当思维块出现在先前的助手轮次中时，可以与其他内容一起缓存。以这种方式缓存时，它们在从缓存中读取时确实会算作输入 token。
* 子内容块（如[引文](/en/docs/build-with-claude/citations)）本身不能直接缓存。相反，请缓存顶级块。

  在引文的情况下，可以缓存作为引文来源的顶级文档内容块。这允许您通过缓存引文将引用的文档来有效地使用提示缓存和引文。
* 空文本块不能被缓存。

### 什么会使缓存失效

对缓存内容的修改可能会使部分或全部缓存失效。

如[构建您的提示](#构建您的提示)中所述，缓存遵循层次结构：`tools` → `system` → `messages`。每个级别的更改都会使该级别和所有后续级别失效。

下表显示了不同类型的更改会使缓存的哪些部分失效。✘ 表示缓存失效，而 ✓ 表示缓存仍然有效。

| 更改内容 | 工具缓存 | 系统缓存 | 消息缓存 | 影响 |
| --- | --- | --- | --- | --- |
| **工具定义** | ✘ | ✘ | ✘ | 修改工具定义（名称、描述、参数）会使整个缓存失效 |
| **Web 搜索切换** | ✓ | ✘ | ✘ | 启用/禁用 Web 搜索会修改系统提示 |
| **引文切换** | ✓ | ✘ | ✘ | 启用/禁用引文会修改系统提示 |
| **工具选择** | ✓ | ✓ | ✘ | 对 `tool_choice` 参数的更改仅影响消息块 |
| **图像** | ✓ | ✓ | ✘ | 在提示中的任何位置添加/删除图像都会影响消息块 |
| **思维参数** | ✓ | ✓ | ✘ | 对扩展思维设置（启用/禁用、预算）的更改会影响消息块 |
| **传递给扩展思维请求的非工具结果** | ✓ | ✓ | ✘ | 当在启用扩展思维的情况下在请求中传递非工具结果时，所有先前缓存的思维块都将从上下文中剥离，并且上下文中跟随这些思维块的任何消息都将从缓存中删除。有关更多详细信息，请参阅[使用思维块进行缓存](#caching-with-thinking-blocks)。 |

### 跟踪缓存性能

使用 API 响应中的这些字段监视缓存性能，在响应的 `usage` 中（如果[流式传输](/en/docs/build-with-claude/streaming)，则在 `message_start` 事件中）：

* `cache_creation_input_tokens`：创建新条目时写入缓存的 token 数。
* `cache_read_input_tokens`：为此请求从缓存中检索的 token 数。
* `input_tokens`：未从缓存中读取或用于创建缓存的输入 token 数。

### 有效缓存的最佳实践

要优化提示缓存性能：

* 缓存稳定、可重用的内容，如系统指令、背景信息、大上下文或频繁的工具定义。
* 将缓存的内容放在提示的开头以获得最佳性能。
* 策略性地使用缓存断点来分隔不同的可缓存前缀部分。
* 定期分析缓存命中率并根据需要调整您的策略。

### 针对不同用例进行优化

根据您的场景调整提示缓存策略：

* 对话代理：降低长时间对话（尤其是那些带有长指令或上传文档的对话）的成本和延迟。
* 编码助手：通过将代码库的相关部分或摘要版本保留在提示中来改进自动完成和代码库问答。
* 大型文档处理：在提示中包含完整的长篇材料（包括图像），而不会增加响应延迟。
* 详细的指令集：共享大量的指令、过程和示例列表，以微调 Claude 的响应。开发人员通常在提示中包含一两个示例，但通过提示缓存，您可以通过包含 20 多个不同类型的高质量答案示例来获得更好的性能。
* 代理工具使用：增强涉及多个工具调用和迭代代码更改的场景的性能，其中每个步骤通常都需要一个新的 API 调用。
* 与书籍、论文、文档、播客文本和其他长篇内容对话：通过将整个文档嵌入提示中，让用户向其提问，从而使任何知识库都变得生动起来。

### 故障排除常见问题

如果遇到意外行为：

* 确保缓存部分完全相同，并在所有调用中的相同位置用 cache_control 标记
* 检查调用是否在缓存生命周期内进行（默认为 5 分钟）
* 验证 `tool_choice` 和图像使用在调用之间是否保持一致
* 验证您是否至少缓存了最小数量的 token
* 系统会自动检查先前内容块边界（直到断点之前约 20 个块）的缓存命中情况。对于具有超过 20 个内容块的提示，您可能需要在提示的更早位置添加额外的 `cache_control` 参数，以确保所有内容都可以被缓存
* 验证 `tool_use` 内容块中的键是否具有稳定的顺序，因为某些语言（例如 Swift、Go）在 JSON 转换期间会随机化键顺序，从而破坏缓存

<Note>
  对 `tool_choice` 或提示中任何位置图像的存在/不存在的更改都将使缓存失效，需要创建新的缓存条目。有关缓存失效的更多详细信息，请参阅[什么会使缓存失效](#什么会使缓存失效)。
</Note>

### 使用思维块进行缓存

当将[扩展思维](/en/docs/build-with-claude/extended-thinking)与提示缓存一起使用时，思维块具有特殊的行为：

**与其他内容一起自动缓存**：虽然思维块不能用 `cache_control` 显式标记，但当您使用工具结果进行后续 API 调用时，它们会作为请求内容的一部分被缓存。这通常在工具使用期间发生，当您传递思维块以继续对话时。

**输入 token 计数**：当从缓存中读取思维块时，它们会算作您使用情况指标中的输入 token。这对于成本计算和 token 预算很重要。

**缓存失效模式**：

* 当仅提供工具结果作为用户消息时，缓存仍然有效
* 当添加非工具结果用户内容时，缓存会失效，导致所有先前的思维块被剥离
* 即使没有显式的 `cache_control` 标记，也会发生此缓存行为

有关缓存失效的更多详细信息，请参阅[什么会使缓存失效](#什么会使缓存失效)。

**工具使用示例**：

```
请求 1：用户：“巴黎的天气怎么样？”
响应：[thinking_block_1] + [tool_use block 1]

请求 2：
用户：[“巴黎的天气怎么样？”]，
助手：[thinking_block_1] + [tool_use block 1]，
用户：[tool_result_1, cache=True]
响应：[thinking_block_2] + [text block 2]
# 请求 2 缓存其请求内容（而不是响应）
# 缓存包括：用户消息、thinking_block_1、tool_use block 1 和 tool_result_1

请求 3：
用户：[“巴黎的天气怎么样？”]，
助手：[thinking_block_1] + [tool_use block 1]，
用户：[tool_result_1, cache=True]，
助手：[thinking_block_2] + [text block 2]，
用户：[Text response, cache=True]
# 非工具结果用户块导致所有思维块被忽略
# 此请求的处理方式就好像思维块从未存在过一样
```

当包含非工具结果用户块时，它会指定一个新的助手循环，并且所有先前的思维块都会从上下文中删除。

有关更多详细信息，请参阅[扩展思维文档](/en/docs/build-with-claude/extended-thinking#understanding-thinking-block-caching-behavior)。

***

## 缓存存储和共享

* **组织隔离**：缓存是组织之间隔离的。不同的组织永远不会共享缓存，即使它们使用相同的提示。

* **完全匹配**：缓存命中需要 100% 相同的提示段，包括所有文本和图像，直到并包括用缓存控制标记的块。

* **输出 Token 生成**：提示缓存对输出 token 生成没有影响。您收到的响应将与不使用提示缓存时收到的响应完全相同。

***

## 1 小时缓存持续时间

如果您发现 5 分钟太短，Anthropic 还提供 1 小时的缓存持续时间。

要使用扩展缓存，请在 `cache_control` 定义中包含 `ttl`，如下所示：

```JSON
"cache_control": {
    "type": "ephemeral",
    "ttl": "5m" | "1h"
}
```

响应将包括详细的缓存信息，如下所示：

```JSON
{
    "usage": {
        "input_tokens": ...,
        "cache_read_input_tokens": ...,
        "cache_creation_input_tokens": ...,
        "output_tokens": ...,
        
        "cache_creation": {
            "ephemeral_5m_input_tokens": 456,
            "ephemeral_1h_input_tokens": 100,
        }
    }
}
```

请注意，当前的 `cache_creation_input_tokens` 字段等于 `cache_creation` 对象中值的总和。

### 何时使用 1 小时缓存

如果您的提示以常规节奏使用（即，系统提示的使用频率高于每 5 分钟一次），请继续使用 5 分钟缓存，因为这将继续免费刷新。

1 小时缓存在以下情况下最适用：

* 当您的提示可能使用频率低于 5 分钟，但高于每小时一次时。例如，当一个代理的副代理需要超过 5 分钟，或者当存储与用户的长时间聊天对话，并且您通常期望该用户可能不会在接下来的 5 分钟内回复时。
* 当延迟很重要并且您的后续提示可能会在 5 分钟后发送时。
* 当您希望提高速率限制利用率时，因为缓存命中不会从您的速率限制中扣除。

<Note>
  5 分钟和 1 小时缓存在延迟方面表现相同。对于长文档，您通常会看到首个 token 生成时间的改善。
</Note>

### 混合使用不同的 TTL

您可以在同一个请求中同时使用 1 小时和 5 分钟的缓存控制，但有一个重要的限制：具有较长 TTL 的缓存条目必须出现在较短 TTL 的条目之前（即，1 小时缓存条目必须出现在任何 5 分钟缓存条目之前）。

混合使用 TTL 时，我们在您的提示中确定三个计费位置：

1. 位置 `A`：最高缓存命中时的 token 计数（如果没有命中，则为 0）。
2. 位置 `B`：在 `A` 之后最高的 1 小时 `cache_control` 块处的 token 计数（如果不存在，则等于 `A`）。
3. 位置 `C`：最后一个 `cache_control` 块处的 token 计数。

<Note>
  如果 `B` 和/或 `C` 大于 `A`，它们必然是缓存未命中，因为 `A` 是最高的缓存命中。
</Note>

您需要为以下内容付费：

1. `A` 的缓存读取 token。
2. `(B - A)` 的 1 小时缓存写入 token。
3. `(C - B)` 的 5 分钟缓存写入 token。

这里有 3 个例子。这描绘了 3 个请求的输入 token，每个请求都有不同的缓存命中和缓存未命中。因此，每个请求都有不同的计算定价，如彩色框所示。
<img src="https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=10a8997695f0f78953fdac300a3373e9" alt="混合 TTL 图" data-og-width="1376" width="1376" data-og-height="976" height="976" data-path="images/prompt-cache-mixed-ttl.svg" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=280&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=7a8de34e52bbf67c60b2eeda57690ea3 280w, https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=560&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=cdc3a1950dc88fbfb5679320df656ef2 560w, https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=840&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=0df34408f5ec905ade69060ac8b5077b 840w, https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=1100&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=0c434f1b04d12e6a0a20cfe58b22d4e5 1100w, https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=1650&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=c800577f55f6e3383e3807644a5e0743 1650w, https://mintcdn.com/anthropic-claude-docs/LF5WV0SNF6oudpT5/images/prompt-cache-mixed-ttl.svg?w=2500&fit=max&auto=format&n=LF5WV0SNF6oudpT5&q=85&s=7e7079d06a969ea814980f4e570d2fa2 2500w" />

***

## 提示缓存示例

为了帮助您开始使用提示缓存，我们准备了一个[提示缓存手册](https://github.com/anthropics/anthropic-cookbook/blob/main/misc/prompt_caching.ipynb)，其中包含详细的示例和最佳实践。

下面，我们包含了一些展示各种提示缓存模式的代码片段。这些示例演示了如何在不同场景中实现缓存，帮助您理解此功能的实际应用：

### 大上下文缓存示例



<AccordionGroup>
  <Accordion title="大上下文缓存示例">
    <CodeGroup>

      ```bash Shell
        ```Python Python
        import anthropic
        client = anthropic.Anthropic()
      
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            system=[
                {
                    "type": "text",
                    "text": "你是一个人工智能助手，任务是分析法律文件。"
                },
                {
                    "type": "text",
                    "text": "这是一份复杂的法律协议的全文：[在此处插入一份 50 页法律协议的全文]",
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            messages=[
                {
                    "role": "user",
                    "content": "本协议中的关键条款和条件是什么？"
                }
            ]
        )
        print(response.model_dump_json())
      ```

此示例演示了基本的提示缓存用法，将法律协议的全文作为前缀缓存，同时保持用户指令不被缓存。

对于第一个请求：

* `input_tokens`：仅用户消息中的 token 数
* `cache_creation_input_tokens`：整个系统消息中的 token 数，包括法律文件
* `cache_read_input_tokens`：0（第一个请求没有缓存命中）

对于缓存生命周期内的后续请求：

* `input_tokens`：仅用户消息中的 token 数
* `cache_creation_input_tokens`：0（没有新的缓存创建）
* `cache_read_input_tokens`：整个缓存系统消息中的 token 数

### 缓存工具定义

      ```python
      import anthropic
        client = anthropic.Anthropic()
      
        response = client.messages.create(
            model="claude-sonnet-4-5",
            max_tokens=1024,
            tools=[
                {
                    "name": "get_weather",
                    "description": "获取给定位置的当前天气",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "location": {
                                "type": "string",
                                "description": "城市和州，例如 San Francisco, CA"
                            },
                            "unit": {
                                "type": "string",
                                "enum": ["celsius", "fahrenheit"],
                                "description": "温度单位，'celsius' 或 'fahrenheit'"
                            }
                        },
                        "required": ["location"]
                    },
                },
                # 更多工具
                {
                    "name": "get_time",
                    "description": "获取给定区域的当前时间",
                    "input_schema": {
                        "type": "object",
                        "properties": {
                            "timezone": {
                                "type": "string",
                                "description": "IANA 时区名称，例如 America/Los_Angeles"
                            }
                        },
                        "required": ["timezone"]
                    },
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            messages=[
                {
                    "role": "user",
                    "content": "纽约的天气和时间是多少？"
                }
            ]
        )
        print(response.model_dump_json())
      ```

在此示例中，我们演示了缓存工具定义。`cache_control` 参数放在最后一个工具（`get_time`）上，以将所有工具指定为静态前缀的一部分。这意味着所有工具定义，包括 `get_weather` 和在 `get_time` 之前定义的任何其他工具，都将作为一个前缀缓存。当您有一组一致的工具，并希望在多个请求中重用它们而无需每次重新处理时，此方法很有用。对于第一个请求：

- `input_tokens`：用户消息中的 token 数
- `cache_creation_input_tokens`：所有工具定义和系统提示中的 token 数
- `cache_read_input_tokens`：0（第一个请求没有缓存命中）

对于缓存生命周期内的后续请求：

- `input_tokens`：用户消息中的 token 数
- `cache_creation_input_tokens`：0（没有新的缓存创建）
- `cache_read_input_tokens`：所有缓存的工具定义和系统提示中的 token 数

### 继续多轮对话

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "...长系统提示",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        # ...到目前为止的长对话
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "你好，你能告诉我更多关于太阳系的信息吗？",
                }
            ]
        },
        {
            "role": "assistant",
            "content": "当然！太阳系是围绕我们的太阳运行的天体的集合。它由八大行星、众多卫星、小行星、彗星和其他物体组成。从离太阳最近到最远的顺序，行星分别是：水星、金星、地球、火星、木星、土星、天王星和海王星。每颗行星都有其独特的特征和特点。您想了解太阳系的哪个特定方面？"
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "知道了。"
                },
                {
                    "type": "text",
                    "text": "告诉我更多关于火星的信息。",
                    "cache_control": {"type": "ephemeral"}
                }
            ]
        }
    ]
)
print(response.model_dump_json())
```
在此示例中，我们演示了如何在多轮对话中使用提示缓存。在每一轮中，我们用 `cache_control` 标记最后一条消息的最后一个块，以便可以增量缓存对话。系统将自动查找并使用先前缓存的最长前缀用于后续消息。也就是说，以前用 `cache_control` 块标记的块后来没有用这个标记，但如果它们在 5 分钟内被命中，它们仍然会被认为是缓存命中（也是缓存刷新！）。此外，请注意 `cache_control` 参数放在系统消息上。这是为了确保如果它从缓存中被逐出（在超过 5 分钟未使用后），它将在下一次请求时被添加回缓存。此方法对于在正在进行的对话中保持上下文而无需重复处理相同信息很有用。正确设置后，您应该在每个请求的使用情况响应中看到以下内容：

- `input_tokens`：新用户消息中的 token 数（将是最小的）
- `cache_creation_input_tokens`：新助手和用户轮次中的 token 数
- `cache_read_input_tokens`：直到上一轮对话中的 token 数

 ### 综合运用：多个缓存断点

```python
import anthropic
client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=[
        {
            "name": "search_documents",
            "description": "搜索知识库",
            "input_schema": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索查询"
                    }
                },
                "required": ["query"]
            }
        },
        {
            "name": "get_document",
            "description": "按 ID 检索特定文档",
            "input_schema": {
                "type": "object",
                "properties": {
                    "doc_id": {
                        "type": "string",
                        "description": "文档 ID"
                    }
                },
                "required": ["doc_id"]
            },
            "cache_control": {"type": "ephemeral"}
        }
    ],
    system=[
        {
            "type": "text",
            "text": "你是一个有用的研究助手，可以访问文档知识库。\n\n# 指令\n- 在回答之前总是搜索相关文档\n- 为你的来源提供引文\n- 在你的回答中保持客观和准确\n- 如果多个文档包含相关信息，请综合它们\n- 当知识库中没有信息时承认",
            "cache_control": {"type": "ephemeral"}
        },
        {
            "type": "text",
            "text": "# 知识库上下文\n\n以下是本次对话的相关文档：\n\n## 文档 1：太阳系概述\n太阳系由太阳和所有围绕它运行的物体组成...\n\n## 文档 2：行星特征\n每颗行星都有独特的特征。水星是最小的行星...\n\n## 文档 3：火星探索\n几十年来，火星一直是探索的目标...\n\n[其他文档...]",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[
        {
            "role": "user",
            "content": "你能搜索关于火星探测器的信息吗？"
        },
        {
            "role": "assistant",
            "content": [
                {
                    "type": "tool_use",
                    "id": "tool_1",
                    "name": "search_documents",
                    "input": {"query": "火星探测器"}
                }
            ]
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "tool_result",
                    "tool_use_id": "tool_1",
                    "content": "找到 3 个相关文档：文档 3（火星探索）、文档 7（探测器技术）、文档 9（任务历史）"
                }
            ]
        },
        {
            "role": "assistant",
            "content": [
                {
                    "type": "text",
                    "text": "我找到了 3 个关于火星探测器的相关文档。让我从火星探索文档中获取更多详细信息。"
                }
            ]
        },
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "是的，请具体告诉我关于“毅力号”探测器的信息。",
                    "cache_control": {"type": "ephemeral"}
                }
            ]
        }
    ]
)
print(response.model_dump_json())
```
这个综合示例演示了如何使用所有 4 个可用的缓存断点来优化提示的不同部分：

1. **工具缓存**（缓存断点 1）：最后一个工具定义上的 `cache_control` 参数缓存所有工具定义。
2. **可重用指令缓存**（缓存断点 2）：系统提示中的静态指令被单独缓存。这些指令在请求之间很少更改。
3. **RAG 上下文缓存**（缓存断点 3）：知识库文档被独立缓存，允许您更新 RAG 文档而不会使工具或指令缓存失效。
4. **对话历史缓存**（缓存断点 4）：助手的响应被标记为 `cache_control`，以在对话进行时启用对话的增量缓存。

这种方法提供了最大的灵活性：

- 如果您只更新最后的用户消息，则所有四个缓存段都将被重用
- 如果您更新 RAG 文档但保持相同的工具和指令，则前两个缓存段将被重用
- 如果您更改对话但保持相同的工具、指令和文档，则前三个段将被重用
- 每个缓存断点都可以根据您的应用程序中的更改独立失效

对于第一个请求：

- `input_tokens`：最后用户消息中的 token
- `cache_creation_input_tokens`：所有缓存段中的 token（工具 + 指令 + RAG 文档 + 对话历史）
- `cache_read_input_tokens`：0（没有缓存命中）

对于仅有新用户消息的后续请求：

- `input_tokens`：仅新用户消息中的 token
- `cache_creation_input_tokens`：添加到对话历史中的任何新 token
- `cache_read_input_tokens`：所有先前缓存的 token（工具 + 指令 + RAG 文档 + 先前对话）

这种模式对于以下情况尤其强大：

- 具有大文档上下文的 RAG 应用程序
- 使用多个工具的代理系统
- 需要维护上下文的长时间运行的对话
- 需要独立优化提示不同部分的应用程序

