# Prompt Caching 全面解析

Prompt caching（提示词缓存）是大语言模型 API 中一项重要的优化功能。当你重复使用相同或相似的提示词时，缓存机制可以显著降低成本和延迟。尤其在考虑成本的时候，不应只看模型的 input 和 output 价格，cache 对成本影响也很大，有的模型可以到第一次成本的 10%。本文将详细介绍各大 AI 提供商的 prompt caching 实现，包括 Claude (Anthropic)、OpenAI、Gemini (Google)。

## 什么是 Prompt Caching？

Prompt caching 的核心思想是：**缓存提示词中重复使用的部分，避免重复处理相同的内容**。在实际应用中，我们经常会遇到以下场景：

- 使用相同的系统提示词和指令
- 反复分析同一份文档或代码库
- 在多轮对话中保持上下文
- 使用相同的工具定义和示例

通过缓存这些重复内容，可以实现：
- **降低成本**：缓存命中时的价格通常是正常输入 token 的 10%-25%
- **减少延迟**：缓存命中可以减少 80% 的处理时间
- **提升体验**：更快的响应速度带来更好的用户体验

### 为什么可以做 Prompt Caching？

Prompt caching 之所以可行，是因为大模型推理可以分为“预填充（prefill）”和“解码（decode）”两个阶段：前者负责把整段提示词转成 token 并运行一次长上下文的自注意力，这部分计算量占了绝大多数；后者则是在已有上下文上逐步生成新 token，成本远低于预填充。通过缓存，服务端可以在多次请求中复用前一次预填充得到的状态，从而绕过最昂贵的计算。

背后的关键技术和基础设施包括：

- **Transformer 的前缀可复用特性**：模型在预填充阶段会生成 Key/Value（KV）缓存。只要后续请求的开头 token 完全一致，这些 KV 张量就可以直接复用，模型仅需继续解码新增的部分。

- **确定性的分词与归一化流程**：服务端会对请求内容进行严格的 token 归一化、去除无意义差异并计算哈希值，确保“相同提示词”在缓存层看来也是一致的二进制表示。

- **分层存储与 TTL 管理**：为了兼顾性能与成本，缓存通常分布在内存、快速存储等不同层级，并通过 TTL（如 5 分钟或 1 小时）来控制生存周期，避免无效数据占用资源。

- **与速率控制、路由协同**：缓存命中率越高，请求的 CPU/GPU 占用越低。厂商会结合负载均衡和速率限制，把相同前缀的请求路由到同一组机器，提高命中概率。

从开发者角度看，“显性（explicit）缓存”与“隐形（implicit）缓存”只是标记方式不同，但底层原理一致：

- **显性缓存**要求你明确告诉 API 哪些段落可以缓存（如 Claude 的 `cache_control`、Gemini 的缓存对象）。这样可以精细控制缓存范围、TTL 和失效策略，适合复杂多模块的提示词结构。
- **隐形缓存**则由平台自动比对提示词前缀（如 OpenAI 自动缓存、Gemini 2.5 默认隐式缓存）。你无需写额外代码，只要保持提示词结构稳定，系统就会尝试命中缓存。

因此，prompt caching 本质上是利用 Transformer 前缀可复用的数学特性，加上云端的分布式缓存与路由工程能力，来在多次调用之间共享昂贵的预填充计算。这与大模型技术高度相关：如果没有超长上下文的 Transformer、确定性的分词流程和可扩展的 KV 缓存管理，就无法实现如今的 prompt caching 体验。

---

## 一、Claude (Anthropic) 的 Prompt Caching

### 1.1 核心特性

Claude 提供了最为灵活和功能丰富的 prompt caching 实现：

- **手动控制**：通过 `cache_control` 参数显式标记要缓存的内容
- **多重缓存点**：支持最多 4 个缓存断点，实现细粒度控制
- **双重 TTL**：支持 5 分钟和 1 小时两种缓存时长
- **自动前缀查找**：系统会自动查找最长匹配的缓存前缀

