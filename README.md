# wasmai

**Credit-aware coding agent + org-owned WASM inference.**  
Capture every enterprise session → train smaller models that burn fewer credits.

| | |
|---|---|
| **Status** | Research → production roadmap (`TODO.md`). Scaffold only. |
| **Runtime** | Rust agent harness · WasmEdge + WASI-NN (GGUF mmap) |
| **Org models** | Qwen2.5-Coder class → Q4_K_M · ≤4 GiB WASM heap |
| **Teachers** | Claude · OpenAI · Kimi · other enterprise APIs · wasmai |

> **GitHub short description** (≤350 chars):  
> *Claude Code–class agent with multi-provider capture (Claude/OpenAI/Kimi), credit-aware routing, and continual training of org-owned WASM models (WasmEdge/Candle) so simple work stops burning enterprise API credits.*

---

## Why wasmai

Teams already pay for heavy models (Claude, OpenAI, Kimi, …). Those sessions are full of org-specific APIs, style, and failure/recovery paths—but the default is still to **re-spend credits on every similar task**.

**wasmai** closes the loop:

1. **Agent harness** — Claude Code–style tools, permissions, sessions (native Rust).
2. **Multi-provider gateway** — one client surface for enterprise APIs **and** org models; redact secrets/PII on every hop.
3. **Credit ladder** — prefer cheap org-small → org-large → mid API → heavy API only when needed.
4. **Continual learning** — harvest redacted telemetry from **all** providers; train org GGUF/adapters with forgetting mitigations; re-bias the router toward cheap tiers as quality allows.

Inference for org models is a **single WASM binary** on WasmEdge: same artifact on CPU edge nodes and GPU nodes (`MODEL_PATH`, `N_GPU_LAYERS`, `CONTEXT_SIZE`). GGUF weights are **mmap’d by WASI-NN**—not stuffed into the 4 GiB WASM linear heap—so large models can run while the sandbox stays small.

```
User / IDE / CLI
        ↓
Agent Harness (tools, memory, ReAct loop)
        ↓
Router + Redaction Gateway
  credit ladder · all providers · capture 100%
        ↓
  Enterprise teachers          Org wasmai (WasmEdge)
  high credit burn             CPU 3B / GPU 7B–70B Q4
        └──────── harvest ────────┘
                    ↓
         Continual FT → promote GGUF
                    ↓
         more traffic on cheap tiers
```

Full phased plan, invariants, and ship gates: **[TODO.md](./TODO.md)**.  
Research notes: **[docs/](./docs/)**.  
Agent project memory: **[AGENTS.md](./AGENTS.md)**.

---

## Features (target)

### Agent
- ReAct tool loop: Read, Glob, Grep, Bash (sandboxed), Edit/Write, TodoWrite
- Project memory (`AGENTS.md` / `PROJECT.md`), context compaction, session resume
- Permissions: Auto / Ask / Plan · hooks · worktree isolation
- Skills, sub-agents, MCP (post-MVP)

### Providers & credits
- Claude, OpenAI, Kimi, and other enterprise models via one abstraction
- wasmai-cpu / wasmai-gpu as first-class providers
- Per-turn tokens + credits; budget caps; route reasons logged

### Inference (org models)
- Rust + Candle / WASI-NN · `wasm32-wasi` · WasmEdge GGML plugin
- Binary GPU offload (`N_GPU_LAYERS=0` or `99`) · `try_load_gpu` CPU fallback
- Confidence scores for escalation

### Learning
- Capture from **every** gateway session (every provider)
- Gateway PII/secret redaction before the training lake
- Implicit preferences + FIM hard negatives (Delulu-style)
- SFT/KL bootstrap; DPO/ORPO + O-LoRA / CURLoRA (etc.) on schedule
- Eval gates (general coding + FIM + corporate + credit simulation) before promote

---

## Architecture (crates, planned)

| Crate / area | Role |
|--------------|------|
| `wasmai-agent` | CLI harness, tools, sessions |
| `wasmai-providers` | Claude / OpenAI / Kimi / wasmai clients |
| `wasmai-router` | Credit ladder, complexity, confidence |
| `wasmai-gateway` | Redaction, meters |
| `wasmai-telemetry` | Versioned events (provider, tokens, credits) |
| `wasmai-infer` | WASM inference binary |
| `training/` | Distillation + continual FT pipelines |
| `deploy/` | Docker, K8s / wasmCloud, WasmEdge setup |

Current repo: single Hello World binary in `src/main.rs` — implementation tracks `TODO.md`.

---

## Quickstart (today)

```bash
# Requires Rust ≥ 1.80 (edition 2024 package)
cargo run
```

### Target local org-model path (when built)

```bash
# Host: WasmEdge with WASI-NN GGML plugin
export MODEL_PATH=/models/org-small-q4_k_m.gguf
export N_GPU_LAYERS=0          # or 99 on GPU hosts
export CONTEXT_SIZE=4096
wasmedge --dir .:. wasmai-infer.wasm
```

Enterprise providers: configure API keys via env / secret store (never commit). See `TODO.md` Phase 0–2.

---

## Invariants (non-negotiable)

- WASM heap ≤ **4.0 GiB** (target peak ≤ 3.8 GiB); GGUF via **mmap**, not guest heap.
- GPU offload **0 or 99** only (no PCIe partial-offload zone).
- **No raw sessions** in the training lake without gateway redaction.
- Router defaults to **cheap**; heavy APIs are escalate / teach, not “always on.”
- Capture **all** providers into one org corpus; train **org** models; re-bias traffic after promote.

---

## Roadmap (summary)

| Phase | Outcome |
|-------|---------|
| 0 | Repo foundation, CI, secrets scan |
| 1 | WasmEdge memory-separation proof + dual-mode infer |
| 2 | Agent harness + multi-provider capture |
| 3A | Redaction gateway + corpus store |
| 3B | Bootstrap org-small / org-large GGUF |
| 3C | Continual FT from live multi-provider telemetry |
| 4–5 | wasmai mesh + credit-aware ladder |
| 6–8 | Profiling, multi-tenant adapters, hybrid cluster + TCO |
| 9–10 | Skills/MCP, hardening, launch |

Details and checkboxes: [TODO.md](./TODO.md).

---

## Success metrics (selected)

- Org-small ≥ **80%** of enterprise teacher on simple tasks  
- ≤ **20%** escalate off org-small  
- ≥ **60%** cost reduction vs always-heavy-API baseline  
- Heavy-API share **trends down** after each successful model promote  
- WASM heap ≤ **3.8 GiB**; CPU p95 (3B, 250 tok) ≤ **4.5 s**; GPU p95 (8B) ≤ **2.0 s**

---

## Security & compliance

- Gateway redaction (Presidio-class + secret scanners) on inference **and** harvest paths  
- Schema-safe session storage (anti chat-history poisoning)  
- Secrets only via platform secret stores; CI Gitleaks/TruffleHog  
- Respect enterprise API ToS and employee notice for training-on-usage  

---

## Contributing

Implementation should follow **[TODO.md](./TODO.md)** phase order and **[AGENTS.md](./AGENTS.md)** conventions. Prefer small, reviewable PRs with tests for config, routing policy, and redaction fixtures.

---

## License

TBD — add a root `LICENSE` before public release (see TODO Phase 0).

---

## Name

**wasmai** — *WASM AI*: org coding intelligence that runs in a WasmEdge sandbox and learns from the enterprise models you already use.
