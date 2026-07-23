Understood. Here is the **full English version** of the research on building a Claude Code–like system in **7 clear steps**.

---

## Research: How to Build a Claude Code–style AI Coding Agent in 7 Steps

At its heart, an AI coding agent like Claude Code is just a **tool‑calling loop** wrapped with a robust system prompt, permission controls, and context management. While Anthropic's production system has hundreds of thousands of lines of code, the core concept can be implemented in **~5000 lines** for a working prototype.

### The Core Loop (The 6‑line Heart)

Any coding agent boils down to this loop:

```python
while True:
    response = client.messages.create(          # 1. Call the LLM
        model=MODEL,
        messages=messages,
        tools=TOOLS                           # 2. Provide available tools
    )
    messages.append({"role": "assistant", "content": response.content})
    
    if response.stop_reason != "tool_use":    # 3. Check if finished
        return response
      
    tool_results = execute_all_tools(response.content) # 4. Execute tools
    messages.append({"role": "user", "content": tool_results}) # 5. Feed results back
```

This loop is the engine. Everything else is scaffolding around it.

---

### Step 1: Build the Agent Loop (The Engine)

Your loop must:
- Maintain a conversation history (`messages`).
- Call the LLM with the system prompt, history, and tool definitions.
- Parse the LLM's response – if it requests a tool call, execute it; if it provides a final answer, stop.
- Handle streaming, retries, and interruptions gracefully.

**Key design decisions:**
- Use **streaming** to show progress in real time.
- Implement **timeouts** and **max iteration limits** to prevent infinite loops.
- Support **parallel tool calls** (many modern models can call multiple tools in one turn).

---

### Step 2: Design the Tool System (The Hands & Feet)

You need a set of tools that allow the agent to read, write, and execute code. The minimal set:

| Tool | Purpose |
| :--- | :--- |
| `Read(file_path)` | Read a file’s content. |
| `Write(file_path, content)` | Create or overwrite a file. |
| `Edit(file_path, old_string, new_string)` | Apply a targeted change. |
| `Glob(pattern)` | Find files by pattern. |
| `Grep(pattern)` | Search inside files (like `rg`). |
| `Bash(command)` | Execute shell commands (with strict sandboxing). |
| `TodoWrite(tasks)` | Manage a task list for the agent itself. |

**Implementation tips:**
- Wrap each tool with a **schema** (JSON‑like) that the LLM can understand.
- Validate inputs (e.g., prevent `Bash("rm -rf /")`).
- Log every tool execution for debugging and auditing.

---

### Step 3: Craft the System Prompt (The Brain)

The system prompt defines the agent's personality, rules, and workflow. A strong prompt includes:

1. **Role definition** – "You are a world‑class software engineer."
2. **Workflow instructions** – e.g., "First, read the relevant files. Then plan your approach. Write a todo list. Then implement changes step by step."
3. **Tool usage guidelines** – e.g., "Always use `Bash` to run tests after editing."
4. **Context injection** – project structure, coding standards, dependencies.

Example snippet:
```
You are an expert software engineer. You have access to the following tools:
- Read, Write, Edit, Glob, Grep – for file operations.
- Bash – for running commands (sandboxed).
- TodoWrite – to track your progress.

When given a task, follow this process:
1. Use `Glob` and `Grep` to understand the codebase.
2. Write a plan using `TodoWrite`.
3. Implement changes using `Edit` or `Write`.
4. Run tests with `Bash`.
5. If tests fail, fix them and repeat.
6. Provide a final summary.
```

---

### Step 4: Implement Context Management (Memory)

LLM context windows are finite. To handle long conversations and large codebases:

