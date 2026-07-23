To build a **Claude Code**-class agent, you're not building a chat loop. You're building a streaming, cancellable, recursive state machine.

Claude Code itself is an agentic harness around Claude — it works in three phases: **gather context, take action, verify results** via tools, adapting based on previous steps. The core is a simple while-loop that calls the model, runs tools, and repeats — most code lives in the systems around it: permissions, 5-layer compaction, extensibility, subagents, session storage.

Here is the source-derived blueprint distilled into an actionable build.

## The 14-Step Roadmap (Each step compiles and runs)

This is the canonical order from the 18-chapter reverse-engineering. Each step is isolated.

**1. Tool interface + factory**
Define `Tool<I,O,P>` with Zod schema as both validator and JSON Schema. Use a `buildTool()` factory with safe defaults: `isParallelSafe => false`, `isReadOnly => false`. Never construct literals directly — fail-closed. Ship one tool: `Read`. Test JSON Schema generation.

**2. Query loop v0**
Async generator only, no tools. This pattern gives you backpressure, cancellation via `.return()`, and typed terminal states for free.

```ts
async function* query(initialState){
  let state = initialState
  while(true){
    state = compress(state) // 4 layers later
    const response = await stream(model, state)
    yield response.messages
    if(!response.toolCalls) return {kind:'completed'}
    //...
  }
}
```

Terminal reasons include `completed`, `max_turns`, `error`, `aborted_streaming`, etc.

**3. Tool execution path**
Implement the 14-step pipeline as one function: lookup, abort check, Zod validation, semantic validation, PreToolUse hooks, permission resolution, execute, result budgeting, PostToolUse hooks. Invariant: every `tool_use` must have a paired `tool_result` before next API call.

**4. Permission modes + rules**
Seven modes most→least permissive: `bypassPermissions`, `dontAsk`, `auto`, `acceptEdits`, `default`, `plan`, `bubble`. Sub-agents default to `bubble`. Resolution chain: Hook → allowed/denied/ask rules → tool.checkPermissions → mode default → prompt → classifier.

**5. Concurrency partition + executor**
Safety is per-invocation, not per-tool-type: `Bash("ls")` safe, `Bash("rm -rf")` not. Partition `[Read][Read][Grep][Edit][Read]` → `[concurrent[Read][Read][Grep], serial[Edit], concurrent[Read]]`. Yield in submission order, not completion order.

**6. Hook system v0**
Two events first: `PreToolUse` (can block/modify) and `PostToolUse`. Freeze config at startup via snapshot — defense against malicious repo modifying settings mid-session. Exit 0=success, 2=blocking error.

**7. State split**
Two tiers: bootstrap `STATE` ~80 fields (cwd, sessionId, cost) mutable via setters, and reactive AppState store (messages, approvals) immutable snapshots. Reactive store in 34 lines with updater-only mutations and `Object.is` guard.

**8. Multi-provider client factory**
`getAnthropicClient()` dispatches to Direct API, Bedrock, Vertex, Azure. Wrap fetch with `x-client-request-id` UUID.

**9. Prompt caching as architecture**
System prompt has literal `=== DYNAMIC BOUNDARY ===` marker: above = global cache (static), below = per-session. Every runtime `if` above doubles cache key space. Use sticky latches — once true, never false — to prevent mid-session busts of 50-70K cached tokens. Cap default `max_tokens` at 8K, retry 64K on truncation — recovers 12-28% context.

**10. Compression v1**
Run before every API call in strict order: tool result budget → snip compact → microcompact → context collapse → auto-compact. Auto-compact triggers at `effectiveContextWindow - 13,000`.

**11. Streaming tool executor**
Watch stream, start tool as soon as `tool_use` block fully parsed — admission rule: run if no tool running or all safe. Saves 2.5s → 2.6s vs sequential 3.1s.

**12. AgentTool + sub-agent lifecycle**
15-step lifecycle: model resolution, agent ID, CLAUDE.md stripping for Explore (saves 10.2% cache tokens), permission isolation, tool resolution (fork agents reuse exact array byte-for-byte for cache), context creation. Fork agents achieve byte-identical prefix → 90%+ input savings for children 2-5 via placeholder results.

**13. Memory: files, not DB**
Layout `~/.claude/projects/<hash>/memory/MEMORY.md` (≤200 lines index) + per-topic files. Four types: user, feedback, project, reference. Two-tier retrieval: always-load index + async LLM recall side-query returning ≤5 files. Write path: write file + pointer to index.

**14. Skills + MCP**
Skills use two-phase loading: frontmatter (name, when_to_use) at startup, body on invocation. MCP is universal external-tool protocol with 8 transports: stdio default, http, ws, sdk, InProcessTransport (63 lines). Wrapping: name → `mcp__{server}__{tool}`, truncate desc at 2048 chars. Critical rule: MCP skills never execute shell.

## Your WASM-Optimized 10-Step Variant

Now map that to your v3 prompt — same WASM binary for CPU and GPU:

| Step | What to Build | Validation |
| --- | --- | --- |
| 1 | Rust toolchain + WasmEdge plugin build with `WASMEDGE_PLUGIN_WASI_NN_BACKEND=GGML` | `wasmedge --version` shows GGML |
| 2 | Distill Qwen2.5-Coder-3B (52.4% HumanEval base) to Q4_K_M GGUF 1.8GB | HumanEval ≥45% post-quant |
| 3 | Rust shim: `try_load_gpu()` — attempt GPU load, fallback to CPU env var on WASI-NN err | Works on t3.xlarge without CUDA |
| 4 | Implement Query Loop as async generator in Rust/Tokio with OpenTelemetry | `cargo test loop_cancel` |
| 5 | Add 3 core tools: Read (mmap'd model), Edit (LoRA r=8, 9MB heap), Bash with per-invocation safety AST parse | Permission tests |
| 6 | Build prompt-cache architecture: static prefix (system rules) + dynamic suffix (project files) | Cache hit rate >90% |
| 7 | Deploy wasmCloud lattice with NATS, label pools `pool=cpu` and `pool=gpu` | `wash lattice` shows hosts |
| 8 | Rust sidecar router: complexity = tokens * sqrt(AST_nodes) * keyword_weight, <10ms | p95 routing 8ms |
| 9 | Confidence fallback: entropy <0.6 escalate small→large, ≤20% rate | Simulation of cascade |
| 10 | Cost harness: measure tokens/sec, heap <3.5GB, VRAM 8B Q4 ≤6GB, 70B ≤42GB | TCO ≥60% saving vs GPU-only |

The Rust rewrite of Claude Code (`ccrust`) already proves this stack works — Rust-native, supports MCP, tools, multiple providers, with prompt compacting keeping cache format. For WASM sandboxing, `openskills` runtime shows the pattern: TS/JS → WASM via javy-codegen, WASI 0.3 execution, native runner with seatbelt/Landlock.

Start with steps 1-3 this week — you get a runnable hybrid binary. Steps 4-6 give you the agentic loop. Steps 7-10 give you production cost wins.