### 1.2 工作原理

Claude 的缓存机制按照特定的层级顺序工作：`tools` → `system` → `messages`

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "你是一个文学分析助手...",
        },
        {
            "type": "text",
            "text": "<Pride and Prejudice 的完整内容>",
            "cache_control": {"type": "ephemeral"}  # 标记为可缓存
        }
    ],
    messages=[{
        "role": "user", 
        "content": "分析《傲慢与偏见》的主题"
    }]
)
```

### 1.3 Content Block 如何理解？

在 Claude 的 API 里，每一条消息的 `content` 字段都是一个“内容块（content block）”列表。常见的内容块类型包括：

- `{"type": "text", "text": "..."}`
- `{"type": "tool_use", ...}` / `{"type": "tool_result", ...}`
- `{"type": "image", "source": {...}}`

缓存系统正是以 content block 为单位来判断前缀是否一致。理解这一点有几个要点：

- **自动前缀匹配的上限**：Claude 只会在最多 20 个 content block 内寻找可复用的缓存前缀。如果提示词由大量小块组成，超过上限的部分不会参与自动匹配，需要通过 `cache_control` 显式标记。

- **分层缓存的粒度**：当你在 `tools`、`system`、`messages` 中插入缓存断点时，服务端会把这些片段拆分成多个 content block 并分别存储。只要其中某个块被修改，**对应的缓存就会失效，但其他未变的块仍可复用**。

- **多模态与工具输出**：图片、音频、tool result 等也都是 content block。缓存这些块能显著降低多模态场景的预处理成本，但前提是它们在多次请求之间保持完全一致。

因此，在设计 Claude 的提示词结构时，建议将相对稳定的大块内容（系统指令、长文档、工具定义等）拆成合理数量的 content block，并使用 `cache_control` 显式声明缓存策略，从而既满足自动前缀匹配的上限要求，又能精细控制失效范围。

### 1.4 缓存限制

| 模型类型 | 最小缓存长度 |
|---------|------------|
| Opus 4, Sonnet 4/3.7/3.5 | 1024 tokens |
| Haiku 3.5/3 | 2048 tokens |

**重要**：自动前缀查找只会回溯约 20 个内容块。如果提示词中有超过 20 个内容块，需要添加额外的缓存断点。

### 1.5 自动前缀查找 vs. 手动缓存点

Claude 的缓存机制结合了自动匹配和手动控制，以提供灵活性。

**自动前缀查找 (Automatic Prefix Lookup):**

是的，Claude **可以自动查找**最长匹配的缓存前缀，**即使你没有设置任何 `cache_control` 断点**。当你发送一个请求时，Claude 会自动将提示词的开头部分与之前缓存的内容进行比较。如果找到了匹配的前缀，它就会利用这个缓存。

**那么，为什么还需要手动设置缓存点呢？**

自动前缀查找有一个重要限制：它只会回溯检查提示词中大约**前 20 个内容块（content blocks）**。如果你的静态前缀部分包含了超过 20 个块（例如，一个非常长的系统指令被分成了很多个 `{"type": "text", ...}` 块），自动查找就可能失败。

**手动设置 `cache_control` 的优势在于：**
- **精确控制**：你可以明确告诉系统：“缓存到此为止”，确保缓存边界符合你的预期，不受 20 个块的限制。

- **多层缓存**：这是手动断点最强大的功能。通过设置多个断点，你可以创建独立的缓存层（如工具、指令、文档），实现变化隔离。自动前缀查找无法实现这种分层缓存。

**总结**：对于简单、内容块不多的前缀，可以依赖自动查找。但对于复杂的、多部分的提示词，或者想要实现精细的、多层级的缓存策略时，**强烈建议使用手动 `cache_control` 断点**。

### 1.6 多缓存断点策略

Claude 的多缓存断点设计是其缓存机制中最强大和最灵活的部分。你可以把它想象成在你的提示词中设置多个独立的“存档点”。

**核心理念：分层缓存**

传统的缓存是单一的，只要提示词的任何部分发生变化，整个缓存就会失效。而 Claude 允许你将提示词的不同部分（如工具定义、系统指令、文档内容、对话历史）划分到不同的缓存层。

工作流程如下：
- **设置断点**：你在提示词的不同位置通过 `cache_control` 参数设置断点。

- **分层处理**：API 会从头开始处理你的提示词。每当遇到一个断点，它就会检查从上一个断点（或从头开始）到当前断点的内容是否与缓存中的记录匹配。

- **寻找最长匹配**：

  *   如果匹配，API 会从缓存中加载这部分的处理状态，然后继续处理下一部分。

  *   如果不匹配，API 会正常处理这部分内容，然后将其结果存入缓存，并使之后的所有缓存层失效。

这种设计的最大优势在于**隔离变化**。例如，如果你只更新了对话历史，那么工具定义、系统指令和 RAG 文档的缓存仍然有效，API 只需处理最新的用户输入，从而极大地节省了成本和时间。

以下是如何在实践中应用多缓存断点的示例：

```python
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=1024,
    tools=[
        # 工具定义 - 缓存点 1
        {"name": "search", "description": "...", },
        {"name": "get_doc", "description": "...",
         "cache_control": {"type": "ephemeral"}},  # 缓存所有工具
    ],
    system=[
        # 静态指令 - 缓存点 2
        {"type": "text", "text": "你是研究助手...",
         "cache_control": {"type": "ephemeral"}},
        # RAG 文档 - 缓存点 3
        {"type": "text", "text": "知识库内容...",
         "cache_control": {"type": "ephemeral"}},
    ],
    messages=[
        # 对话历史 - 缓存点 4
        {"role": "user", "content": [
            {"type": "text", "text": "最新问题",
             "cache_control": {"type": "ephemeral"}}
        ]}
    ]
)
```

这种设计的优势：
- **隔离性**：更新对话历史（变化最频繁）不会使工具定义或系统指令（相对静态）的缓存失效。
- **效率高**：API 只重新计算发生变化的部分以及之后的内容，而不是整个提示词。
- **灵活性**：可以为不同更新频率的内容块设置独立的缓存策略。

### 1.7 1 小时缓存的使用场景

何时选择 1 小时 TTL：
- Agent 任务执行时间超过 5 分钟
- 用户可能间隔 5 分钟以上才回复
- 延迟敏感的应用，需要保持缓存可用性
- 希望减少速率限制的影响（缓存命中不计入速率限制）

注意：5 分钟缓存会在每次使用时自动刷新，适合高频请求场景。

### 1.8 最佳实践

- **内容组织**：将静态内容放在前面，动态内容放在后面

- **策略性断点**：根据更新频率设置不同的缓存断点

- **监控性能**：使用 `cache_creation_input_tokens` 和 `cache_read_input_tokens` 追踪缓存效果

- **保持一致性**：确保缓存标记位置在多次调用间保持一致

---

## 二、OpenAI 的 Prompt Caching

### 2.1 核心特性

OpenAI 的 prompt caching 最大特点是**自动化和零配置**：

- **完全自动**：无需任何代码修改，系统自动启用
- **零额外成本**：缓存写入和读取都不收取额外费用
- **智能路由**：基于提示词前缀的哈希值进行请求路由
- **成本节省**：缓存命中可节省高达 75% 成本，减少 80% 延迟

### 2.2 工作原理

```
1. 缓存路由
   ├─ 计算提示词前缀（通常前 256 tokens）的哈希值
   ├─ 结合 prompt_cache_key 参数（可选）
   └─ 路由到对应的服务器

