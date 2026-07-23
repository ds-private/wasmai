# AGENTS.md — wasmai project memory

High-signal context for humans and coding agents. Keep this short; update when architecture or phase status changes.

## One-liner

Credit-aware coding agent + multi-provider session capture → continual training of **org-owned** WASM (WasmEdge) models so simple work stops burning Claude/OpenAI/Kimi credits.

## Product shape (do not violate)

1. **Agent harness is native** (Rust preferred). Do **not** put the full tool loop inside the 4 GiB WASM instance.
2. **WASM = inference only** (`wasmai-infer`): GGUF via WASI-NN **mmap**; heap holds KV/metadata only.
3. **Every session through the gateway is captured** (all providers), redacted, and eligible for org-model training.
4. **Credit ladder (cheap first):** wasmai-cpu → wasmai-gpu → mid API → heavy enterprise API.
5. **Teachers = enterprise APIs** (and open teachers if used). **Students = org GGUF/adapters.**

## Stack

| Layer | Choice |
|-------|--------|
| Agent | Rust, ReAct tools, sessions, permissions |
| Providers | Claude, OpenAI, Kimi, … + wasmai |
| Infer | Rust, Candle/WASI-NN, WasmEdge, Q4_K_M GGUF |
| Org-small | Qwen2.5-Coder-3B class |
| Org-large | 7B-class coder (escalate) |
| Train | Offline GPU; SFT/KL bootstrap; DPO/ORPO + CL PEFT (O-LoRA/CURLoRA/EWC…) |
| Orchestration | Gateway redaction; later Kubeflow-style FT DAG; wasmCloud or K8s for serve |

## Invariants

- WASM linear memory ≤ 4.0 GiB (peak target ≤ 3.8 GiB).
- `N_GPU_LAYERS` is **0 or 99** only.
- Env config for infer: `MODEL_PATH`, `N_GPU_LAYERS`, `CONTEXT_SIZE`.
- No raw secrets/PII in training lake; redaction at gateway.
- Standard LoRA alone is **not** enough for scheduled continual FT.
- Prefer org-large over heavy API when escalating (save credits).

## Repo map

| Path | Purpose |
|------|---------|
| `TODO.md` | Full whiteboard → production checklist (source of truth for roadmap) |
| `README.md` | Public product overview |
| `AGENTS.md` | This file — agent/project memory |
| `docs/*` | Research prompts and reports (not implementation contracts) |
| `src/main.rs` | Temporary Hello World; real code will move to workspace crates |
| `Cargo.toml` | Package scaffold; deps empty until Phase 1+ |

## Implementation order (do not skip gates)

1. Phase 0 foundation + Phase 1 memory-separation experiment  
2. Phase 2 agent loop + multi-provider + telemetry schema  
3. Phase 3A redaction before any train-on-prod-logs  
4. Phase 3B bootstrap org GGUF; 3C continual from live capture  
5. Phase 4–5 mesh + credit-aware router  
6. Cluster / TCO / launch later  

## Coding conventions

- Prefer small crates matching planned monorepo names in `TODO.md`.
- Fail closed on tool schemas and session JSON (poisoning resistance).
- Log `provider`, `model_id`, tokens, credits, `route_reason` on every model turn when wiring telemetry.
- No API keys or `.gguf` weights in git; expand `.gitignore` early.
- Tests: config parse, GPU fallback state machine, redaction fixtures, routing ladder.

## Current status

- **Whiteboard / research done**; **implementation not started** beyond Cargo Hello World.
- Next concrete work: Phase 0 scaffold + Phase 1 WasmEdge heap vs host mmap proof.

## Anti-goals

- Full agent inside WASM  
- Always default to Opus/GPT/Kimi for simple tasks  
- Capture only one vendor’s sessions  
- Train models and never re-bias router toward cheaper tiers  
- Partial GPU offload (PCIe penalty zone)  

## References

- Roadmap: `TODO.md`  
- Architecture research: `docs/meta.md`, `docs/gemini.md`, `docs/grok-code.md`  
- Continual learning / redaction / DPO-FIM: integrated into `TODO.md` Phase 3A–3C  
