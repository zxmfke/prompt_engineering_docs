# 想了下还是得学 Prompt Engineering

prompt engineering 提示词工程，一个23年的概念，但是现在才去系统性看文档学习。原本打算是不学习 prompt engineering 直接去学 context engineering，我目前要写 prompt，只有2种情况：

- 我自己明确输入和预计输出，自己能够写清楚 prompt
- 我只有一个想法，以及可能的输出，我让 AI 帮我写

**ai 能够帮我写出很好的 prompt 为什么想了下又要学呢？**

因为需要回归到**开发**本身，可能在用 chat 跟 LLM 沟通的时候，并不会太在意格式，很多都是“一锤子买卖”。不过回到开发，真的需要调试效果，很多细节参数还是值得注意的。

另外有一次，碰到有个人问 “system prompt” 跟 “user prompt”，有什么区别，效果一样呀？给我问住了。。。

engineering 的意思是工程，工程在我看来就是一个严谨，经得起推敲的可落地架构。再加上为了能更好地学习 context engineering，prompt engineering 应该作为基础还是有必要学习的。

我希望我每次的 prompt 都是 effective 的。

**为什么要用 claude 的官方文档来学习呢？**

在翻看了国内外的厂商的文档，claude 的专业性确实毋庸置疑，而且事无巨细(抛开反华)。openai 和 gemini 也是非常不错的学习资料，作为补充。国内的很多都只是给你一些例子，但是要学习起来不足以构建对 prompt engineering 的体系。我把几个厂家关于 prompt engineering 的文档中英文放在，有兴趣的可以去学习，或者直接到官网。

prompt engineering 的文章多如牛毛，但我感觉只专注在 top 大模型玩家的文章即可。

---

## 一张图看懂学习路径（四层）

本文主要专注在 层级 2 和 层级 3，其他的带过，层级 4 里面的 context editing 和 prompt caching 会放在之后的文章。

```text
层级 1: 工具层（快速启动）
  ├─ 0. Overview（路线图）
  ├─ 1. Prompt Generator（AI 生成初稿）
  ├─ 2. Prompt Templates（变量管理）
  └─ 3. Prompt Improver（AI 优化迭代）

层级 2: 核心技巧层（普适原则）
  ├─ 4. Be Clear and Direct（清晰指令）
  ├─ 5. Use Examples（few-shot）
  ├─ 6. Let Claude Think（思维链）
  ├─ 7. Use XML Tags（结构化）
  ├─ 8. System Prompts（角色设定）
  └─ 9. Prefill Response（输出控制）

层级 3: 高级策略层（复杂场景）
  ├─ 10. Chain Prompts（任务分解）
  ├─ 11. Long Context Tips（长文档处理）
  └─ 12. Extended Thinking（深度推理）

层级 4: 产品特性层（平台能力）
  ├─ Claude 4 Best Practices（模型专项）
  ├─ Context Editing（上下文管理）
  └─ Prompt Caching（成本优化）
```

先让你“用起来”，再让你“理解为什么有效”，最后“迁移到复杂工作流”。

---

## 工具层：快速从 0 到 1

### 1) Prompt Generator：生成第一版

示例（目标：把研发周报转换为项目简报）：

```text
任务：将研发周报总结为 5 个要点，对管理层可读。
输入格式：
- 原始周报文本（必填）
- 目标受众：管理层
- 输出长度：约 300 字
约束：
- 用项目管理语言，不写实现细节
- 用 1-5 的编号列出要点
```

prompt 生成或者理解为完善，如果你描述的不清楚，可以通过 prompt generator 来优化你的初初级提示词。很多平台都有，但是要注意2点：

- 提示词要出效果，前提是你对需求或者输入，输出有了清晰的定义。开始可以不清楚，跟大模型先沟通到清楚。
- 不同平台都有优化提示词，不过最好的方式，还是找到一个“元提示词”来优化你的提示词。平台优化提示词的提示词，我想也只是简单的优化。这类型的提示词可以参考向阳乔木老师的元提示词，或者把本篇的内容整理成你的优化提示词的提示词。

### 2) Prompt Templates：变量化与复用

