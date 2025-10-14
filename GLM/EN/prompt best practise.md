# Prompt Engineering

https://docs.bigmodel.cn/cn/guide/platform/prompt

> Master the prompt techniques for the GLM language model and CogView image generation model to achieve better generation results.

<Info>
  This guide shares strategies for achieving better generation results with the GLM language model and CogView image generation model. You can combine prompt techniques for better effects.
</Info>

## Language Model Prompt Engineering

### Strategy 1: Write Clear and Specific Instructions

To get the best answers, users need to provide GLM with clear and specific instructions. The more clearly you can state your needs to GLM, the higher the quality of the provided answer.

**Technique: Define a System Prompt**

<Tip>
  A System Prompt is a tool used to set the behavior patterns of an AI assistant, including role-setting, language style, task mode, and specific behavioral guidance for particular problems.
</Tip>

```json
{
  "role": "system",
  "content": "You are good at extracting key information from text. You are precise, data-driven, and focus on highlighting key information. Extract key data and facts from the text snippets provided by the user and present the extracted information in a clear JSON format."
}
```

**Technique: Provide Specific Detail Requirements**

Add details and background information about the desired output content in the prompt.

<CodeGroup>
  ```text Example
  I am very interested in the planets of the solar system, especially Saturn. Please provide basic information about Saturn, including its size, composition, ring system, and any unique astronomical phenomena.
  ```
</CodeGroup>

**Technique: Have GLM Role-play**

Having GLM play a role allows it to more accurately imitate the behavior and conversation style of that role.

<CodeGroup>
  ```text Example
  As a quantum physicist, explain the basic principles of quantum physics and briefly introduce its applications in modern technology.
  ```
</CodeGroup>

**Technique: Use Delimiters to Mark Different Input Parts**

<CodeGroup>
  ```text Example
  Please summarize the core ideas and outline based on the following content:
  """ Article content to be summarized """
  ```
</CodeGroup>

**Technique: Chain-of-Thought Prompting**

<Note>
  This requires the model to solve a problem step-by-step and also to show each step of its reasoning process. This way, the likelihood of inaccurate results is reduced, and it is easier for the user to evaluate the model's response.
</Note>

<CodeGroup>
  ```text Example
  As an AI assistant, your task is to help users solve complex math problems. For each problem, you first need to solve it independently, then compare and evaluate the user's answer, and finally provide feedback. In this process, please show each step of your reasoning.

  I need help with a math problem:
  """The problem is: A farm has a total of 35 chickens and cows. There are 94 legs in total. How many chickens and cows are there? My answer is 23 chickens and 12 cows."""
  ```
</CodeGroup>

**Technique: Few-Shot Learning**

This can serve as an example for few-shot learning. These samples can be used to guide the model to imitate specific behaviors and language styles.

<CodeGroup>
  ```text Example
  Imitate this style:
  ''' 
  1. Three-cup chicken leaps in the pot, a bonfire of the years, a romantic symphony.
  2. The taste of stewed ribs is a winter quilt, an echo of the homeland.
  3. The aroma of braised fish is the ocean's whisper, a love letter from the sea.
  '''
  Generate new sentences.
  ```
</CodeGroup>

**Example of Specifying Output Length**

<Warning>
  You can specify the length of the output, but it is difficult for the model to generate a precise number of words.
</Warning>

<CodeGroup>
  ```text Example
  Please summarize this article in no more than 100 words.
  ```
</CodeGroup>

### Strategy 2: Provide Reference Materials

<Info>
  Citing external materials can effectively improve the accuracy of the model's answers. This practice is particularly suitable for document-based question-answering systems as it helps reduce the generation of false or fabricated information while ensuring the timeliness and accuracy of the answers.
</Info>

When the model is limited by context length and cannot reference very long texts, semantic snippets from documents can be retrieved using a Retrieval tool.

<CodeGroup>
  ```text Example
  As an AI assistant, your task is to help users find and understand a specific company's rules and regulations. In this scenario, you will use search results to answer user queries about the company's leave policy. Please provide accurate and detailed information based on the search results.
  """Specific search results"""
  ```
</CodeGroup>

### Strategy 3: Decompose Complex Tasks into Simple Subtasks

<Tip>
  When dealing with complex tasks, the error rate is usually higher. To improve efficiency and accuracy, the best practice is to reconstruct these complex tasks into a series of simple, coherent subtasks.
</Tip>

In this method, the completion of each subtask sequentially becomes the starting point for the next, forming an efficient workflow. This task flow simplification helps to improve the overall processing quality and reliability of the model, especially when facing complex problems that require the integration of large amounts of data and in-depth analysis.

**Technique: Intent Understanding and Entity Extraction**

The content output by the large model needs to be directly used by the back-end service interface, so the large model must output in a fixed format to facilitate interface parsing of the model's output and prevent errors.

<CodeGroup>
  ```text Example
  When you understand the user's intent to book a meeting room, extract the relevant entities and output them in JSON format.
  ```
</CodeGroup>

**Technique: Summarize Key Information from the Preceding Text**

In long conversations, to ensure coherence and effectiveness, refining and summarizing the previous communication content can maintain the focus of the conversation, reduce repetition and confusion, and speed up the model's processing speed.

**Technique: Summarize Long Documents in Sections and Gradually Build a Complete Summary**

<Note>
  Due to the limited context length for processing text, the model cannot summarize a text that exceeds a certain length all at once. For example, when summarizing a long book, we can adopt a step-by-step method, summarizing chapter by chapter.
</Note>

The summaries of each chapter can be combined and then further generalized to form a more refined overall summary. This process can be repeated until the entire book's content is fully summarized. If the understanding of subsequent chapters depends on the preceding content, necessary contextual information can be included in the summary.
