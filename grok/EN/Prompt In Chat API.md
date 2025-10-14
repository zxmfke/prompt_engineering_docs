## [Chat](https://docs.x.ai/docs/guides/chat#chat)

Text in, text out. Chat is the most popular feature on the xAI API, and can be used for anything from summarizing articles, generating creative writing, answering questions, providing customer support, to assisting with coding tasks.

------

## [Prerequisites](https://docs.x.ai/docs/guides/chat#prerequisites)

- xAI Account: You need an xAI account to access the API.
- API Key: Ensure that your API key has access to the chat endpoint and the chat model is enabled.

If you don't have these and are unsure of how to create one, follow [the Hitchhiker's Guide to Grok](https://docs.x.ai/docs/tutorial).

You can create an API key on the [xAI Console API Keys Page](https://console.x.ai/team/default/api-keys).

Set your API key in your environment:

```bash
export XAI_API_KEY="your_api_key"
```

------

## [A Basic Chat Completions Example](https://docs.x.ai/docs/guides/chat#a-basic-chat-completions-example)

You can also stream the response, which is covered in [Streaming Response](https://docs.x.ai/docs/guides/streaming-response).

The user sends a request to the xAI API endpoint. The API processes this and returns a complete response.

```python
import os

from xai_sdk import Client
from xai_sdk.chat import user, system

client = Client(
    api_key=os.getenv("XAI_API_KEY"),
    timeout=3600, # Override default timeout with longer timeout for reasoning models
)

chat = client.chat.create(model="grok-4")
chat.append(system("You are a PhD-level mathematician."))
chat.append(user("What is 2 + 2?"))

response = chat.sample()
print(response.content)
```

Response:

```
'2 + 2 equals 4.'
```

------

## [Conversations](https://docs.x.ai/docs/guides/chat#conversations)

The xAI API is stateless and does not process new request with the context of your previous request history.

However, you can provide previous chat generation prompts and results to a new chat generation request to let the model process your new request with the context in mind.

An example message:

```json
{
  "role": "system",
  "content": [{ "type": "text", "text": "You are a helpful and funny assistant."}]
}
{
  "role": "user",
  "content": [{ "type": "text", "text": "Why don't eggs tell jokes?" }]
},
{
  "role": "assistant",
  "content": [{ "type": "text", "text": "They'd crack up!" }]
},
{
  "role": "user",
  "content": [{"type": "text", "text": "Can you explain the joke?"}],
}
```

<u>By specifying roles, you can change how the the model ingests the content. The `system` role content should define, in an instructive tone, the way the model should respond to user request. The `user` role content is usually used for user requests or data sent to the model. The `assistant` role content is usually either in the model's response, or when sent within the prompt, indicates the model's response as part of conversation history.</u>

------

## [Message role order flexibility](https://docs.x.ai/docs/guides/chat#message-role-order-flexibility)

Unlike some models from other providers, one of the unique aspects of xAI API is its flexibility with message role ordering:

- No Order Limitation: You can mix `system`, `user`, or `assistant` roles in any order for your conversation context.

**Example 1 - Multiple System Messages:**

```json
[
  { "role": "system", "content": "..." },
  { "role": "system", "content": "..." },
  { "role": "user", "content": "..." },
  { "role": "user", "content": "..." }
]
```

**Example 2 - User Messages First:**

```json
[
  { "role": "user", "content": "..." },
  { "role": "user", "content": "..." },
  { "role": "system", "content": "..." }
]
```