```text
系统角色：你是{role}，擅长{domain}。
目标：将{input_type}转写为{output_type}。
受众：{audience}
输出风格：{style}
长度：约{length}
结构化格式：见 <schema/>

<schema>
<summary></summary>
<bullets></bullets>
<risks></risks>
</schema>
```

不要小看模板的威力，在提示词迭代，生产上测试的时候，尤其运用到 AI 评测里面时。我在评测跑批的时候，没有注意要加入变量，导致跑不起来。但其实这个是最基础的，自己写 code evaluator 用来替换变量来进行评测也是会更加得心应手。

### 3) Prompt Improver：AI 辅助迭代

把你的模板和一段样例输入交给 Claude，请它：

- 明确模糊词（如“简洁”“深入”）
- 添加思维链指令与检查清单
- 用 XML/JSON 定义输出结构

改进请求示例：

```text
请在不改变目标的前提下：
1) 补充不歧义的验收标准；
2) 增加思维链提示但隐藏推理；
3) 给出严格的 XML 输出 schema；
4) 标注必要的输入变量与默认值。
```

claude 的 prompt improver 正是把你的初步整理的提示词，用下面的规则去优化你的提示词，让你的提示词更加精确。

---

## 核心技巧：六个最值好用法

下面每条都给“差/好”对照与可复制片段。

### 1) Be Clear and Direct（清晰指令）

不好：

```text
帮我写个 PR 描述。
```

更好：

```text
为下面 diff 生成 PR 描述：
要求：
- 受众：代码审阅者
- 结构：背景/变更/风险/回滚
- 约束：不超过 200 字，禁止营销语
输入：<diff>...</diff>
输出：严格使用以下 XML：
<pr>
  <background></background>
  <changes></changes>
  <risks></risks>
  <rollback></rollback>
</pr>
```

清晰的说明，其实很多人都说不清楚，尤其在遇到比较复杂的需求，我的方式是：

- 说给大模型听，看它能不能先拆解出正确的步骤，或者有什么额外的问题，知道它没有更多意见为止
- 写不清楚，就全部写下来，把看到的，想到的，关联到的先写下来，然后让大模型引导我简化
- 还有一招就是说给同事听，如果同事听得懂，知道怎么做，那大模型基本上也可以知道怎么做

不要小看说清楚的能力以及影响，跟我上一篇提到的能力一样，不要让大模型猜你想要什么

### 2) Use Examples（示例驱动 few-shot）

模式化表达任务时，用 1-2 个反例 + 1-2 个正例最稳。

```text
[反例]
输入：用户问“怎么提效？”
输出：泛泛而谈，无指标。

[正例]
输入：SaaS 客单价下滑 12%，目标 3 个月内恢复。
输出：
- 设定北极星指标：月度 ARPU
- 提三项对策：分层定价/年度预付/增购提示
- 每项附 1 个验证实验
```

提供例子，也是说清楚的一部分，核心也是不要让大模型猜，提供例子会事半功倍。

### 3) Let Claude Think（让模型思考但隐藏推理）

```text
在产生最终答案前，请先在内部完成思考与拆解，不要在输出中展示推理过程。
用“检查清单”自检：
- 是否覆盖所有输入变量？
- 是否给出可验证的指标或标准？
- 是否提供反例或风险？
```

### 4) Use XML Tags（结构化输出）

```text
输出必须是有效 XML：
<plan>
  <goal></goal>
  <steps>
    <step id="1"></step>
    <step id="2"></step>
  </steps>
  <metrics></metrics>
</plan>
如果无法满足，请返回：<error>原因</error>
```

结构化输出，看情况用 xml，更多时候用的是 json。不过要注意的是，输出json的时候，不要带上 \```json ```，很多时候在 markdown 上写好了，贴到代码里面，会让输出也带上 markdown 的格式。

### 5) System Prompts（角色与行为规范）

```text
系统：你是资深 SRE。优先级：稳定性>成本>新功能。严格遵循 SLO 与变更窗口。
```

system prompt 见下面章节描述。

### 6) Prefill Response（引导模型开头）

```text
输出以如下模板开头：
"背景："\n"变更："\n"回滚："
```

这也算说清楚一部分，因为很多人只说了要什么，但是没说什么格式输出，最后那着输出再改成自己想要的格式。这个一两次还好，多次了之后其实效率是很低下的。

---

## 高级策略：把复杂问题拆开、做深、做大

