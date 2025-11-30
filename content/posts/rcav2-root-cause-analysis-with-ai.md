+++
date = '2025-11-30T16:52:10+02:00'
draft = false
toc = true
title = 'Rcav2: Root Cause Analysis With AI'
+++

[RCAccelerator (RCA)](https://github.com/orgs/RCAccelerator/repositories) is a tool developed by my team at Red Hat. The goal of the application is to help the openstack teams to perform root cause analysis on the [Zuul CI](https://zuul-ci.org) jobs. This article discusses the architecture of RCAv2, its key features, and the ways it leverages LLMs to diagnose the Zuul CI build failures.

## Evolution of the RCAccelerator application

During the last several months my team was working on improving existing RCAv1 application leveraging access to the [Gemini API](https://ai.google.dev/).

While RCAv1 was based on a Retrieval-Augmented Generation (RAG) pattern, [scraping data](https://github.com/RCAccelerator/tools) from various sources and storing it in a vector database for the LLM to reference, RCAv2 adopts a more dynamic and multi-step approach calling the Gemini API.

RCAv2 represents a significant architectural shift from its predecessor RCAv1.

* [RCAv1](https://github.com/RCAccelerator/chatbot) was an AI chatbot which provided UI and API access with a local inference server.
* [RCAv2](https://github.com/RCAccelerator/rca-api) is a [FastAPI](https://fastapi.tiangolo.com/)-based application with an event-driven architecture, designed to be integrated as a module into existing internal Zuul CI dashboards.

RCAv1 had a standalone chat interface. RCAv2 provides a [React component](https://github.com/RCAccelerator/rca-api/blob/main/frontend/RcaComponent.jsx) that is embedded into the existing Zuul CI dashboard, offering a more straightforward UX. If an engineer prefers a command line option, RCAv2 can be run in the CLI mode outputting results into stdout.

RCAv1 used an inference server ([vLLM](https://docs.vllm.ai/)) deployed locally. RCAv2 leverages the "LLM as a service" model, consuming LLMs via the [Gemini API](https://ai.google.dev/), specifically using the **gemini-2.5-pro** model.

RCAv1's workflow was centered around querying a vector database ([Qdrant](https://qdrant.tech/)). RCAv2 implements a multi-step workflow calling LLM, tools like Jirahttps://www.atlassian.com/software/jira and Slack, [LogJuicer](https://logjuicer.github.io/), Zuul and gathering context in real-time.

RCAv1 was mostly reliant on the [OpenAI Python API](https://github.com/openai/openai-python) calls. The unique feature of the RCAv2 application is an implementation of [DSPy Framework](https://github.com/stanfordnlp/dspy) as an interface for the interaction with the LLM.

DSPy provides a programmatic way to build and optimize LLM-based workflows, which is relevant to the multi-step RCA process. Moreover, DSPy provides different modules that offer various modes of communication (like [ReAct](https://dspy.ai/api/modules/ReAct/) and [Predict](https://dspy.ai/api/modules/Predict/)) with the LLM. This feature allows the team to experiment on implementing different approaches and test ideas to get better RCA results.

Last, RCAv2 application is built with observability in mind. When there are multiple implementations of the RCA workflow, we have to be able to test & compare RCA results. RCAv2 is integrated with [Opik](https://www.comet.com/site/products/opik/), a tool for tracing and observing LLM calls. This allows my team members to track, analyze, and debug prompts and responses, which is crucial for improving the accuracy and reliability of the RCA as well as compare different RCA workflow implementations.

## RCAv2 Report Implementation

Let’s talk in detail about how the RCA report is built in the RCAv2 application. RCA report implementation is a multi-step workflow that involves several components. Here's a step by step workflow overview:

1.  The workflow is initiated via an API call, typically triggered by a user clicking a button on an internal Zuul CI dashboard. The API call includes the URL of the failed build. Alternatively the [cli command](https://github.com/RCAccelerator/rca-api/blob/main/CONTRIBUTING.md#system-wide-cli-installation) can be run with a Zuul CI build URL as an argument.
2.  The system fetches and processes the build logs using [LogJuicer](https://logjuicer.github.io/), a tool that extracts structured error information from raw log files.
3.  RCAv2 analyzes the Zuul job definition to understand its purpose and actions. This is done by an "**[AnsibleOracle](https://github.com/RCAccelerator/rca-api/blob/main/rcav2/agent/ansible.py)**" agent that interprets the Ansible playbooks associated with the job.
4.  The core of the analysis is performed by a DSPy agent, which can be either a **[Predict](https://github.com/RCAccelerator/rca-api/blob/main/rcav2/agent/predict.py)** or a **[ReAct](https://github.com/RCAccelerator/rca-api/blob/main/rcav2/agent/react.py)** agent. The agent analyzes the errors, identifies potential root causes, and gathers evidence to support its conclusions.
5.  When the agent is used, it can query external systems like Jira and Slack to find related tickets and messages, or calling internal RCAv2 tools to make decisions based on the Zuul CI build job log folder structure, further enriching the context for the final report.
6.  The final report, which includes a summary of the analysis, a list of possible root causes with supporting evidence, and links to relevant Jira tickets and Slack conversations is streamed to the frontend and displayed to the user (or output to stdout in case of CLI).

## Multi And React Workflows

RCAv2 implements two distinct flows for interacting with the LLM, each represents different scenarios:

### The Multi Workflow
The Multi workflow, combines a **[dspy.ChainOfThought](https://dspy.ai/api/modules/ChainOfThought/)** module with the **[dspy.ReAct](https://dspy.ai/api/modules/ReAct/)** module.
* First chain of thought module (predict agent) receives a consolidated prompt with all the error information and generates a final report.
* Then the react module enriches the final report with the Jira data related to the possible root causes.

### The ReAct Workflow
**ReAct (Reasoning and Acting)** is an iterative workflow that is used when the initial data is insufficient.
* The ReAct agent, a **[dspy.ReAct](https://dspy.ai/api/modules/ReAct/)** module, can leverage external tools to gather additional context.
* It operates in a **[Thought -> Action -> Observation](https://huggingface.co/learn/agents-course/en/unit1/agent-steps-and-structure)** loop, where it first thinks about what to do next, then takes an action (e.g., calls a tool), and finally observes the result of that action.
* This loop continues until the agent decides it has gathered enough information to generate a comprehensive report.

## Challenges of Building RCA with LLM

Both the Multi and ReAct flows have their own set of challenges when it comes to generating accurate RCA reports.

The biggest challenge for the **Predict agent** is the lack of context. Since it operates on a single input (error report), it cannot seek out additional information if the initial error report is incomplete or misleading. This can lead to inaccurate or superficial root cause analysis.

The iterative nature of the **ReAct flow** can lead to "[context rot](https://research.trychroma.com/context-rot)" though. As the **Thought -> Action -> Observation** loop progresses, the context can become cluttered with the details of the reasoning process, potentially diluting the original error information and causing the LLM to lose focus. The ReAct agent has to perform a complex sequence of tasks: remembering the goal, integrating new facts, managing a growing context, and planning the next step. This high cognitive load can lead to errors and inefficiencies.

The ReAct agent can struggle to build an accurate timeline of events, which is crucial for understanding cause and effect. This can lead to failures in identifying the true root cause of a failure.

## Improvements

To address the issues described above, the my team identified several ways the RCA reporting could be improved:

1.  First, a **hybrid approach** that combines the speed of the Predict flow with the power of the ReAct flow could be developed. For example, the system could start with a ChainOfThought analysis and then switch to a ReAct analysis if the initial results are inconclusive.
2.  Second, the **temporal analysis capabilities** of the ReAct agent could be enhanced to provide a more accurate and detailed timeline of events. The most important part of the approach is to make sure each event in the logjuicer report has a valid timestamp. This would help the agent to better understand cause and effect relationships.
3.  Third, techniques for **[smarter context management](https://dspy.ai/tutorials/mem0_react_agent/#step-3-build-the-memory-enhanced-react-agent)** could be implemented to prevent context pollution and help the ReAct agent stay focused on the most relevant information. Here we can reuse approaches introduced by the RCAv1 like RAG or implement partial summarization and sliding window memory pattern.

By addressing these challenges and implementing these improvements in the future as part of the RCAv2 development cycle or as part of RCAv3 has the potential to become an even more useful and effective tool for root cause analysis of the Zuul CI build failures.
