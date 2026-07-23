To build a Claude Code-like CLI coding agent from scratch, you can follow a structured approach modeled after the open-source Python project `woodx9/build-your-claude-code-from-scratch`. Here is a step-by-step breakdown of how to construct the system:

**Step 1: Environment and Core Architecture Setup**
Start by setting up a Python (3.8+) environment and configuring environment variables for your chosen LLM API (such as Anthropic, OpenAI, or OpenRouter). You will need to scaffold four main architectural components:

* An `APIClient` to handle external LLM requests.
* A `ConversationManager` to store and manage message history.
* A `ToolManager` to register and execute local system commands.
* A `BaseAgent` to handle the core decision loop.



**Step 2: Implement Tool Call Fundamentals**
The agent needs the ability to take action on your machine. You must implement tool-calling interfaces, supporting either native JSON Schema function calling or XML-based tool calls (which are highly compatible with text-only models). These tools will allow the agent to read files, write code, and execute terminal commands.

**Step 3: Build the ReAct (Reasoning and Acting) Loop**
Implement the Think-Act-Observe cycle. When a user provides a prompt, the agent must recursively think about the problem, decide which tool to use, execute the tool, observe the terminal output, and then decide its next step. Because the agent executes local code, you must build a user approval system with safety controls to intercept and authorize dangerous operations before they run.

**Step 4: State, Cost, and Memory Management**
As the agent recursively calls tools, token usage will grow rapidly. Expand your `ConversationManager` to track session costs and generate session summaries. Ensure that your state management guarantees the preservation of core system prompts and recent context during long-running tasks.

**Step 5: Smart Context Cropping**
Coding agents quickly exhaust their context windows by reading large files or generating long terminal logs. You will need to implement precision context cropping strategies. A common approach is keeping the TOP and BOTTOM messages of a conversation while gracefully compressing or truncating the middle to stay within the model's token limits.

**Step 6: Advanced Task and Workflow Organization**
To handle complex coding requests, build a workflow breakdown mechanism. For example, implement a "TodoWrite" tool that forces the agent to convert abstract user requests into a structured checklist. The agent can then manage the lifecycle of these sub-tasks (pending, in progress, completed) in real-time, acting as a quality assurance gate that prevents the agent from declaring a job finished prematurely.