这部分技能没有太多实践，更多是理解。

### 1) Chain Prompts（水平拆分）

三步自我校正工作流：

```text
Step1-生成：根据输入草拟方案与假设。
Step2-评审：对 Step1 进行逐条审查并打分（1-5）。
Step3-改进：针对低分项给出修订并产出 V2。
```

这个方法特别适合处理复杂任务，它的核心思想是“分而治之”。不要指望一个超级复杂的 Prompt 能够一步到位解决所有问题，这往往会导致模型输出不稳定或偏离主题。

通过将任务拆分成多个更小、更明确的步骤，每个步骤都由一个独立的 Prompt 来处理，可以大大提高最终结果的质量和可控性。这就像一条流水线，上一个环节的输出是下一个环节的输入，中间还可以加入人工审核或模型的自我校验环节。这种“工作流”的设计思维，比单纯“堆砌” Prompt 的技巧更重要，是实现可靠、高质量自动化任务的关键。

### 2) Extended Thinking（垂直做深）

```text
当不确定性高或需要严谨推理时，先列出假设树与证据，再给结论。
隐藏推理，仅输出决策与关键证据摘要。
```

这与“核心技巧”里提到的 `Let Claude Think` 异曲同工，但更加强调在最终输出前进行深入、结构化的思考。你可以明确要求模型“一步一步思考”，并将思考过程放入像 `<thinking>` 或 `<scratchpad>` 这样的标签里。

这样做的好处是，模型被迫放慢速度，进行逻辑推演，而不是凭“直觉”快速给出答案。这能显著减少事实错误和逻辑漏洞。在最终的应用中，你可以选择只向用户展示最终结论，而将模型的思考过程作为调试或验证的依据。值得注意的是，有时候给出一个宏观的指令（如“请深入思考这个问题”）比规定好每一步要怎么做，效果反而更好，因为它能给予模型更大的创造性空间。对于需要高准确度的任务，比如代码生成、数据分析或法律咨询，这种方法几乎是必需的。

### 3) Long Context（规模做大）

为 100K+ token 设计输入结构：

```xml
<doc>
  <meta>
    <index>章节索引与位置</index>
    <task>任务描述</task>
  </meta>
  <chunks>
    <chunk id="1" src="/path/1" purpose="必读"/>
    <chunk id="2" src="/path/2" purpose="可选"/>
  </chunks>
</doc>
```

随着模型上下文窗口越来越大，如何高效地利用长上下文成了新的挑战。简单地把几百页的文档直接丢给模型，效果往往不佳，因为模型可能会在信息的海洋中“迷失”。

这里的关键是“结构化输入”。通过使用 XML 标签、Markdown 标题或 JSON 对象，将长文档整理成有清晰层级和元数据的格式，等于给了模型一张“地图”。你可以定义文档的索引、摘要、各个章节（chunks）以及它们的用途（比如“必读”、“可选”）。在提问时，可以直接引用这些结构化信息（例如，“根据 chunk id='2' 的内容回答...”），从而引导模型精确地定位和利用相关信息，避免在无关内容上浪费注意力。

除此以外，还有两个非常实用的技巧：
1.  **把长文档放在最前面**：将长篇的背景资料（如PDF、报告）置于整个 Prompt 的最顶端，在具体指令和问题之上，这能显著提高模型召回信息的准确率。
2.  **先引用再回答**：要求模型在给出最终答案前，先从提供的文档中引用所有相关的原文片段。这会强制模型先进行信息检索和定位，再进行归纳总结，能有效提升回答的“忠实度”。

## System Prompt

System Prompt，也常被称为“系统指令”，是一种特殊的指令，用于在对话或任务开始之前，为 AI 模型设定一个高级别的上下文、角色、个性和行为准则。

可以把它理解为给 AI 的一个“出厂设置”或“行为大纲”。它不像用户输入的具体问题（User Prompt）那样只在当前回合有效，而是作为一种持久的背景指令，影响着模型在整个对话过程中的所有回应。例如，你可以在 System Prompt 中定义 AI 是“一个资深的软件架构师”，并要求它“所有的回答都必须是严谨、客观且结构化的”。

为什么必须重视 System Prompt：