2. 缓存查找
   └─ 检查该服务器上是否有匹配的前缀

3. 缓存命中/未命中
   ├─ 命中：使用缓存结果，降低延迟和成本
   └─ 未命中：处理完整提示词，缓存前缀供后续使用
```

### 2.3 缓存要求

- **最小长度**：1024 tokens（对于 gpt-4o 及更新模型）
- **缓存粒度**：128 tokens 的增量（1024, 1152, 1280, ...）
- **缓存时长**：5-10 分钟不活动后自动清除，低峰期可持续长达 1 小时

### 2.4 响应示例

```json
{
  "usage": {
    "prompt_tokens": 2006,
    "completion_tokens": 300,
    "total_tokens": 2306,
    "prompt_tokens_details": {
      "cached_tokens": 1920  // 缓存命中的 token 数
    },
    "completion_tokens_details": {
      "reasoning_tokens": 0
    }
  }
}
```

### 2.5 prompt_cache_key 参数

这是 OpenAI 缓存机制的独特功能：

```python
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[...],
    prompt_cache_key="my-system-prompt-v1"  # 用于路由优化
)
```

**作用**：
- 与前缀哈希结合，影响请求路由
- 帮助系统识别具有相同长前缀的请求
- 提高缓存命中率

**注意**：同一 `prompt_cache_key` 的请求速率应保持在约 15 次/分钟以下，避免缓存溢出到其他机器。

### 2.6 可缓存内容

- **消息数组**：完整的 messages 数组（system、user、assistant）
- **图片**：用户消息中的图片（链接或 base64），确保 `detail` 参数一致
- **工具**：`tools` 数组和 messages 都可缓存
- **结构化输出**：结构化输出的 schema 会作为系统消息的前缀缓存

### 2.7 最佳实践

1. **结构化提示词**：
   - 前面：静态内容（指令、示例、工具定义）
   - 后面：动态内容（用户输入）

2. **一致使用 prompt_cache_key**：
   - 为共享相同前缀的请求使用相同的 key
   - 保持每个 prefix-key 组合的请求频率低于 15/分钟

3. **监控指标**：
   - 缓存命中率
   - 延迟改善
   - 缓存 token 比例

4. **保持请求稳定**：
   - 维持相同前缀的稳定请求流
   - 避免频繁更改提示词结构

---

## 三、Gemini (Google) 的 Prompt Caching

### 3.1 双重缓存机制

Gemini 独特地提供了两种缓存方式：

#### 隐式缓存（Implicit Caching）
- **自动启用**：Gemini 2.5 模型默认开启
- **零配置**：无需任何代码更改
- **自动节省**：自动传递成本节省
- **最小 token 数**：2.5 Flash 为 1,024 tokens，2.5 Pro 为 4,096 tokens

#### 显式缓存（Explicit Caching）
- **手动控制**：需要显式创建缓存对象
- **保证节省**：明确的成本节省承诺
- **更灵活**：支持大多数模型，精细控制 TTL
- **最小 token 数**：2.5 Flash 为 1,024 tokens，2.5 Pro 为 2,048 tokens

### 3.2 隐式缓存使用

隐式缓存已自动启用，你只需：

**提高缓存命中率的技巧**：
- 将大量通用内容放在提示词开头
- 在短时间内发送具有相似前缀的请求

**查看缓存效果**：
```python
response = model.generate_content(...)
print(response.usage_metadata)  # 查看缓存命中的 token 数
```

### 3.3 显式缓存实现

显式缓存需要先创建缓存对象，然后在生成内容时引用：

```go
// 1. 创建缓存
cache, err := client.Caches.Create(ctx, modelName, &genai.CreateCachedContentConfig{
    Contents: contents,  // 要缓存的内容
    SystemInstruction: genai.NewContentFromText(
        "你是分析专家...", genai.RoleUser,
    ),
    // TTL 默认 1 小时
})

