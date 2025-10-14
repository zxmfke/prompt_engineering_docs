# Prompt Best Practices

https://platform.moonshot.cn/docs/guide/prompt-best-practice#%E5%90%91%E6%A8%A1%E5%9E%8B%E6%8F%90%E4%BE%9B%E8%BE%93%E5%87%BA%E7%A4%BA%E4%BE%8B

> System Prompt Best Practices: A system prompt refers to the initial input or instruction that the model receives before generating text or a response. This prompt is crucial for the model's operation.

## Write Clear Instructions

- Why do you need to provide clear instructions to the model?

> The model cannot read your mind. If the output is too long, you can ask the model for a brief reply. If the output is too simple, you can ask the model for expert-level writing. If you don't like the output format, show the model the format you want to see. The less the model has to guess what you want, the more likely you are to get satisfactory results.

### Include more details in your request to get more relevant answers

> To get highly relevant output, make sure to provide all important details and background in your input request.

| General Request | Better Request |
| --- | --- |
| How to add numbers in Excel? | How do I sum a row of numbers in an Excel spreadsheet? I want to automatically sum each row of the entire sheet and put all the totals in the rightmost column named "Total". |
| Work report summary | Summarize the 2023 work record into a paragraph of no more than 500 words. List the work highlights for each month in a sequence and make a summary of the work for the whole year of 2023. |

### Ask the model to play a role in the request to get more accurate output

> Add a role for the model to use in its reply in the 'messages' field of the API request.

```json
{
  "messages": [
    {"role": "system", "content": "You are Kimi, an AI assistant from Moonshot AI. You are better at conversations in Chinese and English. You will provide users with safe, helpful, and accurate answers. At the same time, you will refuse to answer any questions involving terrorism, racial discrimination, pornography, and violence. Moonshot AI is a proper noun and cannot be translated into other languages."},
    {"role": "user", "content": "Hello, my name is Li Lei, what is 1+1?"}
  ]
}
```

### Use delimiters in your request to clearly indicate different parts of the input

> For example, using delimiters such as triple quotes/XML tags/section titles can help distinguish parts of the text that need to be handled differently.

```json
{
  "messages": [
    {"role": "system", "content": "You will receive two articles of the same category, separated by XML tags. First, summarize the argument of each article, then indicate which article presents a better argument and explain why."},
    {"role": "user", "content": "<article>Insert article here</article><article>Insert article here</article>"}
  ]
}
```

```json
{
  "messages": [
    {"role": "system", "content": "You will receive a paper's abstract and title. The title of the paper should give the reader a clear idea of the paper's topic and should also be eye-catching. If the title you receive does not meet these criteria, please propose 5 alternative options."},
    {"role": "user", "content": "Abstract: Insert abstract here.\n\nTitle: Insert title here"}
  ]
}
```

### Specify the steps required to complete the task

> It is recommended to specify a series of steps for the task. Clearly writing out these steps can make it easier for the model to follow and get a better output.

```json
{
  "messages": [
    {"role": "system", "content": "Use the following steps to respond to user input.\nStep 1: The user will provide text in triple quotes. Summarize this text in one sentence with the prefix 'Summary:'.\nStep 2: Translate the summary from the first step into English and add the prefix 'Translation:'."},
    {"role": "user", "content": "\"\"\"Insert text here\"\"\""}
  ]
}
```

### Provide the model with output examples

> Providing the model with example descriptions of general guidance is often more efficient for the model's output than showing all permutations of a task. For example, if you intend for the model to replicate a style that is difficult to describe clearly in response to user queries. This is called "few-shot" prompting.

```json
{
  "messages": [
    {"role": "system", "content": "Answer in a consistent style"},
    {"role": "user", "content": "Insert text here"}
  ]
}
```

### Specify the desired length of the model's output

> You can ask the model to generate an output of a specific target length. The target output length can be specified in terms of words, sentences, paragraphs, bullet points, etc. But please note that instructing the model to generate a specific number of words is not highly precise. The model is better at generating output with a specific number of paragraphs or bullet points.

```json
{
  "messages": [
    {"role": "user", "content": "Summarize the text in triple quotes in two sentences, within 50 words. \"\"\"Insert text here\"\"\""}
  ]
}
```

## Provide Reference Text

### Instruct the model to use reference text to answer questions

> If you can provide a model with credible information related to the current query, you can instruct the model to use the provided information to answer questions.

```json
{
  "messages": [
    {"role": "system", "content": "Use the provided articles (separated by triple quotes) to answer the questions. If the answer cannot be found in the articles, please write 'I can't find the answer.'"},
    {"role": "user", "content": "<Please insert articles, each separated by triple quotes>"}
  ]
}
```

## Split Complex Tasks

### Identify user query-related instructions through classification

> For tasks that require a large number of independent instruction sets to handle different situations, classifying the query type and using that classification to specify which instructions are needed can help the output.

```json
# Based on the classification of the customer's query, a more specific set of instructions can be provided to the model for it to handle subsequent steps. For example, suppose the customer needs help with "troubleshooting."
{
  "messages": [
    {"role": "system", "content": "You will receive user service inquiries that require technical support. You can help users in the following ways:\n\n- Ask them to check if *** is configured.\nIf all *** are configured but the problem persists, ask them for the model of the device they are using.\n- Now you need to tell them how to restart the device:\n= If the device model is A, please perform ***.\n- If the device model is B, suggest they perform ***."}
  ]
}
```

### For long-running conversational applications, summarize or filter previous conversations

> Since the model has a fixed context length display, conversations between the user and the model assistant cannot continue indefinitely.

A solution to this problem is to summarize the first few rounds of the conversation. Once the size of the input reaches a predetermined threshold, a query is triggered to summarize the previous part of the conversation, and the summary of the previous conversation can also be included as part of the system message. Alternatively, the previous conversation throughout the entire dialogue can be summarized asynchronously.

### Summarize long documents in chunks and recursively build a complete summary

> To summarize the content of a book, we can use a series of queries to summarize each chapter of the document. The partial summaries can be aggregated and summarized, producing a summary of summaries. This process can be repeated recursively until the entire book is summarized. If previous chapters are needed to understand subsequent parts, then a summary of the chapters preceding a given point can be included when summarizing the content at that point in the book.

