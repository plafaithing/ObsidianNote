Over the past year, we've worked with dozens of teams building large language model (LLM) agents across industries. Consistently, the most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns.

In this post, we share what we’ve learned from working with our customers and building agents ourselves, and give practical advice for developers on building effective agents.

## What are agents?

"Agent" can be defined in several ways. Some customers define agents as fully autonomous systems that operate independently over extended periods, using various tools to accomplish complex tasks. Others use the term to describe more prescriptive implementations that follow predefined workflows. At Anthropic, we categorize all these variations as **agentic systems**, but draw an important architectural distinction between **workflows** and **agents**:

- **Workflows** are systems where LLMs and tools are orchestrated through predefined code paths.
- **Agents**, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks.

Below, we will explore both types of agentic systems in detail. In Appendix 1 (“Agents in Practice”), we describe two domains where customers have found particular value in using these kinds of systems.

## When (and when not) to use agents

When building applications with LLMs, we recommend finding the simplest solution possible, and only increasing complexity when needed. This might mean not building agentic systems at all. Agentic systems often trade latency and cost for better task performance, and you should consider when this tradeoff makes sense.

When more complexity is warranted, workflows offer predictability and consistency for well-defined tasks, whereas agents are the better option when flexibility and model-driven decision-making are needed at scale. For many applications, however, optimizing single LLM calls with retrieval and in-context examples is usually enough.

## When and how to use frameworks

There are many frameworks that make agentic systems easier to implement, including:

