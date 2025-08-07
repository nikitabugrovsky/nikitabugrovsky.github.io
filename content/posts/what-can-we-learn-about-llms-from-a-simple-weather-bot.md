+++
date = '2025-08-06T21:05:03+03:00'
draft = false
toc = true
title = 'What Can We Learn About LLMs From a Simple Weather Bot?'
+++

When we build applications with Large Language Models (LLMs), one of the most powerful capabilities we can unlock is function calling. This post explores how two different models, Google's Gemini and the open model Gemma, approach this task. While both rely on a shared philosophy—the Natural Language Interface (NLI)—their implementations reveal two distinct engineering experiences.

The idea for this weather bot was born from the official [Google Gemini documentation on function calling](https://ai.google.dev/gemini-api/docs/function-calling?example=weather), which makes it clear that while the LLM can *suggest* calling a function, it is our responsibility to implement **and** call that function.

Using a simple weather bot as our case study, we will explore these two NLI implementations and the practical implications they have on our workflow, from initial setup to long-term maintenance.

## The Goal: A Smart, Conversational Weather Bot

Let's explain what our bot is capable of. The bot is not only sends requests to remote weather API & gets responces. It is able to hold a conversation, remember things, although its "memory" is very short.

It's important to note that models served either from remote APIs (Gemini) or locally via Ollama (Gemma) are used for bot's reasoning. Those models do not have any tools enabled by default. The core logic, which applies to *both* Gemini and Gemma models, follows a two-step interaction flow:

1.  **Understand the Request:** The user asks for the weather. The LLM's first job is to understand the user's intent and determine that it needs to call the [get_current_weather](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/tools/weather_tool.py#L166-L171) function. It also needs to extract the necessary arguments, such as `location="New York"`.

2.  **Summarize the Result:** The raw data from the weather API is then fed back to the LLM. The model's second job is to take this structured data and generate a natural, human-readable sentence, such as, "The weather in New York is currently 75 degrees and sunny."

This two-step process establishes the baseline interaction pattern we want to achieve, regardless of which AI model is powering the bot. It's a simple yet powerful demonstration of how LLMs can act as a reasoning engine to interact with external tools.

The complete source code for the bot is available on [GitHub](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot).

## How LLM Models Think: A Shared NLI Approach with Different Implementations

Both models use a **Natural Language Interface (NLI)** for function calling. It means effectively that LLMs are given instructions in natural language and use their reasoning capabilities to decide when to call a tool. The key difference lies in how this NLI is implemented and what that means for us as engineers.

### Gemini: Native, Integrated NLI

With large models like Gemini, the NLI is a **natively integrated feature of the API**. As engineers, we describe our tools using a structured [JSON schema](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/genai_client.py#L20-L33) and provide it with our prompt. Gemini then uses its advanced, built-in reasoning to understand the user's intent, select the right tool, and generate a structured function call object. It's a clean, reliable interaction where the complexity is handled by the model.

### Gemma: Custom, Tailored NLI

Gemma, as an open model, also uses an NLI, but it requires us to **explicitly implement the pattern through prompt engineering**. Instead of a built-in API feature, we provide a detailed [system prompt](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/ollama_client.py#L27-L48) that explains the tools, their parameters, and the exact JSON format the model should output. The model isn't "calling a tool"; it's using the instructions we provided to **generate text that is essentially a JSON function call**. This places more responsibility on our prompt engineering skills to ensure the model's output is consistently in the correct format, which our application code must then parse.

## Bridging the Gap for Gemma Models

Usually the Gemma model approach requires more involvement from an engineer to avoid hallucinations. Hence we need to focus on the specific techniques we can use to make model responses reliable.

### The "Few-Shot Prompting" Strategy

This is the key strategy to make our NLI implementation robust. As seen in the [`ollama_client.py`](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/ollama_client.py), we provide examples in the system prompt to "teach" the model how to behave:

1.  **A simple greeting:** This teaches the model when *not* to call a tool.
2.  **A complete, multi-turn weather query:** This shows the model the full end-to-end flow: identifying the user's intent, formatting the JSON, summarizing the result, and even answering a follow-up question.

This highlights a fundamental difference in the engineering experience. With Gemini, we are focusing on the API. With Gemma models, we master the art of the prompt and introduce prompt parsing. In both cases, we rely on the model's reasoning capabilities to get an appropriate response.

## Managing Memory: A Universal Challenge with Unique Solutions

All conversational AI needs memory management to avoid high costs and manage context limits effectively.

### General Approach: The Sliding Window

A simple, effective solution for both models is the "sliding window," where only the most recent parts of the conversation are kept in memory. This is implemented in both clients to manage the conversation history.

### The Hybrid Approach for Gemma Model

The [`OllamaClient`](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/ollama_client.py) demonstrates a nuanced approach. The critical "few-shot" examples in the prompt must always be present, so the sliding window only applies to the *actual* user conversation. This reinforces the theme that models like Gemma often require more manual, bespoke engineering approach to achieve the same level of sophistication.

## Conclusion: The Right LLM for the Job

Our simple weather bot, despite its simplicity, has provided a clear lens through which to view the practical differences in modern LLM development. The choice is about which implementation of the Natural Language Interface (NLI) best suits our project's needs. Here are the key takeaways:

### 1. The Foundation is the Same: It’s All NLI
At their core, both Gemini and Gemma models use reasoning to interpret natural language instructions and decide when to call a function. Both models rely on the descriptive power of the context we provide. This shared foundation means we are always in the business of clearly defining our tools and goals.

### 2. Engineering Experience
The primary difference lies in *how* we provide that context:

*   **Gemini's Integrated NLI** offers an **API-centric** experience. Our main task is to create a structured [JSON schema](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/genai_client.py#L20-L33) that defines our tools. The model handles the complex reasoning internally, providing a clean, predictable function call object. This path prioritizes ease of use and reliability.
*   **Gemma's Explicit NLI** demands a **prompt-centric** experience. Our main task is to become prompt engineers, carefully crafting a [system prompt](https://github.com/nikitabugrovsky/multi-model-function-calling-chatbot/blob/main/clients/ollama_client.py#L27-L48) with instructions, examples, and the desired output format. This path gives us maximum control but requires more effort in both prompt design and parsing the model's output correctly. Model's context window gains extra importance in this case.

### 3. The Choice Depends on Our Project Goals
Ultimately, there is no single "best" approach. The decision hinges on the project's specific needs:

*   **Choose the Gemini (Integrated) approach** when speed of development and out-of-the-box reliability are key. It is ideal for prototypes, applications with rapidly changing tools, or teams that prefer to work with structured APIs over complex prompts.
*   **Choose the Gemma (Explicit) approach** when control, customization, and privacy (when deployed locally) are paramount. It is perfect for applications where we need to fine-tune the model's behavior, operate in a completely offline environment, or have strict requirements for data handling.
