## [聊天](https://docs.x.ai/docs/guides/chat#chat)

输入文本，输出文本。聊天是 xAI API 上最受欢迎的功能，可用于从总结文章、生成创意写作、回答问题、提供客户支持到协助编码任务等任何事情。

------

## [先决条件](https://docs.x.ai/docs/guides/chat#prerequisites)

- xAI 账户：您需要一个 xAI 账户才能访问 API。
- API 密钥：确保您的 API 密钥有权访问聊天端点并已启用聊天模型。

如果您没有这些并且不确定如何创建一个，请遵循 [Grok 漫游指南](https://docs.x.ai/docs/tutorial)。

您可以在 [xAI 控制台 API 密钥页面](https://console.x.ai/team/default/api-keys) 上创建一个 API 密钥。

在您的环境中设置您的 API 密钥：

```bash
export XAI_API_KEY="your_api_key"
```

------

## [一个基本的聊天补全示例](https://docs.x.ai/docs/guides/chat#a-basic-chat-completions-example)

您还可以流式传输响应，这在[流式响应](https://docs.x.ai/docs/guides/streaming-response)中有介绍。

用户向 xAI API 端点发送请求。API 处理此请求并返回一个完整的响应。

```python
import os

from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    timeout=3600, # 使用更长的超时时间覆盖默认超时时间以用于推理模型
)

chat = client.chat.create(model="grok-4")
chat.append(system("你是一位博士级别的数学家。"))
chat.append(user("2 + 2 是多少？"))

response = chat.sample()
print(response.content)
```

响应：

```
'2 + 2 等于 4。'
```

------

## [对话](https://docs.x.ai/docs/guides/chat#conversations)

xAI API 是无状态的，不会在您之前的请求历史记录的上下文中处理新请求。

但是，您可以将之前的聊天生成提示和结果提供给新的聊天生成请求，以让模型在考虑上下文的情况下处理您的新请求。

一个示例消息：

```json
{
  "role": "system",
  "content": [{ "type": "text", "text": "你是一个乐于助人且有趣的助手。"}]
}
{
  "role": "user",
  "content": [{ "type": "text", "text": "为什么鸡蛋不说笑话？" }]
},
{
  "role": "assistant",
  "content": [{ "type": "text", "text": "因为它们会笑裂！" }]
},
{
  "role": "user",
  "content": [{"type": "text", "text": "你能解释一下这个笑话吗？"}],
}
```

<u>通过指定角色，您可以更改模型接收内容的方式。`system` 角色内容应以指导性语气定义模型响应用户请求的方式。`user` 角色内容通常用于用户请求或发送给模型的数据。`assistant` 角色内容通常是模型的响应，或者在提示中发送时，表示模型作为对话历史的一部分的响应。</u>

------

## [消息角色顺序的灵活性](https://docs.x.ai/docs/guides/chat#message-role-order-flexibility)

与其他提供商的某些模型不同，xAI API 的一个独特之处在于其消息角色排序的灵活性：

- 无顺序限制：您可以按任何顺序混合 `system`、`user` 或 `assistant` 角色作为您的对话上下文。

**示例 1 - 多条系统消息：**

```json
[
  { "role": "system", "content": "..." },
  { "role": "system", "content": "..." },
  { "role": "user", "content": "..." },
  { "role": "user", "content": "..." }
]
```

**示例 2 - 用户消息在前：**

```json
[
  { "role": "user", "content": "..." },
  { "role": "user", "content": "..." },
  { "role": "system", "content": "..." }
]
```
