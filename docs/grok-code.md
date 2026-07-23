**Conclusion**  
Build a Claude Code–equivalent agentic coding harness first, then plug in the research architecture (distilled models + single WASM binary + hybrid CPU/GPU routing) as the inference backend. Do not try to put the full agent loop inside the 4 GB WASM instance.

Claude Code is a tool-using agent loop with strong context engineering (CLAUDE.md memory, compaction, sub-agents, safety gates). The research prompt supplies the cost-efficient, portable inference layer. Treat them as two layers.

### Recommended Architecture
```
User / IDE / CLI
        ↓
Agent Harness (native Rust preferred)
  - Tool registry + event loop
  - Project memory (CLAUDE.md equivalent)
  - Context compaction / summarization
  - Complexity classifier + confidence check
        ↓
HTTP/gRPC Router (lightweight)
  - Routes simple → CPU pool (≤3B Q4)
  - Routes complex / low-confidence → GPU pool (7B–70B)
        ↓
WasmEdge instances (one binary)
  - MODEL_PATH + N_GPU_LAYERS from env
  - WASI-NN GGML plugin (mmap weights on host)
```

### Build Plan (10 Steps)

**1. Validate the inference foundation (1–2 days)**  
Run the memory-separation experiment from the research prompt: load a 7–8B Q4 GGUF under WasmEdge, confirm WASM heap stays ≪ 4 GB while host RAM/VRAM absorbs the model. Add a minimal confidence probe (top-1 logit or entropy). This is the only non-negotiable prerequisite.

**2. Core agent loop (the Claude Code primitive)**  
Implement the standard tool-calling loop:
- User message → model
- Model returns tool calls or final answer
- Execute tools, feed results back
- Repeat until done

Use any solid base (Go workshop pattern, Aider-style, or OpenCode-style). Start with Claude/GPT-4o or a strong open model so the harness works before you swap the backend.

**3. Essential tools only**  
Implement these five first (they cover >80 % of Claude Code value):
- `Read` (file or range)
- `Glob` / `ListDir`
- `Grep` (prefer ripgrep)
- `Bash` (with sandbox + allow-list)
- `Edit` / `Write` (prefer exact-string replace over free-form write)

Add `TodoWrite` early for multi-step tracking.

**4. Persistent project memory**  
Replicate CLAUDE.md:
- Auto-load `AGENTS.md` / `PROJECT.md` at session start
- Keep it short and high-signal (architecture, conventions, current plan)
- Treat the filesystem as primary memory; context window is secondary

**5. Context management**  
- Track token usage
- Implement micro-compaction (drop bulky tool results early) + full summarization when approaching limit
- After compaction, re-inject the project memory file and active todo list

**6. Complexity classifier + routing client**  
Replace the model call with a client that talks to your router:
- Cheap heuristic: token count + keywords (“refactor”, “debug”, “architecture”) + AST depth if cheap
- Call the small CPU model first
- If confidence < threshold → escalate to GPU model and return the better answer
- Measure escalation rate; target ≤20 %

**7. Swap in the research inference backend**  
Point the harness at the WasmEdge service (one binary, env-driven).  
Keep the same tool schemas and system prompt. The only change is the model endpoint and the routing logic.

**8. Safety and permission layer**  
- Explicit confirmation for destructive Bash / large writes
- Deny-list of dangerous commands
- Optional worktree isolation for risky sessions

**9. Sub-agent / plan mode (optional but high leverage)**  
- Simple “Explore” agent that only reads + greps and returns a summary
- “Plan” mode that forces a written plan before edits
These can run as separate harness instances or as specialized system prompts.

**10. Hybrid cluster + cost measurement**  
Deploy the two node pools, wire the router, run a realistic workload (80 % simple completions / 20 % complex), and measure:
- TCO per 1 M tokens
- Escalation rate
- End-to-end p95 latency
- Accuracy vs pure-GPU baseline

### Key Trade-offs
| Approach | Pros | Cons |
|----------|------|------|
| Pure Claude Code clone (API only) | Fastest to usable | Expensive, no control over model, no WASM portability |
| Harness + research backend | Meets the original research goals, cost scalable | Two systems to maintain; small models need good distillation |
| Full agent inside WASM | Maximal isolation | Hits 4 GB ceiling, tool execution becomes awkward |

### Main Risks
- Small models (≤3B) will underperform on complex multi-file reasoning → escalation rate becomes the real cost driver.
- Over-engineering the router/AST classifier before the basic loop is solid.
- Trying to hot-swap LoRAs inside the WASI-NN path before the base system works.
- Context compaction that loses critical state (mitigate with strong project memory files).

### Clear Next Step
Execute Step 1 (memory-separation experiment) and Step 2 (minimal tool-calling loop) in parallel this week. Once both exist, the rest of the plan becomes incremental engineering rather than research.

If you want concrete starter code for the agent loop (Rust or Python), the confidence probe, or the minimal router, specify the language and I will provide it.