- **Compression** – Summarise old messages when the window gets full.
- **Sliding window** – Keep only the last N turns.
- **Semantic memory** – Embed important code snippets and retrieve them on demand (RAG).
- **State pruning** – Drop tool outputs that are no longer needed (e.g., after a successful edit, you don't need to keep the full file content).

**Practical approach:** Start with a simple sliding window (e.g., last 20 messages) and add summarisation later if needed.

---

### Step 5: Manage State & Sessions (Persistence)

Allow users to resume sessions. Store:
- Full conversation history.
- Current todo list.
- List of modified files (with diffs).
- Environment variables and working directory.

Store this in JSON or a lightweight database (SQLite). On resume, reconstruct the agent's state and continue from where it left off.

---

### Step 6: Add Permissions & Security (Safety)

Running arbitrary shell commands is dangerous. Implement:

| Feature | Description |
| :--- | :--- |
| **Permission modes** | `Auto` – approve all safe actions; `Ask` – ask before every action; `Plan` – only produce a plan, never execute. |
| **Rule system** | Allow users to define rules like "never edit `/etc`", "always ask before `git push`". |
| **Dangerous command detection** | Block `rm -rf /`, `:(){ :|:& };:`, etc. |
| **Sudo prevention** | Disallow commands that require `sudo` unless explicitly configured. |
| **User confirmation** | For high‑risk actions, present a clear diff and ask for approval. |

---

### Step 7: Advanced Features (Optional but Powerful)

Once the basics work, you can add:

- **Skills** – Allow users to inject domain‑specific knowledge via `SKILL.md` files. The agent reads these files and adapts its behaviour (e.g., "use our internal React component library").
- **Sub‑agents** – The main agent can delegate subtasks to specialised sub‑agents (e.g., a "testing agent" that only writes unit tests). This keeps the main context clean.
- **MCP (Model Context Protocol)** – Integrate with external data sources (Jira, GitHub, Slack) via MCP servers.
- **Multi‑model routing** – Use a cheap model (e.g., 3B Q4) for simple tasks and a large model (e.g., 70B) for complex ones – exactly fitting your hybrid cluster idea!

---

## Learning Resources – Build It Yourself, Step by Step

Several open‑source projects show exactly how to build such an agent from scratch:

| Project | Language | Size | Best for |
| :--- | :--- | :--- | :--- |
| **claude-code-from-scratch** | TypeScript / Python | ~5000 lines, 13 chapters | A comprehensive, structured tutorial. Each chapter adds one feature. |
| **agent-zero-to-hero** | Python | ~5000 lines, 7‑week course | A course that builds an agent; costs < $0.50 to run through the whole course (no API key needed for testing). |
| **mini-claude-code** | Python | ~1100 lines, 5 versions | A minimal progressive tutorial; start with `v0_bash_agent.py` (~50 lines) and add features one by one. |
| **claude-code-handbook** | N/A | Documentation + workshops | A knowledge base with practical recipes and an "Agent Runtime" workshop that walks you through implementing a Claude‑style agent. |

### Recommended Path

1. **Start with `mini-claude-code`** – get the core loop working in ~50 lines.
2. **Then follow `claude-code-from-scratch`** – systematically add file tools, editing, permission modes, etc.
3. **Optionally, take the `agent-zero-to-hero` course** – it gives you a deep understanding of the theoretical underpinnings.
4. **Adapt your Rust/Candle WASM stack** – once you understand the logic, rewrite the core loop in Rust, use Candle for inference, and package it for WasmEdge with your hybrid cluster architecture.

---

## Summary: Your 7‑Step Implementation Plan

| Step | Component | Key Deliverable |
| :--- | :--- | :--- |
| 1 | Agent Loop | A working `while` loop that calls the LLM and handles tool requests. |
| 2 | Tool System | Implement `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Bash`, `TodoWrite`. |
| 3 | System Prompt | Write a comprehensive prompt that guides the agent's workflow. |
| 4 | Context Management | Implement sliding window or summarisation. |
| 5 | State & Sessions | Store and restore conversation history. |
| 6 | Permissions & Security | Add mode selection (`Auto`/`Ask`/`Plan`) and dangerous command filtering. |
| 7 | Advanced Features | Optionally add Skills, Sub‑agents, or MCP integration. |

If you want the **Rust + Candle + WASM implementation details** for any of these steps (e.g., how to expose the tools as WASI‑NN‑compatible functions, or how to manage memory for the KV cache), just ask and I'll provide the code snippets.