- The [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview);
- [Strands Agents SDK by AWS](https://strandsagents.com/latest/);
- [Rivet](https://rivet.ironcladapp.com/), a drag and drop GUI LLM workflow builder; and
- [Vellum](https://www.vellum.ai/), another GUI tool for building and testing complex workflows.

These frameworks make it easy to get started by simplifying standard low-level tasks like calling LLMs, defining and parsing tools, and chaining calls together. However, they often create extra layers of abstraction that can obscure the underlying prompts ​​and responses, making them harder to debug. They can also make it tempting to add complexity when a simpler setup would suffice.

We suggest that developers start by using LLM APIs directly: many patterns can be implemented in a few lines of code. If you do use a framework, ensure you understand the underlying code. Incorrect assumptions about what's under the hood are a common source of customer error.

See our [cookbook](https://platform.claude.com/cookbook/patterns-agents-basic-workflows) for some sample implementations.

## Building blocks, workflows, and agents

In this section, we’ll explore the common patterns for agentic systems we’ve seen in production. We'll start with our foundational building block—the augmented LLM—and progressively increase complexity, from simple compositional workflows to autonomous agents.

### Building block: The augmented LLM

The basic building block of agentic systems is an LLM enhanced with augmentations such as retrieval, tools, and memory. Our current models can actively use these capabilities—generating their own search queries, selecting appropriate tools, and determining what information to retain.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fd3083d3f40bb2b6f477901cc9a240738d3dd1371-2401x1000.png&w=3840&q=75)

The augmented LLM

We recommend focusing on two key aspects of the implementation: tailoring these capabilities to your specific use case and ensuring they provide an easy, well-documented interface for your LLM. While there are many ways to implement these augmentations, one approach is through our recently released [Model Context Protocol](https://www.anthropic.com/news/model-context-protocol), which allows developers to integrate with a growing ecosystem of third-party tools with a simple [client implementation](https://modelcontextprotocol.io/tutorials/building-a-client#building-mcp-clients).

For the remainder of this post, we'll assume each LLM call has access to these augmented capabilities.

### Workflow: Prompt chaining

Prompt chaining decomposes a task into a sequence of steps, where each LLM call processes the output of the previous one. You can add programmatic checks (see "gate” in the diagram below) on any intermediate steps to ensure that the process is still on track.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

The prompt chaining workflow

**When to use this workflow:** This workflow is ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task.

**Examples where prompt chaining is useful:**

- Generating Marketing copy, then translating it into a different language.
- Writing an outline of a document, checking that the outline meets certain criteria, then writing the document based on the outline.

### Workflow: Routing

Routing classifies an input and directs it to a specialized followup task. This workflow allows for separation of concerns, and building more specialized prompts. Without this workflow, optimizing for one kind of input can hurt performance on other inputs.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

The routing workflow

**When to use this workflow:** Routing works well for complex tasks where there are distinct categories that are better handled separately, and where classification can be handled accurately, either by an LLM or a more traditional classification model/algorithm.

**Examples where routing is useful:**

- Directing different types of customer service queries (general questions, refund requests, technical support) into different downstream processes, prompts, and tools.
- Routing easy/common questions to smaller, cost-efficient models like Claude Haiku 4.5 and hard/unusual questions to more capable models like Claude Sonnet 4.5 to optimize for best performance.

### Workflow: Parallelization

LLMs can sometimes work simultaneously on a task and have their outputs aggregated programmatically. This workflow, parallelization, manifests in two key variations:

- **Sectioning**: Breaking a task into independent subtasks run in parallel.
- **Voting:** Running the same task multiple times to get diverse outputs.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

The parallelization workflow

**When to use this workflow:** Parallelization is effective when the divided subtasks can be parallelized for speed, or when multiple perspectives or attempts are needed for higher confidence results. For complex tasks with multiple considerations, LLMs generally perform better when each consideration is handled by a separate LLM call, allowing focused attention on each specific aspect.

**Examples where parallelization is useful:**

- **Sectioning**:
    - Implementing guardrails where one model instance processes user queries while another screens them for inappropriate content or requests. This tends to perform better than having the same LLM call handle both guardrails and the core response.
    - Automating evals for evaluating LLM performance, where each LLM call evaluates a different aspect of the model’s performance on a given prompt.
- **Voting**:
    - Reviewing a piece of code for vulnerabilities, where several different prompts review and flag the code if they find a problem.
    - Evaluating whether a given piece of content is inappropriate, with multiple prompts evaluating different aspects or requiring different vote thresholds to balance false positives and negatives.

### Workflow: Orchestrator-workers

In the orchestrator-workers workflow, a central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

The orchestrator-workers workflow

**When to use this workflow:** This workflow is well-suited for complex tasks where you can’t predict the subtasks needed (in coding, for example, the number of files that need to be changed and the nature of the change in each file likely depend on the task). Whereas it’s topographically similar, the key difference from parallelization is its flexibility—subtasks aren't pre-defined, but determined by the orchestrator based on the specific input.

**Example where orchestrator-workers is useful:**

- Coding products that make complex changes to multiple files each time.
- Search tasks that involve gathering and analyzing information from multiple sources for possible relevant information.

### Workflow: Evaluator-optimizer

In the evaluator-optimizer workflow, one LLM call generates a response while another provides evaluation and feedback in a loop.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

The evaluator-optimizer workflow

**When to use this workflow:** This workflow is particularly effective when we have clear evaluation criteria, and when iterative refinement provides measurable value. The two signs of good fit are, first, that LLM responses can be demonstrably improved when a human articulates their feedback; and second, that the LLM can provide such feedback. This is analogous to the iterative writing process a human writer might go through when producing a polished document.

**Examples where evaluator-optimizer is useful:**

- Literary translation where there are nuances that the translator LLM might not capture initially, but where an evaluator LLM can provide useful critiques.
- Complex search tasks that require multiple rounds of searching and analysis to gather comprehensive information, where the evaluator decides whether further searches are warranted.

### Agents

Agents are emerging in production as LLMs mature in key capabilities—understanding complex inputs, engaging in reasoning and planning, using tools reliably, and recovering from errors. Agents begin their work with either a command from, or interactive discussion with, the human user. Once the task is clear, agents plan and operate independently, potentially returning to the human for further information or judgement. During execution, it's crucial for the agents to gain “ground truth” from the environment at each step (such as tool call results or code execution) to assess its progress. Agents can then pause for human feedback at checkpoints or when encountering blockers. The task often terminates upon completion, but it’s also common to include stopping conditions (such as a maximum number of iterations) to maintain control.

Agents can handle sophisticated tasks, but their implementation is often straightforward. They are typically just LLMs using tools based on environmental feedback in a loop. It is therefore crucial to design toolsets and their documentation clearly and thoughtfully. We expand on best practices for tool development in Appendix 2 ("Prompt Engineering your Tools").

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F58d9f10c985c4eb5d53798dea315f7bb5ab6249e-2401x1000.png&w=3840&q=75)

Autonomous agent

**When to use agents:** Agents can be used for open-ended problems where it’s difficult or impossible to predict the required number of steps, and where you can’t hardcode a fixed path. The LLM will potentially operate for many turns, and you must have some level of trust in its decision-making. Agents' autonomy makes them ideal for scaling tasks in trusted environments.

The autonomous nature of agents means higher costs, and the potential for compounding errors. We recommend extensive testing in sandboxed environments, along with the appropriate guardrails.

**Examples where agents are useful:**

The following examples are from our own implementations:

- A coding Agent to resolve [SWE-bench tasks](https://www.anthropic.com/research/swe-bench-sonnet), which involve edits to many files based on a task description;
- Our [“computer use” reference implementation](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo), where Claude uses a computer to accomplish tasks.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F4b9a1f4eb63d5962a6e1746ac26bbc857cf3474f-2400x1666.png&w=3840&q=75)

High-level flow of a coding agent

## Combining and customizing these patterns

These building blocks aren't prescriptive. They're common patterns that developers can shape and combine to fit different use cases. The key to success, as with any LLM features, is measuring performance and iterating on implementations. To repeat: you should consider adding complexity _only_ when it demonstrably improves outcomes.

## Summary

Success in the LLM space isn't about building the most sophisticated system. It's about building the _right_ system for your needs. Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short.

When implementing agents, we try to follow three core principles:

1. Maintain **simplicity** in your agent's design.
2. Prioritize **transparency** by explicitly showing the agent’s planning steps.
3. Carefully craft your agent-computer interface (ACI) through thorough tool **documentation and testing**.

Frameworks can help you get started quickly, but don't hesitate to reduce abstraction layers and build with basic components as you move to production. By following these principles, you can create agents that are not only powerful but also reliable, maintainable, and trusted by their users.

### Acknowledgements

Written by Erik S. and Barry Zhang. This work draws upon our experiences building agents at Anthropic and the valuable insights shared by our customers, for which we're deeply grateful.

## Appendix 1: Agents in practice

Our work with customers has revealed two particularly promising applications for AI agents that demonstrate the practical value of the patterns discussed above. Both applications illustrate how agents add the most value for tasks that require both conversation and action, have clear success criteria, enable feedback loops, and integrate meaningful human oversight.

### A. Customer support

Customer support combines familiar chatbot interfaces with enhanced capabilities through tool integration. This is a natural fit for more open-ended agents because:

- Support interactions naturally follow a conversation flow while requiring access to external information and actions;
- Tools can be integrated to pull customer data, order history, and knowledge base articles;
- Actions such as issuing refunds or updating tickets can be handled programmatically; and
- Success can be clearly measured through user-defined resolutions.

Several companies have demonstrated the viability of this approach through usage-based pricing models that charge only for successful resolutions, showing confidence in their agents' effectiveness.

### B. Coding agents

The software development space has shown remarkable potential for LLM features, with capabilities evolving from code completion to autonomous problem-solving. Agents are particularly effective because:

- Code solutions are verifiable through automated tests;
- Agents can iterate on solutions using test results as feedback;
- The problem space is well-defined and structured; and
- Output quality can be measured objectively.

In our own implementation, agents can now solve real GitHub issues in the [SWE-bench Verified](https://www.anthropic.com/research/swe-bench-sonnet) benchmark based on the pull request description alone. However, whereas automated testing helps verify functionality, human review remains crucial for ensuring solutions align with broader system requirements.

## Appendix 2: Prompt engineering your tools

No matter which agentic system you're building, tools will likely be an important part of your agent. [Tools](https://www.anthropic.com/news/tool-use-ga) enable Claude to interact with external services and APIs by specifying their exact structure and definition in our API. When Claude responds, it will include a [tool use block](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-api-response-with-a-tool-use-content-block) in the API response if it plans to invoke a tool. Tool definitions and specifications should be given just as much prompt engineering attention as your overall prompts. In this brief appendix, we describe how to prompt engineer your tools.

There are often several ways to specify the same action. For instance, you can specify a file edit by writing a diff, or by rewriting the entire file. For structured output, you can return code inside markdown or inside JSON. In software engineering, differences like these are cosmetic and can be converted losslessly from one to the other. However, some formats are much more difficult for an LLM to write than others. Writing a diff requires knowing how many lines are changing in the chunk header before the new code is written. Writing code inside JSON (compared to markdown) requires extra escaping of newlines and quotes.

Our suggestions for deciding on tool formats are the following:

- Give the model enough tokens to "think" before it writes itself into a corner.
- Keep the format close to what the model has seen naturally occurring in text on the internet.
- Make sure there's no formatting "overhead" such as having to keep an accurate count of thousands of lines of code, or string-escaping any code it writes.

One rule of thumb is to think about how much effort goes into human-computer interfaces (HCI), and plan to invest just as much effort in creating good _agent_-computer interfaces (ACI). Here are some thoughts on how to do so:

- Put yourself in the model's shoes. Is it obvious how to use this tool, based on the description and parameters, or would you need to think carefully about it? If so, then it’s probably also true for the model. A good tool definition often includes example usage, edge cases, input format requirements, and clear boundaries from other tools.
- How can you change parameter names or descriptions to make things more obvious? Think of this as writing a great docstring for a junior developer on your team. This is especially important when using many similar tools.
- Test how the model uses your tools: Run many example inputs in our [workbench](https://console.anthropic.com/workbench) to see what mistakes the model makes, and iterate.
- [Poka-yoke](https://en.wikipedia.org/wiki/Poka-yoke) your tools. Change the arguments so that it is harder to make mistakes.

While building our agent for [SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet), we actually spent more time optimizing our tools than the overall prompt. For example, we found that the model would make mistakes with tools using relative filepaths after the agent had moved out of the root directory. To fix this, we changed the tool to always require absolute filepaths—and we found that the model used this method flawlessly.
过去一年，我们与数十个跨行业构建大型语言模型（LLM）智能体的团队合作。最成功的实现案例始终不是在使用复杂框架或专用库，而是采用简单、可组合的模式构建。

本文将分享我们从客户合作及自主构建智能体中获得的经验，并为开发者提供构建有效智能体的实用建议。

## 什么是智能体？

“智能体”可有多种定义方式。Over the past year, we've worked with dozens of teams building large language model (LLM) agents across industries. Consistently, the most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns.

In this post, we share what we’ve learned from working with our customers and building agents ourselves, and give practical advice for developers on building effective agents.

## What are agents?

"Agent" can be defined in several ways. Some customers define agents as fully autonomous systems that operate independently over extended periods, using various tools to accomplish complex tasks. Others use the term to describe more prescriptive implementations that follow predefined workflows. At Anthropic, we categorize all these variations as **agentic systems**, but draw an important architectural distinction between **workflows** and **agents**:

- **Workflows** are systems where LLMs and tools are orchestrated through predefined code paths.
- **Agents**, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks.

Below, we will explore both types of agentic systems in detail. In Appendix 1 (“Agents in Practice”), we describe two domains where customers have found particular value in using these kinds of systems.

## When (and when not) to use agents

When building applications with LLMs, we recommend finding the simplest solution possible, and only increasing complexity when needed. This might mean not building agentic systems at all. Agentic systems often trade latency and cost for better task performance, and you should consider when this tradeoff makes sense.

When more complexity is warranted, workflows offer predictability and consistency for well-defined tasks, whereas agents are the better option when flexibility and model-driven decision-making are needed at scale. For many applications, however, optimizing single LLM calls with retrieval and in-context examples is usually enough.

## When and how to use frameworks

There are many frameworks that make agentic systems easier to implement, including:

- The [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview);
- [Strands Agents SDK by AWS](https://strandsagents.com/latest/);
- [Rivet](https://rivet.ironcladapp.com/), a drag and drop GUI LLM workflow builder; and
- [Vellum](https://www.vellum.ai/), another GUI tool for building and testing complex workflows.

These frameworks make it easy to get started by simplifying standard low-level tasks like calling LLMs, defining and parsing tools, and chaining calls together. However, they often create extra layers of abstraction that can obscure the underlying prompts ​​and responses, making them harder to debug. They can also make it tempting to add complexity when a simpler setup would suffice.

We suggest that developers start by using LLM APIs directly: many patterns can be implemented in a few lines of code. If you do use a framework, ensure you understand the underlying code. Incorrect assumptions about what's under the hood are a common source of customer error.

See our [cookbook](https://platform.claude.com/cookbook/patterns-agents-basic-workflows) for some sample implementations.

## Building blocks, workflows, and agents

In this section, we’ll explore the common patterns for agentic systems we’ve seen in production. We'll start with our foundational building block—the augmented LLM—and progressively increase complexity, from simple compositional workflows to autonomous agents.

### Building block: The augmented LLM

The basic building block of agentic systems is an LLM enhanced with augmentations such as retrieval, tools, and memory. Our current models can actively use these capabilities—generating their own search queries, selecting appropriate tools, and determining what information to retain.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fd3083d3f40bb2b6f477901cc9a240738d3dd1371-2401x1000.png&w=3840&q=75)

The augmented LLM

We recommend focusing on two key aspects of the implementation: tailoring these capabilities to your specific use case and ensuring they provide an easy, well-documented interface for your LLM. While there are many ways to implement these augmentations, one approach is through our recently released [Model Context Protocol](https://www.anthropic.com/news/model-context-protocol), which allows developers to integrate with a growing ecosystem of third-party tools with a simple [client implementation](https://modelcontextprotocol.io/tutorials/building-a-client#building-mcp-clients).

For the remainder of this post, we'll assume each LLM call has access to these augmented capabilities.

### Workflow: Prompt chaining

Prompt chaining decomposes a task into a sequence of steps, where each LLM call processes the output of the previous one. You can add programmatic checks (see "gate” in the diagram below) on any intermediate steps to ensure that the process is still on track.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

The prompt chaining workflow

**When to use this workflow:** This workflow is ideal for situations where the task can be easily and cleanly decomposed into fixed subtasks. The main goal is to trade off latency for higher accuracy, by making each LLM call an easier task.

**Examples where prompt chaining is useful:**

- Generating Marketing copy, then translating it into a different language.
- Writing an outline of a document, checking that the outline meets certain criteria, then writing the document based on the outline.

### Workflow: Routing

Routing classifies an input and directs it to a specialized followup task. This workflow allows for separation of concerns, and building more specialized prompts. Without this workflow, optimizing for one kind of input can hurt performance on other inputs.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

The routing workflow

**When to use this workflow:** Routing works well for complex tasks where there are distinct categories that are better handled separately, and where classification can be handled accurately, either by an LLM or a more traditional classification model/algorithm.

**Examples where routing is useful:**

- Directing different types of customer service queries (general questions, refund requests, technical support) into different downstream processes, prompts, and tools.
- Routing easy/common questions to smaller, cost-efficient models like Claude Haiku 4.5 and hard/unusual questions to more capable models like Claude Sonnet 4.5 to optimize for best performance.

### Workflow: Parallelization

LLMs can sometimes work simultaneously on a task and have their outputs aggregated programmatically. This workflow, parallelization, manifests in two key variations:

- **Sectioning**: Breaking a task into independent subtasks run in parallel.
- **Voting:** Running the same task multiple times to get diverse outputs.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

The parallelization workflow

**When to use this workflow:** Parallelization is effective when the divided subtasks can be parallelized for speed, or when multiple perspectives or attempts are needed for higher confidence results. For complex tasks with multiple considerations, LLMs generally perform better when each consideration is handled by a separate LLM call, allowing focused attention on each specific aspect.

**Examples where parallelization is useful:**

- **Sectioning**:
    - Implementing guardrails where one model instance processes user queries while another screens them for inappropriate content or requests. This tends to perform better than having the same LLM call handle both guardrails and the core response.
    - Automating evals for evaluating LLM performance, where each LLM call evaluates a different aspect of the model’s performance on a given prompt.
- **Voting**:
    - Reviewing a piece of code for vulnerabilities, where several different prompts review and flag the code if they find a problem.
    - Evaluating whether a given piece of content is inappropriate, with multiple prompts evaluating different aspects or requiring different vote thresholds to balance false positives and negatives.

### Workflow: Orchestrator-workers

In the orchestrator-workers workflow, a central LLM dynamically breaks down tasks, delegates them to worker LLMs, and synthesizes their results.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

The orchestrator-workers workflow

**When to use this workflow:** This workflow is well-suited for complex tasks where you can’t predict the subtasks needed (in coding, for example, the number of files that need to be changed and the nature of the change in each file likely depend on the task). Whereas it’s topographically similar, the key difference from parallelization is its flexibility—subtasks aren't pre-defined, but determined by the orchestrator based on the specific input.

**Example where orchestrator-workers is useful:**

- Coding products that make complex changes to multiple files each time.
- Search tasks that involve gathering and analyzing information from multiple sources for possible relevant information.

### Workflow: Evaluator-optimizer

In the evaluator-optimizer workflow, one LLM call generates a response while another provides evaluation and feedback in a loop.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

The evaluator-optimizer workflow

**When to use this workflow:** This workflow is particularly effective when we have clear evaluation criteria, and when iterative refinement provides measurable value. The two signs of good fit are, first, that LLM responses can be demonstrably improved when a human articulates their feedback; and second, that the LLM can provide such feedback. This is analogous to the iterative writing process a human writer might go through when producing a polished document.

**Examples where evaluator-optimizer is useful:**

- Literary translation where there are nuances that the translator LLM might not capture initially, but where an evaluator LLM can provide useful critiques.
- Complex search tasks that require multiple rounds of searching and analysis to gather comprehensive information, where the evaluator decides whether further searches are warranted.

### Agents

Agents are emerging in production as LLMs mature in key capabilities—understanding complex inputs, engaging in reasoning and planning, using tools reliably, and recovering from errors. Agents begin their work with either a command from, or interactive discussion with, the human user. Once the task is clear, agents plan and operate independently, potentially returning to the human for further information or judgement. During execution, it's crucial for the agents to gain “ground truth” from the environment at each step (such as tool call results or code execution) to assess its progress. Agents can then pause for human feedback at checkpoints or when encountering blockers. The task often terminates upon completion, but it’s also common to include stopping conditions (such as a maximum number of iterations) to maintain control.

Agents can handle sophisticated tasks, but their implementation is often straightforward. They are typically just LLMs using tools based on environmental feedback in a loop. It is therefore crucial to design toolsets and their documentation clearly and thoughtfully. We expand on best practices for tool development in Appendix 2 ("Prompt Engineering your Tools").

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F58d9f10c985c4eb5d53798dea315f7bb5ab6249e-2401x1000.png&w=3840&q=75)

Autonomous agent

**When to use agents:** Agents can be used for open-ended problems where it’s difficult or impossible to predict the required number of steps, and where you can’t hardcode a fixed path. The LLM will potentially operate for many turns, and you must have some level of trust in its decision-making. Agents' autonomy makes them ideal for scaling tasks in trusted environments.

The autonomous nature of agents means higher costs, and the potential for compounding errors. We recommend extensive testing in sandboxed environments, along with the appropriate guardrails.

**Examples where agents are useful:**

The following examples are from our own implementations:

- A coding Agent to resolve [SWE-bench tasks](https://www.anthropic.com/research/swe-bench-sonnet), which involve edits to many files based on a task description;
- Our [“computer use” reference implementation](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo), where Claude uses a computer to accomplish tasks.

![](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F4b9a1f4eb63d5962a6e1746ac26bbc857cf3474f-2400x1666.png&w=3840&q=75)

High-level flow of a coding agent

## Combining and customizing these patterns

These building blocks aren't prescriptive. They're common patterns that developers can shape and combine to fit different use cases. The key to success, as with any LLM features, is measuring performance and iterating on implementations. To repeat: you should consider adding complexity _only_ when it demonstrably improves outcomes.

## Summary

Success in the LLM space isn't about building the most sophisticated system. It's about building the _right_ system for your needs. Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short.

When implementing agents, we try to follow three core principles:

1. Maintain **simplicity** in your agent's design.
2. Prioritize **transparency** by explicitly showing the agent’s planning steps.
3. Carefully craft your agent-computer interface (ACI) through thorough tool **documentation and testing**.

Frameworks can help you get started quickly, but don't hesitate to reduce abstraction layers and build with basic components as you move to production. By following these principles, you can create agents that are not only powerful but also reliable, maintainable, and trusted by their users.

### Acknowledgements

Written by Erik S. and Barry Zhang. This work draws upon our experiences building agents at Anthropic and the valuable insights shared by our customers, for which we're deeply grateful.

## Appendix 1: Agents in practice

Our work with customers has revealed two particularly promising applications for AI agents that demonstrate the practical value of the patterns discussed above. Both applications illustrate how agents add the most value for tasks that require both conversation and action, have clear success criteria, enable feedback loops, and integrate meaningful human oversight.

### A. Customer support

Customer support combines familiar chatbot interfaces with enhanced capabilities through tool integration. This is a natural fit for more open-ended agents because:

- Support interactions naturally follow a conversation flow while requiring access to external information and actions;
- Tools can be integrated to pull customer data, order history, and knowledge base articles;
- Actions such as issuing refunds or updating tickets can be handled programmatically; and
- Success can be clearly measured through user-defined resolutions.

Several companies have demonstrated the viability of this approach through usage-based pricing models that charge only for successful resolutions, showing confidence in their agents' effectiveness.

### B. Coding agents

The software development space has shown remarkable potential for LLM features, with capabilities evolving from code completion to autonomous problem-solving. Agents are particularly effective because:

- Code solutions are verifiable through automated tests;
- Agents can iterate on solutions using test results as feedback;
- The problem space is well-defined and structured; and
- Output quality can be measured objectively.

In our own implementation, agents can now solve real GitHub issues in the [SWE-bench Verified](https://www.anthropic.com/research/swe-bench-sonnet) benchmark based on the pull request description alone. However, whereas automated testing helps verify functionality, human review remains crucial for ensuring solutions align with broader system requirements.

## Appendix 2: Prompt engineering your tools

No matter which agentic system you're building, tools will likely be an important part of your agent. [Tools](https://www.anthropic.com/news/tool-use-ga) enable Claude to interact with external services and APIs by specifying their exact structure and definition in our API. When Claude responds, it will include a [tool use block](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-api-response-with-a-tool-use-content-block) in the API response if it plans to invoke a tool. Tool definitions and specifications should be given just as much prompt engineering attention as your overall prompts. In this brief appendix, we describe how to prompt engineer your tools.

There are often several ways to specify the same action. For instance, you can specify a file edit by writing a diff, or by rewriting the entire file. For structured output, you can return code inside markdown or inside JSON. In software engineering, differences like these are cosmetic and can be converted losslessly from one to the other. However, some formats are much more difficult for an LLM to write than others. Writing a diff requires knowing how many lines are changing in the chunk header before the new code is written. Writing code inside JSON (compared to markdown) requires extra escaping of newlines and quotes.

Our suggestions for deciding on tool formats are the following:

- Give the model enough tokens to "think" before it writes itself into a corner.
- Keep the format close to what the model has seen naturally occurring in text on the internet.
- Make sure there's no formatting "overhead" such as having to keep an accurate count of thousands of lines of code, or string-escaping any code it writes.

One rule of thumb is to think about how much effort goes into human-computer interfaces (HCI), and plan to invest just as much effort in creating good _agent_-computer interfaces (ACI). Here are some thoughts on how to do so:

- Put yourself in the model's shoes. Is it obvious how to use this tool, based on the description and parameters, or would you need to think carefully about it? If so, then it’s probably also true for the model. A good tool definition often includes example usage, edge cases, input format requirements, and clear boundaries from other tools.
- How can you change parameter names or descriptions to make things more obvious? Think of this as writing a great docstring for a junior developer on your team. This is especially important when using many similar tools.
- Test how the model uses your tools: Run many example inputs in our [workbench](https://console.anthropic.com/workbench) to see what mistakes the model makes, and iterate.
- [Poka-yoke](https://en.wikipedia.org/wiki/Poka-yoke) your tools. Change the arguments so that it is harder to make mistakes.

While building our agent for [SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet), we actually spent more time optimizing our tools than the overall prompt. For example, we found that the model would make mistakes with tools using relative filepaths after the agent had moved out of the root directory. To fix this, we changed the tool to always require absolute filepaths—and we found that the model used this method flawlessly.
过去一年，我们与数十个跨行业团队合作构建大语言模型代理。最成功的实施方案始终未使用复杂框架或专用库，而是采用简单、可组合的模式。

本文将分享我们与客户合作及自主构建代理过程中积累的经验，并为开发者提供构建高效代理的实用建议。

## 什么是代理？

"代理"有不同定义方式。部分客户将其定义为能长期独立运作、使用多种工具完成复杂任务的完全自主系统；另一些客户则用此术语描述遵循预设工作流的规范性实现。在Anthropic，我们将这些变体统称为**代理系统**，但对**工作流**与**代理**做出重要架构区分：

- **工作流**是通过预定义代码路径编排大语言模型和工具的系统。
- **代理**则是大语言模型动态主导自身流程与工具使用的系统，对任务完成方式保持控制权。

下文将详细探讨两类代理系统。附录1（"实践中的代理"）将介绍客户在这些系统中发现特别价值的两个领域。

## 何时（及何时不）使用代理

构建大语言模型应用时，我们建议寻找最简方案，仅在必要时增加复杂性。这可能意味着完全不需要构建代理系统。代理系统常以延迟和成本换取更优任务性能，开发者需权衡此取舍是否合理。

当需要更高复杂度时，工作流为明确任务提供可预测性和一致性；而在需要大规模灵活性和模型驱动决策时，代理是更优选择。但对多数应用而言，通过检索和上下文示例优化单次调用通常已足够。

## 何时及如何使用框架

许多框架能简化代理系统实施，包括：

- [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)
- [AWS Strands Agents SDK](https://strandsagents.com/latest/)
- [Rivet](https://rivet.ironcladapp.com/) 拖拽式图形界面LLM工作流构建器
- [Vellum](https://www.vellum.ai/) 用于构建和测试复杂工作流的图形界面工具

这些框架通过简化调用LLM、定义解析工具、链式调用等基础任务简化入门，但常增加抽象层，掩盖底层提示与响应，增加调试难度。更简单的设置足以满足需求时，它们也可能诱使开发者增加复杂性。

我们建议开发者直接使用LLM API：许多模式仅需数行代码即可实现。若使用框架，请确保理解底层代码。对框架的误解是客户常见错误根源。

参见我们的[示例实现代码库](https://platform.claude.com/cookbook/patterns-agents-basic-workflows)。

## 构建模块、工作流与代理

本节将探讨生产环境中常见的代理系统模式。从基础构建模块——增强型LLM开始，逐步增加复杂度，从简单组合式工作流到自主代理。

### 构建模块：增强型LLM

代理系统的基础构建模块是经过检索、工具和记忆等功能增强的LLM。当前模型能主动运用这些能力——自主生成搜索查询、选择合适工具、决定保留哪些信息。

增强型LLM

我们建议聚焦两个关键方面：根据具体用例定制这些能力，确保为LLM提供易用且文档完善的接口。实现方式多样，其中一种途径是通过我们近期发布的[模型上下文协议](https://www.anthropic.com/news/model-context-protocol)，开发者可通过简单的[客户端实现](https://modelcontextprotocol.io/tutorials/building-a-client#building-mcp-clients)集成不断扩展的第三方工具生态系统。

下文假设每次LLM调用都具备这些增强能力。

### 工作流：提示链式调用

提示链式调用将任务分解为步骤序列，每次LLM调用处理前一次输出。可在中间步骤添加程序化检查（见下图中"门控"）确保流程正确。

提示链式调用工作流

**适用场景：** 任务可清晰分解为固定子任务时。主要目标是通过让每次LLM调用处理更简单的子任务，以延迟换取更高精度。

**典型用例：**
- 生成营销文案后翻译为不同语言
- 先撰写文档大纲，检查是否符合标准，再基于大纲撰写文档

### 工作流：路由

路由对输入进行分类并导向专门的后续任务。此工作流实现关注点分离，构建更专精的提示。若无此工作流，优化某类输入可能损害其他输入的性能。

路由工作流

**适用场景：** 存在可通过LLM或传统分类模型/算法准确区分的不同类别，且这些类别需分别处理时。

**典型用例：**
- 将不同客户咨询（常规问题、退款请求、技术支持）导向不同下游流程、提示和工具
- 将常见问题路由至Claude Haiku 4.5等低成本模型，将复杂问题路由至Claude Sonnet 4.5等更强模型以优化性能

### 工作流：并行处理

LLM可同时处理任务并通过程序化聚合输出。此工作流有两种主要变体：

- **分块处理**：将任务分解为独立子任务并行运行
- **投票机制**：多次执行相同任务获取多样化输出

并行处理工作流

**适用场景：** 子任务可并行化提升速度，或需多方视角/多次尝试获取更高置信度时。对于涉及多重考量的复杂任务，让每个考量点由独立LLM调用处理通常表现更佳。

**典型用例：**
- **分块处理**：
  - 实现护栏机制：一个模型实例处理用户查询，另一个实例筛查不当内容。这通常优于让同次LLM调用同时处理护栏和核心响应
  - 自动化评估：每次LLM调用评估模型在特定提示下不同维度的表现
- **投票机制**：
  - 代码安全审查：多个不同提示分别审查标记问题代码
  - 内容合规评估：多提示评估不同维度或设定不同投票阈值平衡误报与漏报

### 工作流：编排器-执行器

在编排器-执行器工作流中，中央LLM动态分解任务、分派给执行LLM并合成结果。

编排器-执行器工作流

**适用场景：** 无法预判所需子任务的复杂任务（如编码时需修改的文件数量及性质取决于具体任务）。与并行处理的关键区别在于灵活性——子任务非预定义，由编排器根据输入确定。

**典型用例：**
- 每次需对多个文件进行复杂修改的编程产品
- 需从多来源收集分析信息的搜索任务

### 工作流：评估器-优化器

此工作流中，一次LLM调用生成响应，另一次LLM循环提供评估和反馈。

评估器-优化器工作流

**适用场景：** 具有明确评估标准且迭代优化可产生可衡量价值时。两个适用信号：首先，人工反馈能明显改进LLM响应；其次，LLM能提供此类反馈。类似于人类撰写精良文档时的迭代写作过程。

**典型用例：**
- 文学翻译中，翻译LLM可能遗漏微妙之处，评估LLM可提供有益批评
- 复杂搜索需多轮搜索分析收集全面信息，由评估器决定是否需要继续搜索

### 代理

随着LLM在关键能力（理解复杂输入、推理规划、可靠使用工具、错误恢复）上的成熟，代理逐渐进入生产环境。代理从人类指令或互动讨论开始工作，任务明确后自主规划运行，必要时返回人类获取信息或判断。执行过程中，代理需从环境获取"真实信息"（如工具调用结果或代码执行）评估进展，在检查点或遇到障碍时暂停等待人类反馈。任务通常在完成后终止，也可设置停止条件（如最大迭代次数）保持控制。

代理可处理复杂任务，但实施通常简单——本质上是基于环境反馈循环使用工具的LLM。因此需精心设计工具集及其文档。我们在附录2（"提示工程您的工具"）中详述工具开发最佳实践。

自主代理

**适用场景：** 开放性问题中，难以或无法预测所需步骤数，且无法硬编码固定路径时。LLM可能运行多轮，需对其决策有足够信任。代理的自主性使其适合在可信环境中扩展任务。

代理的自主性意味着更高成本和潜在错误累积风险。建议在沙盒环境中进行充分测试并设置适当护栏。

**典型用例：**

以下来自我们的实践：
- 解决[SWE-bench任务](https://www.anthropic.com/research/swe-bench-sonnet)的编程代理，需根据任务描述修改多个文件
- 我们的["计算机使用"参考实现](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)，Claude通过计算机完成操作

编程代理的高层流程

## 组合与定制这些模式

这些构建模块并非规定性方案，而是开发者可根据不同用例塑形组合的通用模式。与任何LLM功能一样，成功关键在于衡量性能并迭代实现。重申：应仅在明显改善结果时增加复杂性。

## 总结

LLM领域的成功不在于构建最复杂的系统，而在于构建**适合**需求的系统。从简单提示开始，通过全面评估优化，仅在简单方案无法满足需求时引入多步代理系统。

实施代理时，我们遵循三个核心原则：

1. 保持代理设计的**简洁性**
2. 通过显式展示代理规划步骤优先保证**透明度**
3. 通过彻底的工具**文档记录与测试**精心设计代理-计算机接口（ACI）

框架可帮助快速起步，但在投入生产时不妨减少抽象层，使用基础组件构建。遵循这些原则，可创建既强大又可靠、可维护且值得用户信赖的代理。

### 致谢

作者：Erik S. 与 Barry Zhang。本文借鉴了我们在Anthropic构建代理的经验及客户分享的宝贵见解，对此深表感谢。

## 附录1：实践中的代理

与客户合作中，我们发现了两个特别有前景的AI代理应用领域，展示了上述模式的实用价值。这两个应用都表明，代理在需要对话与行动结合、具有明确成功标准、支持反馈循环并融入有意义的监督的任务中能发挥最大价值。

### A. 客户支持

通过工具集成，客户支持结合了熟悉的聊天界面与增强功能。这对开放型代理是天然适用场景，因为：
- 支持互动遵循对话流程，同时需要访问外部信息并执行操作
- 可集成拉取客户数据、订单历史、知识库文章的工具
- 退款或工单更新等操作可编程处理
- 可通过用户定义的解决方案清晰衡量成功

多家公司已通过按次付费模式证明这种方法可行性——仅在成功解决时收费，显示对其代理效能的信心。

### B. 编程代理

软件开发领域展现出LLM功能的惊人潜力，能力从代码补全发展到自主解决问题。代理特别有效是因为：
- 代码解决方案可通过自动化测试验证
- 代理可利用测试结果反馈迭代方案
- 问题空间定义明确且结构化
- 输出质量可客观衡量

在我们的实现中，代理现已能仅基于拉取请求描述解决[SWE-bench Verified](https://www.anthropic.com/research/swe-bench-sonnet)基准测试中的真实GitHub问题。尽管自动化测试有助于验证功能，但人工审查对确保解决方案符合更广泛系统要求仍至关重要。

## 附录2：提示工程您的工具

无论构建何种代理系统，工具都将是重要组成部分。[工具](https://www.anthropic.com/news/tool-use-ga)通过我们在API中精确指定其结构和定义，使Claude能与外部服务和API交互。Claude响应时，如果计划调用工具，会在API响应中包含[工具使用块](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-api-response-with-a-tool-use-content-block)。工具定义和规格应与整体提示给予同等重视。本附录将简要描述如何进行工具提示工程。

同一动作常有多种指定方式。例如，文件编辑可通过差异格式指定，也可重写整个文件；结构化输出可在Markdown或JSON内返回代码。在软件工程中，这类差异是表面性的，可无损转换。但某些格式对LLM编写难度更大。编写差异需知晓块头中代码修改前后的行数变化；在JSON中编写代码（相比Markdown）需要对换行和引号进行额外转义。

我们的工具格式建议如下：
- 给模型足够令牌空间"思考"，避免陷入僵局
- 保持格式接近模型在互联网文本中常见的自然形式
- 确保无格式化"开销"，如需精确统计数千行代码或字符串转义

一条经验法则：思考人机交互（HCI）的投入力度，在创建优质_代理_-计算机接口（ACI）时应投入同等精力。具体建议：

- 站在模型角度思考：根据描述和参数，工具使用是否一目了然？否则模型也可能困惑。优质工具定义常包含使用示例、边界情况、输入格式要求及与其他工具的清晰界限。
- 如何修改参数名称或描述使意图更明确？想象为团队初级开发者编写精良docstring。在使用多个相似工具时尤为重要。
- 测试模型如何使用工具：在[工作台](https://console.anthropic.com/workbench)运行大量示例输入观察模型常见错误，并迭代改进
- 对工具实施[防错设计](https://en.wikipedia.org/wiki/Poka-yoke)：调整参数使错误更难发生

为SWE-bench构建代理时，我们实际花在优化工具上的时间超过整体提示。例如发现代理移出根目录后，使用相对路径的工具会出错。解决方案是要求工具始终使用绝对路径——模型此后完美使用方法。运作，使用多种工具完成复杂任务；另一些客户则用它描述遵循预设工作流的规范化实现。在Anthropic，我们将这些变体统称为**智能体系统**，但**工作流**与**智能体**之间存在着重要的架构差异：

- **工作流**：通过预定义代码路径协调LLM与工具的系统。
- **智能体**：由LLM动态引导自身流程与工具使用，自主控制任务完成方式的系统。

下文将详细探讨两类智能体系统。附录1（“实践中的智能体”）阐述客户在应用这类系统时获得显著价值的两个领域。

## 何时（与何时不）使用智能体

构建LLM应用时，建议优先寻找最简单方案，仅在必要时增加复杂度。这意味着可能根本不需要构建智能体系统——智能体系统常以延迟和成本换取更优任务性能，需权衡利弊。

当确需更高复杂度时，工作流为明确任务提供可预测性和一致性；而需要大规模灵活性和模型驱动决策时，智能体是更优选择。但对多数应用而言，通过检索和上下文示例优化单次LLM调用通常已足够。

## 何时及如何使用框架

诸多框架简化了智能体系统的实现，包括：

- [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)
- [AWS Strands Agents SDK](https://strandsagents.com/latest/)
- [Rivet](https://rivet.ironcladapp.com/)（拖放式GUI LLM工作流构建器）
- [Vellum](https://www.vellum.ai/)（用于构建和测试复杂工作流的GUI工具）

这些框架通过简化标准底层任务（如调用LLM、定义解析工具、链式调用）降低入门门槛，但常引入额外抽象层，掩盖底层提示与响应，增加调试难度，并容易诱使开发者增加不必要的复杂性。

建议开发者从直接使用LLM API开始：多数模式仅需几行代码即可实现。若使用框架，务必理解底层代码——对底层机制的误解是用户错误的常见来源。

示例实现请参阅我们的[代码库](https://platform.claude.com/cookbook/patterns-agents-basic-workflows)。

## 构建模块、工作流与智能体

本节将探讨生产环境中常见的智能体系统模式，从基础构建模块（增强型LLM）开始，逐步增加复杂度，直至自主智能体。

### 构建模块：增强型LLM

智能体系统的基础构建模块是经过检索、工具和记忆等增强的LLM。当前模型能主动使用这些能力——自主生成搜索查询、选择适当工具、决定保留哪些信息。

![增强型LLM](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2Fd3083d3f40bb2b6f477901cc9a240738d3dd1371-2401x1000.png&w=3840&q=75)

建议聚焦两个关键方面：针对具体用例定制这些能力，并确保为LLM提供易于使用的文档化接口。多种实现方法中，可通过近期发布的[模型上下文协议](https://www.anthropic.com/news/model-context-protocol)实现，开发者通过简单的[客户端实现](https://modelcontextprotocol.io/tutorials/building-a-client#building-mcp-clients)即可集成不断增长的第三方工具生态。

下文假设每次LLM调用都具备这些增强能力。

### 工作流：提示链

提示链将任务分解为连续步骤，每次LLM调用处理上一步输出。可在中间步骤添加程序化检查（参见下图中“门控”），确保流程按预期推进。

![提示链工作流](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F7418719e3dab222dccb379b8879e1dc08ad34c78-2401x1000.png&w=3840&q=75)

**适用场景：** 适用于可清晰分解为固定子任务的任务，主要通过牺牲延迟换取更高精度，使每次LLM调用变得更简单。

**典型应用：**
- 生成营销文案后翻译成另一种语言
- 编写文档大纲并验证其符合标准，再基于大纲撰写正文

### 工作流：路由

路由对输入进行分类，并导向专门的下游任务。该工作流实现关注点分离，构建更专业的提示。若无此工作流，针对某类输入的优化可能损害其他输入的性能。

![路由工作流](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F5c0c0e9fe4def0b584c04d37849941da55e5e71c-2401x1000.png&w=3840&q=75)

**适用场景：** 适用于存在明显分类需分别处理的复杂任务，且分类可由LLM或传统分类算法精确完成。

**典型应用：**
- 将客服查询（通用问题、退款申请、技术支持）导向不同下游流程、提示和工具
- 将简单常见问题路由至Claude Haiku 4.5等小型经济模型，难度问题路由至Claude Sonnet 4.5等更强模型以优化性能

### 工作流：并行化

LLM可同时处理任务并通过程序化聚合输出。并行化工作流有两种主要变体：

- **分段**：将任务分解为独立子任务并行执行
- **投票**：多次执行同一任务获取多样化输出

![并行化工作流](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F406bb032ca007fd1624f261af717d70e6ca86286-2401x1000.png&w=3840&q=75)

**适用场景：** 当子任务可并行化提升速度，或需要多角度/多次尝试以获得更高置信度结果时有效。处理涉及多个维度的复杂任务时，分派独立LLM调用处理各维度通常性能更优。

**典型应用：**
- **分段**：实现防护机制，一个模型实例处理用户查询，另实例审查不当内容（比单次调用同时处理两者更优）；自动化评估LLM性能的不同维度
- **投票**：代码安全审查，多个提示独立检测问题；内容适切性评估，多提示评估不同方面或设定不同投票阈值平衡误报与漏报

### 工作流：编排器-工作者

编排器-工作者工作流中，中央LLM动态分解任务，分配给工作者LLM并合成结果。

![编排器-工作者工作流](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F8985fc683fae4780fb34eab1365ab78c7e51bc8e-2401x1000.png&w=3840&q=75)

**适用场景：** 适用于无法预判子任务的复杂任务（如编码中需修改的文件数量及变更性质取决于任务）。与并行化的关键区别在于灵活性——子任务不预定义，由编排器根据具体输入决定。

**典型应用：**
- 每次需对多个文件进行复杂变更的编码产品
- 需从多来源收集分析信息的搜索任务

### 工作流：评估器-优化器

评估器-优化器工作流中，一次LLM调用生成响应，另一次提供循环评估与反馈。

![评估器-优化器工作流](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F14f51e6406ccb29e695da48b17017e899a6119c7-2401x1000.png&w=3840&q=75)

**适用场景：** 当具有明确评估标准且迭代优化能带来可衡量价值时特别有效。两大适配标志：人类反馈能显著改进LLM响应；LLM能提供此类反馈。这类似于人类撰写精良文稿的迭代过程。

**典型应用：**
- 文学翻译中，翻译LLM可能遗漏细微之处，评估LLM可提供有效修改建议
- 需多轮搜索分析的复杂搜索任务，由评估器决定是否需要继续搜索

### 智能体

随着LLM在理解复杂输入、推理规划、可靠使用工具和错误恢复等关键能力上的成熟，智能体正在生产中崭露头角。智能体通过人类用户的指令或交互讨论开始工作，任务明确后自主规划运作，必要时返回人类寻求信息或判断。执行过程中，智能体需从环境获取“地面真相”（如工具调用结果或代码执行）评估进度，可在检查点或遇到阻碍时暂停等待人类反馈。任务通常完成后终止，但常包含停止条件（如最大迭代次数）以保持控制。

智能体能处理复杂任务，但实现通常简单直接，本质上是基于环境反馈循环使用工具的LLM。因此，精心设计工具集及其文档至关重要。工具开发最佳实践详见附录2（“提示工程你的工具”）。

![自主智能体](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F58d9f10c985c4eb5d53798dea315f7bb5ab6249e-2401x1000.png&w=3840&q=75)

**适用场景：** 适用于步骤数难以预测的开放式问题，以及无法硬编码固定路径的场景。LLM可能运行多轮，需对其决策有一定信任。智能体的自主性使其成为可信环境中规模化任务的理想选择。

自主性意味着更高成本和潜在错误累积风险。建议在沙盒环境中进行广泛测试，并配置适当防护措施。

**典型应用：**
- 解决[SWE-bench任务](https://www.anthropic.com/research/swe-bench-sonnet)的编码智能体（根据任务描述编辑多个文件）
- 我们的[“计算机使用”参考实现](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)，Claude通过计算机完成任务

![编码智能体高层流程](https://www.anthropic.com/_next/image?url=https%3A%2F%2Fwww-cdn.anthropic.com%2Fimages%2F4zrzovbb%2Fwebsite%2F4b9a1f4eb63d5962a6e1746ac26bbc857cf3474f-2400x1666.png&w=3840&q=75)

## 组合与定制这些模式

这些构建模块并非规定性模板，而是开发者可塑形组合的通用模式。与所有LLM特性一样，成功关键在于衡量性能并迭代实现。重申：仅当能证明提升效果时才应考虑增加复杂性。

## 总结

LLM领域的成功不在于构建最复杂的系统，而在于构建**适合需求**的系统。从简单提示开始，通过全面评估优化，仅当简单方案不足时再引入多步骤智能体系统。

实现智能体时，我们遵循三项核心原则：

1. 保持**简洁性**的设计
2. 通过显式展示计划步骤优先保证**透明度**
3. 通过彻底的**文档化与测试**精心构建智能体-计算机接口（ACI）

框架可助快速入门，但进入生产环境时不要犹豫减少抽象层，使用基础组件构建。遵循这些原则，你能创建既强大又可靠、可维护且受用户信赖的智能体。

### 致谢

由Erik S.和Barry Zhang撰写。本文借鉴了我们在Anthropic构建智能体的经验及客户分享的宝贵见解，对此深表感谢。

## 附录1：实践中的智能体

通过与客户合作，我们发现了AI智能体的两个特别有前景的应用方向，体现了上述模式的实际价值。两个应用都说明智能体在需要对话与行动结合、有明确成功标准、支持反馈循环且整合有意义的人类监督时能发挥最大价值。

### A. 客户支持

客户支持结合了熟悉的聊天界面与通过工具集成增强的能力，天然适合更开放的智能体：
- 支持交互遵循对话流程，同时需要访问外部信息和执行操作
- 可集成工具拉取客户数据、订单历史、知识库文章
- 退款、工单更新等操作可编程处理
- 通过用户定义的解决方案明确衡量成功

多家公司通过按成功解决方案计费的模式验证了该方法的可行性，展示了对其智能体效能的信心。

### B. 编码智能体

软件开发领域展现了LLM功能的卓越潜力，能力从代码补全演进到自主解决问题。智能体特别有效的原因为：
- 代码解决方案可通过自动化测试验证
- 智能体可利用测试结果迭代优化
- 问题空间定义明确且结构化
- 输出质量可客观衡量

在我们的实现中，智能体已能仅根据拉取请求描述解决[SWE-bench Verified](https://www.anthropic.com/research/swe-bench-sonnet)基准中的真实GitHub问题。但尽管自动化测试验证功能，人工审查对确保解决方案符合整体系统要求仍至关重要。

## 附录2：提示工程你的工具

无论构建何种智能体系统，工具都是重要组成部分。[工具](https://www.anthropic.com/news/tool-use-ga)通过在我们的API中指定精确结构和定义，使Claude能交互外部服务。Claude响应时，若计划调用工具，将在API响应中包含[工具使用块](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#example-api-response-with-a-tool-use-content-block)。工具定义和规范应获得与总体提示同等程度的提示工程关注。

同一操作常有多种指定方式：例如，可通过编写差异（diff）或重写整个文件来指定文件编辑。对于结构化输出，可在Markdown或JSON中返回代码。软件工程中这类差异是表象性的，可无损转换。但某些格式对LLM来说更难编写：编写差异需在新代码前了解块头中变更的行数；在JSON中编写代码（相比Markdown）需要对换行和引号进行额外转义。

决定工具格式的建议：
- 给模型足够令牌“思考”，避免陷入僵局
- 保持格式接近互联网文本中自然出现的形态
- 确保无格式化“开销”（如准确计数数千行代码或对生成的代码进行字符串转义）

一条经验法则：思考人机交互（HCI）的投入力度，对创建良好**智能体**-计算机接口（ACI）投入等同努力。具体做法：
- 换位思考：基于描述和参数，工具使用是否显而易见？需要深思熟虑吗？如果是，对模型亦然。好的工具定义通常包含示例用法、边界情况、输入格式要求及与其他工具的明确边界。
- 参数命名/描述如何能更直观？如同为团队新人编写优秀文档字符串——使用多个相似工具时尤其重要。
- 测试模型工具使用：在[工作台](https://console.anthropic.com/workbench)中运行多种示例输入观察模型错误并迭代。
- [防错设计](https://en.wikipedia.org/wiki/Poka-yoke)工具：修改参数使其更难出错。

在构建[SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet)智能体时，我们优化工具所花时间远超优化整体提示。例如发现智能体移出根目录后，使用相对文件路径的工具会出现错误。修复方法是将工具改为始终要求绝对路径——模型此后完美执行该方法。