// 2. 使用缓存生成内容
response, err := client.Models.GenerateContent(
    ctx,
    modelName,
    genai.Text("请总结这份文档"),
    &genai.GenerateContentConfig{
        CachedContent: cache.Name,  // 引用缓存
    },
)
```

### 3.4 缓存管理操作

#### 列出所有缓存
```go
caches, err := client.Caches.All(ctx)
for _, item := range caches {
    fmt.Println(item.Name)
}
```

#### 分页列出缓存
```go
page, err := client.Caches.List(ctx, &genai.ListCachedContentsConfig{
    PageSize: 2,
})

for {
    for _, item := range page.Items {
        fmt.Println(item.Name)
    }
    if page.NextPageToken == "" {
        break
    }
    page, err = page.Next(ctx)
}
```

#### 更新缓存 TTL
```go
cache, err = client.Caches.Update(ctx, cache.Name, 
    &genai.UpdateCachedContentConfig{
        TTL: 7200 * time.Second,  // 更新为 2 小时
    },
)
```

#### 删除缓存
```go
_, err = client.Caches.Delete(ctx, cache.Name, 
    &genai.DeleteCachedContentConfig{},
)
```

### 3.5 何时使用显式缓存

显式缓存适合以下场景：

- **系统指令丰富的聊天机器人**：长系统提示词被短请求反复引用
- **视频文件的重复分析**：相同视频需要多次处理
- **大型文档集的查询**：频繁查询同一批文档
- **代码库分析或 bug 修复**：代码仓库内容需要多次引用

### 3.6 与 OpenAI 库集成

如果使用 OpenAI 库，可通过 `extra_body` 启用显式缓存：

```python
response = client.chat.completions.create(
    model="gemini-2.0-flash",
    messages=[...],
    extra_body={
        "cached_content": cache_name
    }
)
```

---

## 四、各家 Prompt Caching 对比总结

### 4.1 功能对比表

| 特性 | Claude | OpenAI | Gemini |
|-----|--------|--------|--------|
| **配置方式** | 手动标记 | 全自动 | 隐式/显式双模式 |
| **最小 token** | 1024-2048 | 1024 | 1024-4096 |
| **缓存粒度** | 任意 | 128 增量 | 任意 |
| **TTL 选项** | 5分钟 / 1小时 | 5-10分钟 | 默认1小时（可调） |
| **缓存点数量** | 最多 4 个 | 单个前缀 | 单个对象 |
| **手动管理** | ✓ | ✗ | ✓（显式模式） |
| **额外费用** | ✓（写入成本） | ✗ | ✓（存储成本） |
| **成本节省** | **90%（命中）** | **最高 75%** | **提供优惠（未明确比例）** |
| **延迟改善** | 显著 | 最高 80% | 显著 |

### 4.2 使用场景推荐

#### Claude - 适合需要精细控制的场景
- 需要独立控制不同部分的缓存（工具、文档、对话）
- 长时间的 Agent 任务（使用 1 小时 TTL）
- 需要监控详细的缓存指标
- 复杂的多步骤工作流

#### OpenAI - 适合追求简单的场景
- 希望零配置、自动优化
- 不想增加代码复杂度
- 标准的聊天/对话应用
- 预算有限（无额外缓存费用）

#### Gemini - 适合灵活切换的场景
- 想要同时享受自动化和手动控制
- 需要长 TTL（1 小时+）
- 需要主动管理缓存对象
- 复杂的文档/视频分析任务

### 4.3 架构提示词结构设计

**通用原则**（适用于所有提供商）：
```
┌─────────────────────────┐
│ 系统提示词 + 指令        │ ← 最少变化，缓存价值最高
├─────────────────────────┤
│ 工具定义                │ ← 偶尔变化
├─────────────────────────┤
│ 文档 / RAG 上下文       │ ← 定期更新
├─────────────────────────┤
│ 对话历史                │ ← 频繁变化
├─────────────────────────┤
│ 当前用户输入            │ ← 每次都变化
└─────────────────────────┘
```

**Claude 特定策略**：
```python
# 为不同层级设置独立缓存点
tools=[...],              # 缓存点 1
system=[
    {...},                # 缓存点 2 - 指令
    {...},                # 缓存点 3 - 文档
],
messages=[
    {...},                # 缓存点 4 - 历史
    {"role": "user", "content": "新输入"}  # 不缓存
]
```

**OpenAI 特定策略**：
```python
# 使用 prompt_cache_key 优化路由
messages=[
    {"role": "system", "content": "静态指令..."},
    {"role": "user", "content": "大量上下文..."},
    # ... 更多静态内容
    {"role": "user", "content": "动态用户输入"}  # 放在最后
],
prompt_cache_key="my-app-v1"  # 帮助路由
```

**Gemini 特定策略**：
```python
# 显式缓存：将静态内容创建为缓存对象
cache = client.Caches.Create(
    contents=[system_prompt, large_documents],
    ttl=3600  # 1 小时
)

