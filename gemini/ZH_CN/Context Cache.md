## 上下文缓存

https://ai.google.dev/gemini-api/docs/caching?lang=go

在典型的人工智能工作流中，您可能会一遍又一遍地将相同的输入令牌传递给模型。Gemini API 提供两种不同的缓存机制：

- 隐式缓存（在 Gemini 1.5 模型上自动启用，不保证节省成本）
- 显式缓存（可以在大多数模型上手动启用，保证节省成本）

在您希望保证节省成本的情况下，显式缓存非常有用，但这需要一些额外的开发工作。

## 隐式缓存

所有 Gemini 1.5 模型默认启用隐式缓存。如果您的请求命中缓存，我们会自动传递成本节省。您无需执行任何操作来启用此功能。它自 2024 年 5 月 8 日起生效。上下文缓存的最小输入令牌数为 1.5 Flash 的 1,024 和 1.5 Pro 的 4,096。

要增加隐式缓存命中的机会：

- 尝试将大而常见的内容放在提示的开头
- 尝试在短时间内发送具有相似前缀的请求

您可以在响应对象的 `usage_metadata` 字段中查看缓存命中的令牌数。

## 显式缓存

使用 Gemini API 显式缓存功能，您可以一次将一些内容传递给模型，缓存输入令牌，然后在后续请求中引用缓存的令牌。在某些量级下，使用缓存令牌的成本低于重复传入相同的令牌语料库。

当您缓存一组令牌时，您可以选择在令牌自动删除之前希望缓存存在多长时间。此缓存持续时间称为*生存时间* (TTL)。如果未设置，TTL 默认为 1 小时。缓存成本取决于输入令牌大小以及您希望令牌持续多长时间。

本节假定您已安装 Gemini SDK（或已安装 curl）并且已配置 API 密钥，如[快速入门](https://ai.google.dev/gemini-api/docs/quickstart)所示。

### 使用缓存生成内容

以下示例展示了如何使用缓存生成内容。

```go
package main

import (
    "context"
    "fmt"
    "log"

    "google.golang.org/genai"
)

func main() {
    ctx := context.Background()
    client, err := genai.NewClient(ctx, &genai.ClientConfig{
        APIKey: "GOOGLE_API_KEY",
        Backend: genai.BackendGeminiAPI,
    })
    if err != nil {
        log.Fatal(err)
    }

    modelName := "gemini-1.5-flash"
    document, err := client.Files.UploadFromPath(
        ctx,
        "media/a11.txt",
        &genai.UploadFileConfig{
          MIMEType: "text/plain",
        },
    )
    if err != nil {
        log.Fatal(err)
    }
    parts := []*genai.Part{
        genai.NewPartFromURI(document.URI, document.MIMEType),
    }
    contents := []*genai.Content{
        genai.NewContentFromParts(parts, genai.RoleUser),
    }
    cache, err := client.Caches.Create(ctx, modelName, &genai.CreateCachedContentConfig{
        Contents: contents,
        SystemInstruction: genai.NewContentFromText(
          "你是一位分析转录稿的专家。", genai.RoleUser,
        ),
    })
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println("缓存已创建：")
    fmt.Println(cache)

    // 使用缓存生成内容。
    response, err := client.Models.GenerateContent(
        ctx,
        modelName,
        genai.Text("请总结这份转录稿"),
        &genai.GenerateContentConfig{
          CachedContent: cache.Name,
        },
    )
    if err != nil {
        log.Fatal(err)
    }
    printResponse(response) // 用于打印响应部分的辅助函数
}
```

### 列出缓存

无法检索或查看缓存的内容，但可以检索缓存元数据。以下示例列出了所有缓存。

```go
caches, err := client.Caches.All(ctx)
if err != nil {
    log.Fatal(err)
}
fmt.Println("列出所有缓存：")
for _, item := range caches {
    fmt.Println("   ", item.Name)
}
```

以下示例使用 2 的页面大小列出缓存。

```go
page, err := client.Caches.List(ctx, &genai.ListCachedContentsConfig{PageSize: 2})
if err != nil {
    log.Fatal(err)
}

pageIndex := 1
for {
    fmt.Printf("列出缓存（第 %d 页）：\n", pageIndex)
    for _, item := range page.Items {
        fmt.Println("   ", item.Name)
    }
    if page.NextPageToken == "" {
        break
    }
    page, err = page.Next(ctx)
    if err == genai.ErrPageDone {
        break
    } else if err != nil {
        return err
    }
    pageIndex++
}
```

### 更新缓存

您可以为缓存设置新的 `TTL` 或 `ExpireTime`。不支持更改缓存的任何其他内容。

以下示例展示了如何更新缓存的 `TTL`。

```go
// 更新 TTL（2 小时）。
cache, err = client.Caches.Update(ctx, cache.Name, &genai.UpdateCachedContentConfig{
    TTL: 7200 * time.Second,
})
if err != nil {
    log.Fatal(err)
}
fmt.Println("更新后：")
fmt.Println(cache)
```

### 删除缓存

缓存服务提供了一个删除操作，用于手动从缓存中删除内容。以下示例展示了如何删除缓存。

```go
_, err = client.Caches.Delete(ctx, cache.Name, &genai.DeleteCachedContentConfig{})
if err != nil {
    log.Fatal(err)
}
fmt.Println("缓存已删除：", cache.Name)
```

### 使用 OpenAI 库进行显式缓存

如果您正在使用 [OpenAI 库](https://ai.google.dev/gemini-api/docs/openai)，您可以使用 [`extra_body`](https://ai.google.dev/gemini-api/docs/openai#extra-body) 上的 `cached_content` 属性启用显式缓存。

## 何时使用显式缓存

上下文缓存在大量初始上下文被较短请求重复引用的场景中特别适用。考虑在以下用例中使用上下文缓存：

- 具有大量[系统指令](https://ai.google.dev/gemini-api/docs/system-instructions)的聊天机器人
- 对冗长视频文件的重复分析
- 对大型文档集的重复查询
- 频繁的代码库分析或错误修复

### 显式缓存如何降低成本

上下文缓存是一项付费功能，旨在降低总体运营成本。计费基于以下因素：

1. **缓存令牌数：** 缓存的输入令牌数，在后续提示中包含时以较低费率计费。
2. **存储持续时间：** 缓存令牌的存储时间 (TTL)，根据缓存令牌数的 TTL 持续时间计费。TTL 没有最小或最大限制。
3. **其他因素：** 其他费用适用，例如未缓存的输入令牌和输出令牌。

有关最新的定价详情，请参阅 Gemini API [定价页面](https://ai.google.dev/pricing)。要了解如何计算令牌，请参阅[令牌指南](https://ai.google.dev/gemini-api/docs/tokens)。

### 其他注意事项

使用上下文缓存时请牢记以下注意事项：

- 上下文缓存的*最小*输入令牌数为 1.5 Flash 的 1,024 和 1.5 Pro 的 2,048。*最大*值与给定模型的最大值相同。（有关计算令牌的更多信息，请参阅[令牌指南](https://ai.google.dev/gemini-api/docs/tokens)）。
- 模型不对缓存令牌和常规输入令牌做任何区分。缓存内容是提示的前缀。
- 上下文缓存没有特殊的速率或使用限制；`GenerateContent` 的标准速率限制适用，并且令牌限制包括缓存的令牌。
- 缓存的令牌数在缓存服务的创建、获取和列出操作的 `usage_metadata` 中返回，并且在使用缓存时也在 `GenerateContent` 中返回。