- 它定义“硬约束”和长期行为边界，优先级通常高于 user 指令。
- 它能显著降低指令注入/越权风险，将不可违背的政策集中到一个位置。
- 它让输出风格与结构长期稳定，利于评测与自动化。

**Claude（Anthropic）**

- 写法/位置：使用 system 消息，建议将角色、约束、风格、结构化 schema 放在 system。
- 注意事项：Claude 对结构化标签（XML/JSON）解析能力强；硬约束应在 system，task 变量放 user。
- 示例：

```text
[system]
你是资深技术写作者。优先级：准确性>可读性>简洁。
输出为有效 XML（<article/>），字段：title, summary, bullets。

[user]
请基于以下草稿生成文章：<draft>...</draft>
```

**OpenAI（Chat Completions / Responses）**

- 写法/位置：对话顶部的 system 消息；不可违背的规则放 system；任务变量放 user。
- 注意事项：结构化输出可结合 response_format/json_schema 或工具调用校验；可用 seed 降低方差。
- 示例：

```text
[system]
你是严格的技术编辑，只输出 JSON：{"title":string,"summary":string,"risks":string[]}。

[user]
<doc>...</doc>
```

**Gemini（Google）**

- 写法/位置：通过 systemInstruction 设置长期角色/约束；具体任务放 contents(user)。
- 注意事项：可用 response_mime_type="application/json" 强制 JSON；函数调用使用 functionDeclarations。
- 示例（伪格式）：

```text
systemInstruction: 你是架构评审助手，优先级：稳定性>SLO>成本。
user: 请评审设计文档并给出风险与回滚方案。
```

实操要点：

- 将“不可违背政策/风格/输出结构”放 system；“当次任务变量/输入”放 user。
- system 提醒安全与越权边界；user 聚焦上下文与数据。

### 消息角色

很多人会把“system prompt”当作普通 user 指令，这会带来优先级与安全边界的混淆。

**消息角色**

- system / developer：长期、不变、不可违背的“开发者/系统”级规则；用于角色、政策、输出结构、工具使用规范。
- user：本次任务的上下文与变量输入；可变、场次相关。
- assistant：模型的历史输出（可用于 few-shot 示例或预填）。
- tool/function（若支持）：描述可调用的工具与函数签名；输出为调用意图与参数。

**差异**

- OpenAI
  - Chat Completions：roles=system/user/assistant/tool；将长期规则放 system。
  - Responses API：支持 developer（≈更强的系统级指令），推荐把“不可违背规则”放 developer，把具体任务放 user；还可配合 response_format/json_schema 强化结构化输出。
- Anthropic Claude
  - 使用 system 指定长期规则；messages 中包含 user/assistant；工具经由 tool_use/tool_result。
- Google Gemini
  - 使用 systemInstruction 承载长期规则；用户内容在 contents(user)；可通过 response_mime_type 强制 JSON，函数通过 functionDeclarations。

**简易示例对照**

```text
[OpenAI · Chat Completions]
system: 你是代码审阅助手，只输出符合 <schema/> 的 XML
user: <diff>...</diff>

[OpenAI · Responses]
developer: 你是代码审阅助手，只输出 JSON，字段：title, risks[]
user: 这是本次 diff: ...

[Claude]
system: 你是代码审阅助手，遵循 <schema/>
user: <diff>...</diff>

[Gemini]
systemInstruction: 你是代码审阅助手，只输出 JSON
user: <diff>...</diff>
```

**实践建议**

- 不要把“系统级规则”写进 user；否则容易被后续 user 指令覆盖，或受注入影响。
- 在 OpenAI Responses 场景优先使用 developer 代替在 user 里写死规则；在 Chat Completions/Claude 用 system；在 Gemini 用 systemInstruction。

---

## 总结

Prompt Engineering 的精髓在于“**工程化**”：把技巧转化为模板、组件，并嵌入 Agent 工作流。通过迭代优化和实际应用，你的输出将越来越可靠、高效。记住，好的 Prompt 不是猜出来的，而是设计出来的——从简单开始，逐步复杂化，提示词的效果终会事半功倍。

最后所有技巧的目的其实是为了 **effective**，这个单词可以在 claude 的很多博客文档里面看到，技巧五花八门，最终目的得是有效。多实践总结自己的提示词技巧，说不定会比 claude 的更好。