# 每次只传递动态内容
response = client.Models.GenerateContent(
    model_name,
    user_query,
    cached_content=cache.name
)
```

## 五、最佳实践总结

### 5.1 通用最佳实践

1. **结构化提示词**
   - 静态内容在前，动态内容在后
   - 按照变化频率组织内容
   - 保持缓存边界清晰

2. **监控和优化**
   - 跟踪缓存命中率
   - 分析成本节省
   - 测量延迟改善
   - 定期调整策略

3. **处理缓存失效**
   - 理解不同修改的影响
   - 计划内容更新策略
   - 考虑缓存预热

4. **成本效益分析**
   - 计算缓存写入 vs 读取成本
   - 评估请求频率
   - 确定最优 TTL

### 5.2 特定平台建议

**Claude 用户**：
- 利用多缓存点分离不同更新频率的内容
- 根据使用模式选择 5 分钟或 1 小时 TTL
- 监控 `cache_creation_input_tokens` 和 `cache_read_input_tokens`
- 注意 20 块内容的自动查找限制

**OpenAI 用户**：
- 信任自动机制，专注于提示词结构
- 合理使用 `prompt_cache_key` 优化路由
- 保持请求频率稳定
- 利用免费缓存节省成本

**Gemini 用户**：
- 简单场景使用隐式缓存
- 复杂场景使用显式缓存
- 主动管理缓存生命周期
- 利用 cache 对象的灵活性

### 5.3 常见陷阱

1. **过度缓存**
   - 缓存频繁变化的内容
   - 设置过长的 TTL 却不经常使用
   - 增加不必要的缓存点

2. **缓存结构不当**
   - 动态内容在前，静态内容在后
   - 缓存边界设置不合理
   - 忽略内容变化模式

3. **忽略失效规则**
   - 频繁修改缓存前缀
   - 不理解层级失效机制
   - 工具定义变化导致全缓存失效

4. **监控不足**
   - 不跟踪缓存命中率
   - 未验证成本节省
   - 忽略延迟改善数据

---

## 六、总结

Prompt caching 是降低 AI 应用成本和延迟的关键技术。不同提供商的实现各有特色：

- **Claude** 提供最精细的控制，适合复杂应用
- **OpenAI** 提供最简单的体验，适合快速开发
- **Gemini** 提供最灵活的选择，兼顾两者优势

选择合适的方案需要考虑：
- 应用的复杂度

- 开发团队的能力

- 成本预算

- 性能要求

无论选择哪个平台，理解缓存原理和最佳实践都能帮助你最大化收益。建议：
- 从简单开始，逐步优化
- 持续监控和调整，甚至加入到 AI 评测过程中的 token 消耗监控
- 根据实际数据做决策
- 保持提示词结构的灵活性

希望本文能帮助你深入理解 prompt caching，并在实际项目中有效应用！

**参考资料**

- [Claude Prompt Caching 文档](https://docs.anthropic.com/claude/docs/prompt-caching)
- [OpenAI Prompt Caching 指南](https://platform.openai.com/docs/guides/prompt-caching)
- [Gemini Prompt Caching 文档](https://ai.google.dev/gemini-api/docs/caching)
