# wasmai — Whiteboard → Production TODO

> Consolidated from `docs/*` (deepseek, gemini, grok, meta + code variants)  
> **plus** *Strategic Engineering of Continual Learning Pipelines for Enterprise Coding Models* (telemetry → redaction → DPO/FIM → CL mitigations → LLMOps).  
> Status: **whiteboard / research complete**; implementation not started.  
> Last synthesized: 2026-07-23

## Product north star

Ship a **Claude Code–class coding agent** that can call **any enterprise LLM the org already pays for** (Claude, OpenAI, Kimi, Gemini, DeepSeek, …) **and** org-owned **wasmai** students, with a **closed-loop continual learning pipeline**: capture **every** session → redact → train **org models** that absorb team style so **more traffic shifts to cheaper tiers over time**.

### Credit / cost philosophy (first-class product constraint)

| Tier | Examples | Credits / cost | Default use |
|------|----------|----------------|-------------|
| **Heavy teacher** | Claude Opus/Sonnet, GPT-4/5-class, Kimi large, etc. | **Burn credits fast** | Complex multi-file reasoning, hard debug, teacher labels, hard-negative synthesis |
| **Mid commercial** | Smaller API models the org licenses | Medium | Medium tasks when wasmai not ready yet |
| **Org small (wasmai CPU)** | Qwen2.5-Coder-3B Q4, self-hosted | **Cheapest** (self-host or tiny credits) | High-volume simple completions — default path once good enough |
| **Org large (wasmai GPU)** | 7B–70B Q4 self-hosted | Medium infra cost, **no API credits** | Escalation when small org model is low-confidence |

- Prefer **small org models** for volume; escalate only when complexity/confidence demands it.
- **Every session is captured** regardless of which provider answered — enterprise APIs are **teachers + telemetry sources**, not permanent exclusive runtimes.
- Goal: **shrink share of heavy-API spend** while quality on corporate vernacular holds or improves.

### Runtime (inference)

1. **Multi-provider gateway** — one harness, many backends (enterprise APIs + wasmai).
2. **Economy path** — org small models (≤3B Q4_K_M) for high volume / low credit burn.
3. **Premium path** — org large models (8B–70B) and/or heavy enterprise APIs for hard tasks.
4. **≤4.0 GiB WASM** per wasmai instance (GGUF **mmap’d by WASI-NN**).
5. **Credit-aware routing** + complexity + confidence escalation (cheap → expensive, not the reverse by default).

### Learning loop (continual fine-tuning)

6. **Harvest from all sessions** — every provider (Claude, OpenAI, Kimi, wasmai, …), ReAct/tools/sub-agents/MCP/todos, IDE/FIM.
7. **Redact** PII/secrets/IP at the **gateway** before any training store.
8. **Structure** teacher traces + implicit preference pairs + FIM hard negatives.
9. **Schedule** FT with **CL mitigations** so proprietary style lands in **org models**.
10. **Deploy** new adapters/GGUF → fleet; **re-bias router** toward cheaper org tiers as evals allow.

**Do not put the full agent loop inside the 4 GB WASM instance.**  
Layers:

```
User / IDE / CLI
        ↓
Agent Harness (native Rust preferred)
  · tool registry + ReAct event loop
  · project memory + context compaction
  · session telemetry (provider, model id, tokens, $ / credits)
        ↓
Router + Redaction Gateway  ← ALL traffic (every provider)
  · credit-aware: default cheap tier; escalate only if needed
  · providers: Claude | OpenAI | Kimi | … | wasmai-cpu | wasmai-gpu
  · PII/secret redaction on I/O + training harvest
        ↓
  ┌─────────────────────┬──────────────────────────┐
  Enterprise API teachers    WasmEdge org models
  (high credit burn)         (MODEL_PATH, N_GPU_LAYERS)
  └─────────────────────┴──────────────────────────┘
              │ all sessions (redacted)
              ▼
Continual Learning Pipeline (offline, scheduled)
  · multi-provider corpus → DPO/ORPO + FIM hard negatives
  · distill enterprise teacher behavior into org GGUF/adapters
  · eval gates → promote → lower default API share
```

---

## Current baseline (repo reality)

| Item | State |
|------|--------|
| Cargo package `wasmai` 0.1.0 (edition 2024) | ✅ Hello World |
| Dependencies (candle, wasi-nn, …) | ❌ empty |
| README / LICENSE / CI / toolchain files | ❌ missing or minimal |
| Research docs under `docs/` | ✅ present |
| Inference binary / agent harness / models | ❌ none |
| Telemetry harvest / redaction gateway / CL pipeline | ❌ none |
| Multi-provider client + credit accounting | ❌ none |

---

## Invariants (non-negotiable)

### Inference

- [ ] WASM linear memory ≤ **4.0 GiB** (hard fail / sliding window before OOM).
- [ ] Peak heap target ≤ **3.8 GiB** (200 MiB safety buffer).
- [ ] GGUF weights **mmap’d via WASI-NN** — model files may exceed 4 GB.
- [ ] Quantization: **Q4_K_M** default; optional Q5_K_M / Q8_0 on GPU if VRAM allows.
- [ ] GPU offload is **binary**: `N_GPU_LAYERS=0` **or** `99` — never partial (PCIe penalty zone).
- [ ] Same binary on CPU and GPU; config only via env: `MODEL_PATH`, `N_GPU_LAYERS`, `CONTEXT_SIZE`.
- [ ] CUDA load failure → `try_load_gpu()` falls back to CPU without killing the instance.
- [ ] Domain v1: Python + JavaScript code completion, bug-fix, refactor (FIM + agentic).

### Multi-provider & credits

- [ ] Harness talks to a **single provider abstraction** — Anthropic, OpenAI, Moonshot/Kimi, others, and wasmai — without rewriting tools/prompts.
- [ ] **Capture 100% of sessions** that go through the gateway (every provider, every model id), subject to legal/opt-in policy.
- [ ] Record per turn: `provider`, `model_id`, **input/output tokens**, **estimated credits / $**, latency, route reason.
- [ ] Router is **credit-aware by default**: try org-small / cheap tier first; escalate to mid/heavy only on complexity, low confidence, or explicit user/mode override.
- [ ] Heavy enterprise models are **teachers and safety nets**, not the permanent default for 80% simple workload.
- [ ] Org models are the **training targets** of the continual pipeline; API traffic is the **corpus source**.

### Data & learning security

- [ ] **No raw chat/session logs** enter training storage without gateway redaction.
- [ ] Secrets/PII/IP redacted or rejected at **proxy layer** (not only at training time).
- [ ] Session persistence uses **strict JSON schemas** + untrusted-context escaping (block chat-history poisoning).
- [ ] Training data residency: redaction + placeholder maps stay inside enterprise VPC.
- [ ] Scheduled fine-tune only when **prompting + RAG exhausted** for the capability *and* **≥ ~500** high-quality preference examples with a **measurable eval metric** proving lift (scale up to 25k–100k as corpus grows).
- [ ] Continual FT must use a **CL-aware method** (O-LoRA / CURLoRA / EWC / FIP / function-vector regularization) — **standard LoRA alone is insufficient** against catastrophic/spurious forgetting.
- [ ] Every scheduled train job must pass **regression gates** (HumanEval/MBPP + Delulu-style FIM + corporate suite) before fleet promotion.

### Target models (org-owned students + external teachers)

| Tier | Model / provider | Role | Credits |
|------|------------------|------|---------|
| **Enterprise teachers** | Claude, OpenAI, Kimi, … (org contracts) | Live heavy path + distillation source | High burn — minimize share over time |
| CPU / Student-Small | **Qwen2.5-Coder-3B** Q4 | Default org volume path | Lowest (self-host) |
| GPU / Student-Large | Qwen2.5-Coder-7B-class Q4 | Org escalate / FIM DPO class | Infra only (no API credits) |
| Optional large org | up to 70B Q4 | Highest self-hosted accuracy | Infra-heavy |
| Avoid as primary | Stable-Code-3B | Weak HumanEval | Research only |
| Ultra-edge only | DeepSeek-Coder-1.3B | <2 GB devices | Lowest accuracy |

### Success metrics (ship gates)

| Metric | Target |
|--------|--------|
| Small org model correctness | ≥ 80% of **enterprise teacher** on simple tasks |
| Large org model correctness | ≥ 90% of teacher on complex tasks |
| Peak WASM heap | ≤ 3.8 GiB all modes |
| CPU p95 (3B, 250 tokens) | ≤ 4.5 s |
| GPU p95 (8B, 250 tokens) | ≤ 2.0 s |
| VRAM 8B full offload | ≤ 6.5 GiB |
| VRAM 70B full offload | ≤ 42 GiB |
| Cold start (mmap + WASI-NN) | ≤ 10 s |
| Router decision | < 10 ms |
| Escalation rate (within org tiers) | ≤ 20% of small-model requests escalate |
| Hybrid TCO vs GPU-only / API-only | ≥ 60% cost reduction vs all-heavy-API baseline |
| **Heavy-API share of requests** | Trend down after each successful train promote (track %) |
| **Credits per 1M tokens** (blended) | Dashboard + quarterly reduction target |
| Adapter logit shift (static LoRA merge) | < 0.5% cosine deviation |
| Workload mix assumption | ~80% simple / 20% complex |
| **Gateway redaction overhead** | Keep low-latency (research ~11 µs/req class at proxy; measure in our stack) |
| **Preference FT floor** | ≥ 500 high-quality chosen/rejected rows before first schedule |
| **FIM hallucination lift** | Track Delulu-style EM / edit similarity (research: ~+18.8 EM / +0.22 edit-sim on 100k rows for 7B — treat as aspirational, not v1 gate) |
| **Catastrophic forgetting** | Base-task regression within agreed band; no ship if general coding collapses |
| **Session capture coverage** | ≥ 99% of gateway turns logged (redacted) with provider + model id |
| **Poisoning resistance** | Adversarial history fixtures never train; eval with trigger patterns |

---

## Phase 0 — Repo & engineering foundation

**Goal:** Project is buildable, documented, and CI-ready before research code lands.

- [ ] Expand root `.gitignore` (Rust/Cargo, models `*.gguf`, `.env`, secrets, `target/`, telemetry dumps)
- [ ] Add `README.md` (vision, three-layer architecture, quickstart, status)
- [ ] Add `LICENSE` (choose and document)
- [ ] Add `rust-toolchain.toml` (Rust ≥ 1.80, wasm32-wasi / wasip1 targets as needed)
- [ ] Add `.editorconfig` + `rustfmt.toml` / `clippy` lints in CI
- [ ] Add GitHub Actions: `cargo fmt --check`, `clippy`, `test`, wasm build smoke
- [ ] Secrets scanning in CI: **Gitleaks** and/or **TruffleHog** / detect-secrets
- [ ] Optional SAST: Semgrep / CodeQL; Dependabot for SCA
- [ ] Add `SECURITY.md` (vuln reporting; no secrets in tree; training-data policy pointer)
- [ ] Document required host tools: WasmEdge, WASI-NN GGML plugin, optional CUDA plugin
- [ ] Decide monorepo layout, e.g.:
  - `crates/wasmai-infer` — WASM inference binary (org models)
  - `crates/wasmai-agent` — native agent harness
  - `crates/wasmai-router` — credit-aware routing / escalation
  - `crates/wasmai-providers` — Claude / OpenAI / Kimi / … / wasmai clients
  - `crates/wasmai-tools` — shared tool schemas
  - `crates/wasmai-telemetry` — session/event schemas + exporters (all providers)
  - `crates/wasmai-gateway` — redaction + optional billing/credit meters
  - `training/` — SFT / DPO / CL jobs (Python +/or Rust)
  - `training/pipelines/` — Kubeflow (or equiv.) DAG definitions
  - `deploy/` — Docker / K8s / wasmCloud / Terraform
  - `bench/` — profiling + Delulu/FIM + forgetting + **credit burn** evals
- [ ] Pin workspace `Cargo.toml`; empty deps → planned crate graph only for now
- [ ] Document org-configured provider list (env/secrets for each enterprise API key)

---

## Phase 1 — Prove the inference foundation (gate for everything else)

**Goal:** Validate heap vs host-memory separation and a minimal dual-mode load path.  
**Parallel with Phase 2 (agent loop can use cloud APIs first).**

### 1.1 Host / runtime setup

- [ ] Install Rust 1.80+ toolchain + `wasm32-wasi` (or current wasip1 target name)
- [ ] Build/install WasmEdge with `WASMEDGE_PLUGIN_WASI_NN_BACKEND=GGML`
- [ ] Verify `wasmedge --version` reports GGML backend
- [ ] Optional: CUDA WASI-NN plugin (`wasmedge-plugin-wasi_nn-cuda`) on a GPU host
- [ ] Document install script in `deploy/scripts/setup-wasmedge.sh`

### 1.2 Memory-separation experiment (non-negotiable)

- [ ] Obtain a **7–8B Q4_K_M GGUF** (~4–5 GB file)
- [ ] Run under WasmEdge on a machine with ≥ 8 GB RAM
- [ ] Measure WASM heap via `wasmedge --enable-time-measuring` (or runtime stats)
- [ ] Measure host RAM / VRAM (`/proc`, `nvidia-smi`)
- [ ] **Pass criteria:** WASM heap stays **≪ 4 GB** (target &lt; 1 GB for metadata path); host absorbs model weight footprint
- [ ] Record results in `bench/memory-separation.md`

### 1.3 Minimal Rust inference shim

- [ ] Scaffold `wasmai-infer` binary crate
- [ ] Config struct from env: `MODEL_PATH`, `N_GPU_LAYERS`, `CONTEXT_SIZE`
- [ ] WASI-NN load path: GGUF via plugin (`load` / `load_by_name_with_config` + JSON config)
- [ ] Implement `try_load_gpu()`:
  - attempt GPU / `N_GPU_LAYERS=99`
  - on `WASINN_ERR` / backend error → log, force `N_GPU_LAYERS=0`, rebind CPU
- [ ] Enforce binary offload (reject or clamp partial layer counts)
- [ ] Single-prompt CLI: stdin or argv → generate tokens → stdout
- [ ] Export **confidence**: top-1 softmax prob and/or sequence entropy from logits
- [ ] Unit tests for config parsing and fallback state machine (mock errors if needed)

### 1.4 Packaging smoke

- [ ] Cross-compile to wasm target
- [ ] Docker image: WasmEdge + plugin + sample env
- [ ] OCI artifact layout plan: `.wasm` binary separate from `.gguf` / LoRA artifacts
- [ ] Smoke run: one completion on CPU path

**Exit criteria:** Documented memory experiment green + one working CPU inference path + GPU fallback code path tested or simulated.

---

## Phase 2 — Agent harness (Claude Code primitive)

**Goal:** Usable coding agent with cloud/strong model first; swap backend later.  
**Instrument every turn so telemetry is a first-class product surface.**  
**Language preference (docs consensus):** native **Rust** for harness; Python OK for early prototype.

### 2.1 Core loop (ReAct)

- [ ] Conversation / message history (singleton conversation manager pattern)
- [ ] Think–Act–Observe cycle; record reasoning, tool calls, observations
- [ ] **Multi-provider LLM client** (not a single-vendor lock-in):
  - Anthropic (Claude), OpenAI, Kimi/Moonshot, others as configured
  - wasmai local/OpenAI-compatible endpoint as another provider
  - Unified tool-calling + streaming adapters per provider quirks
- [ ] Streaming responses + cancellation
- [ ] Max iterations / timeouts (no infinite tool loops)
- [ ] Parallel tool calls where model supports them
- [ ] Terminal states: completed, max_turns, error, aborted, …
- [ ] Per-turn **credit/token meter** (provider usage APIs or token estimates × price table)

### 2.2 Essential tools (ship order)

| Priority | Tool | Notes |
|----------|------|--------|
| P0 | `Read` | file or range |
| P0 | `Glob` / `ListDir` | discovery |
| P0 | `Grep` | prefer ripgrep |
| P0 | `Bash` | sandboxed + allow/deny lists |
| P0 | `Edit` / `Write` | exact-string replace preferred |
| P0 | `TodoWrite` | multi-step task lifecycle (pending → in_progress → completed) |
| P1 | Schema validation | JSON Schema / typed args; fail-closed factory |
| P1 | Audit log | every tool invocation (feeds telemetry) |

### 2.3 System prompt & workflow

- [ ] Role + workflow (explore → plan → todo → edit → test → summarize)
- [ ] Tool-usage rules (“run tests after edits”)
- [ ] Project standards injection from memory files

### 2.4 Project memory (files, not DB)

- [ ] Auto-load `AGENTS.md` / `PROJECT.md` / equivalent at session start
- [ ] Keep high-signal and short (architecture, conventions, plan)
- [ ] Filesystem as primary memory; context window secondary
- [ ] Optional later: `MEMORY.md` index + topic files (≤200 lines index)

### 2.5 Context management

- [ ] Token usage tracking
- [ ] Top/bottom message cropping (preserve system instructions)
- [ ] Sliding window (e.g. last N turns) as v0
- [ ] Micro-compaction: drop bulky tool results early
- [ ] Full summarization near limit
- [ ] After compact: re-inject project memory + active todos
- [ ] Later: multi-layer order (tool-result budget → snip → microcompact → collapse → auto-compact)

### 2.6 Sessions & state (schema-safe)

- [ ] Persist: messages, todos, modified files/diffs, cwd, env snapshot
- [ ] SQLite or JSON session store with **strict schema** (no free-form untrusted blobs)
- [ ] Sanitize / escape client-controlled context (anti **chat-history poisoning**)
- [ ] Resume path reconstructs agent state
- [ ] Export path for training harvest (Phase 3A) — opt-in, redacted

### 2.7 Permissions & security

- [ ] Modes: at least `Auto` / `Ask` / `Plan` (expand toward full mode set if needed)
- [ ] Rule allow/deny lists (paths, commands)
- [ ] Dangerous-command detection (`rm -rf /`, fork bombs, unrestricted sudo)
- [ ] Confirm high-risk Bash / large writes with clear diffs
- [ ] Optional worktree isolation for risky sessions
- [ ] Hooks v0: PreToolUse (block/modify) + PostToolUse; freeze config at session start

### 2.8 Concurrency (agent tools)

- [ ] Partition concurrent-safe reads vs serial writes
- [ ] Yield tool results in submission order

### 2.9 Telemetry hooks — capture from **all** sessions

Every gateway session is training fuel for **org models**, whether the answer came from Claude, OpenAI, Kimi, or wasmai.

| Component | Capture for training / credits |
|-----------|--------------------------------|
| **Provider + model id** | Which teacher answered (required on every turn) |
| **Tokens + credits** | Input/output tokens, estimated $ / credit units |
| ReAct loop state | Intermediate reasoning + iterative debug paths |
| Conversation manager | Multi-turn boundaries, system vs user vs tool roles |
| Sub-agent manager | Hierarchical delegation graph, isolated contexts |
| MCP tools | External tool I/O (redact payloads aggressively) |
| TodoWrite lifecycle | Temporal QA gates (pending / in_progress / completed) |
| Route decision | Why cheap vs heavy was chosen (complexity, confidence, override) |
| Implicit IDE signals (later) | Accept / edit / reject / backspace on completions |

- [ ] Define `TelemetryEvent` schema (versioned) including `provider`, `model_id`, `token_in/out`, `credit_cost`
- [ ] Correlation IDs: session → turn → tool_use → sub-agent
- [ ] **No provider-private silos**: one corpus stream for the org
- [ ] Never log raw secrets; redact at write time or only write via gateway
- [ ] Legal/policy: enterprise ToS + employee notice for training-on-usage where required

**Exit criteria:** Local agent can complete a multi-step task via **at least two enterprise providers** + optional wasmai stub; each session export is schema-valid, redacted, and includes provider + credit fields.

---

## Phase 3A — Gateway redaction & secure corpus store

**Goal:** Make it **impossible** for unfiltered prompts/history to become training data.  
**Must land before any automated fine-tune on production telemetry.**

### 3.A.1 Threat model (track explicitly)

- [ ] Secret memorization (API keys, DB creds, proprietary algorithms)
- [ ] **Chat history poisoning** → persistent prompt injection via trained triggers
- [ ] Cross-border / residency violations (GDPR-class personal data)
- [ ] Federated-style update poisoning (if multi-tenant later)

### 3.A.2 Gateway redaction layer

- [ ] Proxy (Bifrost-class or custom) on all LLM provider + wasmai traffic
- [ ] Detect + rewrite PII/secrets to placeholders **before** storage or third-party providers
- [ ] **Presidio** (or equiv.) for NLP + pattern PII on text and structured JSON
- [ ] Entropy/heuristic scanners: TruffleHog / detect-secrets on harvest path
- [ ] Keep detection + placeholder maps inside VPC
- [ ] Benchmark gateway latency under load; document overhead

### 3.A.3 Corpus store

- [ ] Append-only, access-controlled training lake (encrypted at rest)
- [ ] Retention + residency policies documented
- [ ] Poisoning filters: reject schema violations, suspicious control tokens, known injection patterns
- [ ] Dual path: live inference redaction **and** batch re-scan before train

### 3.A.4 DevSecOps feedback loop

- [ ] Pre-commit + CI secrets scan (same tools as Phase 0)
- [ ] IDE/CI SAST (Semgrep/CodeQL) + SCA (Dependabot/Snyk) so vuln patterns are not trained as “normal”

**Exit criteria:** Red-team fixture (secrets + injection payload in history) never lands in training store; clean synthetic logs do.

---

## Phase 3B — Offline distillation (bootstrap org students)

**Goal:** First domain-specialized GGUF **org models** for CPU and GPU pools.  
Teachers = **org’s enterprise models** (Claude, OpenAI, Kimi, …) and/or open teachers when contracts allow.  
Runs on GPU Linux hosts **outside** WASM.

### 3.B.1 Synthetic / repo-based data curation

- [ ] Ingest target repos; strip PII/secrets (same scanners as 3A)
- [ ] **Teacher panel** from configured enterprise providers (not hard-coded to one vendor):
  - Primary: org’s strongest coding models (e.g. Claude, GPT-class, Kimi)
  - Optional open teachers: DeepSeek-V3.x / Qwen3-Coder-class if preferred for cost
- [ ] Generate instruction pairs: refactoring, docs, unit tests, bug localization
- [ ] Target volume: **~25k–30k** high-quality pairs (floor ~20k; up to 50k–100k pre-filter)
- [ ] Filter: AST parse / linters / executor-verified where possible
- [ ] MinHash near-duplicate removal
- [ ] Preserve general ability: mix ~10–20% non-corporate code/text if overfitting appears
- [ ] Tag each row with `teacher_provider` / `teacher_model` for lineage

### 3.B.2 Training recipe (bootstrap)

**Sub-3B (org-small — credit saver):**

- [ ] Prefer **SFT** and/or **white-box logits (KL) distillation** for first specialization
- [ ] Avoid **pure DPO on Sub-3B** until preference set is large and stable (representation-collapse risk from earlier research)
- [ ] Success = enough quality that router can **default** simple traffic here and stop burning enterprise credits

**7B-class (org-large — escalate / no API credits):**

- [ ] SFT bootstrap, then DPO/ORPO when preference pairs available
- [ ] QLoRA **r=8** preferred for WASM multi-tenant headroom (r=16 only if measured safe)

### 3.B.3 Export & eval

- [ ] Merge adapters offline for single-tenant edge; keep separate for multi-tenant hot-swap
- [ ] Export → **Q4_K_M GGUF**
- [ ] HumanEval / MBPP + corporate unit tests
- [ ] Adapter shift: cosine similarity of logits vs base &lt; 0.5%
- [ ] Publish `bench/model-report.md`

**Exit criteria:** Published GGUF artifacts for small (+ large if ready) meeting correctness gates on a fixed eval suite.

---

## Phase 3C — Preference datasets, FIM hard negatives & continual learning

**Goal:** Turn live agent + IDE telemetry into scheduled capability lift **without** catastrophic forgetting.  
Depends on Phases 2, 3A, and initial 3B models.

### 3.C.1 Implicit feedback mining (all providers)

From IDE completions and agent outcomes (no thumbs-up required).  
Corpus includes sessions that used **Claude, OpenAI, Kimi, wasmai, …** — teacher quality may vary; weight by provider strength if needed.

| Signal | Preference label |
|--------|------------------|
| Accept + leave unmodified / minor syntax edit before commit | **chosen** |
| Immediate delete / ignore / aggressive backspace | **rejected** |
| Agent: tests green + human keeps edits | **chosen** |
| Agent: human reverts / rewrites heavily | **rejected** |
| Heavy-API answer kept + small-model earlier attempt rejected | **chosen** = heavy, **rejected** = small (distill escalate path) |

- [ ] Emit keystroke/completion outcome events (IDE extension or CLI post-hoc)
- [ ] Join with redacted context windows → DPO/ORPO JSON (`prompt`, `chosen`, `rejected`, `source_provider`)
- [ ] Optional: distill **successful heavy-API trajectories** as SFT gold for org-large
- [ ] Quality filters: length, language, AST validity, near-dup
- [ ] Gate: **≥ 500** high-quality pairs + defined eval metric before first schedule

### 3.C.2 FIM-focused hard negatives (Delulu taxonomy)

Small local models hallucinate in mid-keystroke FIM; execution oracles do not scale at keystroke latency.

- [ ] Extract FIM contexts (prefix/suffix) from enterprise repos (and optional public code)
- [ ] Golden continuation = developer-accepted line → **chosen**
- [ ] Frontier model panel generates one **plausible-but-wrong** completion → **rejected**
- [ ] Cover Delulu failure modes:

| Type | Edit to golden | Training effect |
|------|----------------|-----------------|
| Method hallucination | Invented method name | Respect real class APIs |
| Parameter hallucination | Fake kwargs / positions | Respect signatures |
| Undefined variable | Identifier not in scope | Local binding fidelity |
| Import hallucination | Fictitious package/symbol | Real dependency graph |

- [ ] Format for DPO/ORPO; track EM + edit similarity on internal Delulu-style holdout
- [ ] Confirm general coding suites (HumanEval-Infilling / SAFIM or internal equivalents) do not regress

### 3.C.3 When to run scheduled fine-tunes

- [ ] Document policy: **only after** prompting + RAG exhausted for the gap
- [ ] Cadence: bi-weekly or monthly, or volume-triggered (N new pairs)
- [ ] Capability-lift metric required (not “train because calendar says so”)

### 3.C.4 Catastrophic forgetting mitigations (required stack)

Naive LoRA does **not** guarantee retention on rugged LLM loss landscapes (spurious forgetting).

Pick and implement at least one primary + one backup:

| Strategy | Mechanism | Why it matters |
|----------|-----------|----------------|
| **EWC** | Fisher Information + quadratic penalty on important weights | Protects prior knowledge without replaying old data (~45% CF reduction in cited work) |
| **LwF** | Distillation to teacher outputs on new data | Soft retention of prior behavior |
| **O-LoRA** | New adapters orthogonal to prior task gradient subspaces | Non-interference by construction; privacy-friendly (no replay) |
| **FIP** | Optimize along functionally invariant paths on a Riemann manifold | Dual-task states in non-Euclidean weight geometry |
| **CURLoRA** | CUR decomposition; inverted selection probs; zero-init low-rank core | Stable continual FT; freeze base perplexity empirically |
| **Function-vector guided** | Regularize task activation patterns | Surgical preservation without freezing whole blocks |
| **Low-perplexity token learning** | Mask high-perplexity tokens during FT | Reduce interference with established knowledge |

Recommended default for enterprise schedule (research synthesis):

- [ ] **Primary:** O-LoRA or **CURLoRA** for PEFT continual updates on constrained hardware
- [ ] **Add:** EWC or LwF term when multi-epoch sequential domains stack
- [ ] **Eval:** base perplexity / HumanEval holdout frozen within tolerance after each job
- [ ] **Avoid:** memory-replay of raw enterprise chat (privacy); if replay needed, use redacted synthetic only

### 3.C.5 LLMOps orchestration

- [ ] DAG (Kubeflow Pipelines or equivalent) nodes:
  1. Ingest multi-provider conversation/IDE JSON  
  2. Security scan (Presidio + TruffleHog containers)  
  3. Implicit pair builder + FIM hard-negative synthesis (frontier/enterprise teachers for negatives)  
  4. DPO/ORPO format + train org-small and/or org-large (CURLoRA/O-LoRA)  
  5. Eval gate (general + FIM + corporate + **credit-savings simulation**)  
  6. Quantize/export GGUF + adapter artifacts  
  7. Promote to staging registry → canary fleet  
  8. **Update router defaults** (raise share of traffic eligible for org-small if evals green)
- [ ] Fail closed: any security or eval gate red → no promote
- [ ] Model cards + dataset version + training config lineage + teacher-provider mix
- [ ] Rollback path to previous GGUF within minutes

**Exit criteria:** One successful scheduled run promotes an org model that improves FIM/preference metrics without HumanEval collapse **and** offline routing sim shows lower blended credits vs pre-promote baseline.

---

## Phase 4 — Wire harness to wasmai + multi-provider mesh

**Goal:** Same tools/prompts; router chooses among enterprise APIs and wasmai.

- [ ] HTTP or gRPC OpenAI-compatible API over WasmEdge (wasmai-cpu / wasmai-gpu)
- [ ] Register wasmai as a first-class provider next to Claude / OpenAI / Kimi
- [ ] Streaming token path for CLI UX on all providers
- [ ] Pass-through of confidence metrics from wasmai → router
- [ ] All traffic through redaction gateway (3A)
- [ ] Feature flags / org policy: allowlists of providers, max credit tier, force-local mode
- [ ] Integration tests: tool loop on wasmai-cpu; smoke on each configured enterprise provider

**Exit criteria:** End-to-end task works on wasmai-cpu; same harness completes a task on at least one enterprise API without code changes beyond config.

---

## Phase 5 — Credit-aware router, confidence, escalation

**Goal:** Minimize credits while holding quality — small models first, heavy models sparingly.

### 5.0 Routing policy (credit ladder)

Default ladder (cheapest first):

1. **wasmai-cpu** (org-small) — burn almost no enterprise credits  
2. **wasmai-gpu** (org-large) — infra cost only  
3. **Mid commercial API** (if licensed)  
4. **Heavy enterprise API** (Claude / OpenAI / Kimi top tier) — last resort / teacher

- [ ] Org config: price table (credits per 1K tokens per model) + monthly budget caps
- [ ] Hard caps: stop or queue heavy tier if budget exhausted (fail open to wasmai or fail closed per policy)
- [ ] User/mode overrides: `Plan`/`Ask` may force stronger model; `Auto` stays ladder-default
- [ ] Log `route_reason` for every decision (complexity | confidence | override | budget)

### 5.1 Complexity classifier (&lt; 10 ms)

- [ ] Score = f(token count, AST depth/node count, keyword weights)
  - keywords e.g. `refactor`, `debug`, `optimize` weighted higher; simple `complete` lower
- [ ] tree-sitter (Python/JS) in Rust sidecar optional but preferred
- [ ] Threshold maps to **ladder step**, not only CPU vs GPU
- [ ] Mock first (Flask/Python) then production Rust (Tokio/axum)

### 5.2 Confidence escalation

- [ ] If org-small confidence &lt; **0.6** → escalate one step up the ladder (org-large, then heavy API)
- [ ] Prefer **org-large before heavy API** when both available (save credits)
- [ ] Circuit breaker / max escalate rate / max heavy-API QPS
- [ ] Metrics: escalation %, accuracy lift, **credits saved vs always-heavy**, added latency
- [ ] Optional: small MLP on prompt embeddings for preemptive step-up

### 5.3 Ops metrics

- [ ] OpenTelemetry on harness + router + infer + training pipeline
- [ ] Dashboards: tps, p95, heap, VRAM, escalate rate, **credits/$ by provider**, train job health, forgetting metrics, **% traffic on org-small**

**Exit criteria:** Load test shows routing p95 &lt; 10 ms; ≤20% escalate from org-small; blended credits **materially below** always-heavy baseline at equal or better eval quality.

---

## Phase 6 — Profiling matrix & performance hardening

**Goal:** Evidence pack for production sizing.

### Scenarios

- [ ] A: CPU, 3B, `N_GPU_LAYERS=0`
- [ ] B: GPU full, 3B, `N_GPU_LAYERS=99`
- [ ] C: CPU, 8B (if host RAM allows)
- [ ] D: GPU full, 8B
- [ ] Extend: 1.3B, 7B, 70B; n_gpu_layers ∈ {0, 99} only for prod configs (bench may include 33/66 to **document** the penalty zone)

### Measure

- [ ] tokens/sec, p95 latency (250-token gen)
- [ ] WASM heap, host RAM, VRAM (`nvidia-smi`)
- [ ] Energy / watt-hours per query if available
- [ ] Cold start time
- [ ] KV cache growth vs context size; sliding window at 3.8 GiB

### Risk mitigations (implement)

- [ ] Sliding window / context shift when heap ≥ 3.8 GiB
- [ ] Binary offload enforcement
- [ ] Pre-warm models on nodes (avoid cold-start on hot path)

**Exit criteria:** Filled performance matrix + all metric rows green or documented exceptions.

---

## Phase 7 — Multi-tenancy & adapters (after base path works)

**Do not prioritize before Phases 1–4 are solid.**

- [ ] Hot-swap LoRA / O-LoRA adapters without reloading base
- [ ] LRU cache of 3–5 adapters in heap
- [ ] Multi-tenant adapter registry (company/domain → adapter path)
- [ ] Deterministic inference mode (seed, temp=0) for regression tests
- [ ] Prefer **pre-merged** GGUF for single-tenant edge (static memory)
- [ ] Per-tenant data isolation in telemetry lake and train jobs

---

## Phase 8 — Cluster & production deployment

**Goal:** Hybrid CPU+GPU fleet with reference IaC.

### 8.1 Node pools

- [ ] **Pool-CPU:** e.g. t3.xlarge-class (4 vCPU, 16 GB RAM), SIMD
- [ ] **Pool-GPU:** e.g. g4dn / L4 class (≥8 GB VRAM for 8B; scale for 70B)
- [ ] Labels: `pool=cpu` / `pool=gpu` (wasmCloud host labels or K8s node selectors)

### 8.2 Orchestration options (pick one primary)

- [ ] **A:** wasmCloud lattice + NATS (aligns with inference research)
- [ ] **B:** Kubernetes + WasmEdge runtime class / DaemonSet plugins
- [ ] Sidecar router on each path or central Envoy/L7
- [ ] Training cluster: Kubeflow (or equivalent) on GPU nodes, isolated from serving if needed

### 8.3 Reference architecture deliverables

- [ ] Terraform / Crossplane (or documented Kustomize) for dual pools + train pipeline
- [ ] Docker/OCI publish pipeline for wasm + models + redaction gateway
- [ ] Secrets via platform secret store only
- [ ] Branch protection, Dependabot, secret scanning (GitHub)
- [ ] Horizontal scaling + health checks + circuit breaker on escalate path

### 8.4 TCO & credit validation

- [ ] Workload: 80% simple / 20% complex
- [ ] Cost per 1M tokens: always-heavy-API vs hybrid ladder vs wasmai-only
- [ ] Include train job cost amortized per 1M inference tokens
- [ ] Spreadsheet + cloud billing export + **enterprise credit invoices**
- [ ] Gate: ≥ 60% savings vs always-heavy-API (or GPU-only) at ≥ 95% of large/teacher quality on simple tasks
- [ ] Track monthly **credit burn by provider** and **org-small traffic share**

**Exit criteria:** Staging cluster serves hybrid traffic; TCO and SLOs met for a defined load.

---

## Phase 9 — Advanced agent features (post-MVP)

- [ ] **Skills:** two-phase load (`SKILL.md` frontmatter at start, body on use)
- [ ] **Sub-agents:** Explore (read-only), Plan (no edits), specialized workers — telemetry includes hierarchy edges
- [ ] **MCP:** external tools (GitHub, Jira, Slack, …); never shell from MCP skill body; redact MCP payloads
- [ ] Prompt-cache architecture: static system prefix vs dynamic project suffix
- [ ] Multi-provider client factory (if still bridging commercial APIs)
- [ ] Optional Candle upstream patch: expose VRAM / plugin capability metrics via WASI-NN
- [ ] IDE extension path for FIM + implicit accept/reject telemetry

---

## Phase 10 — Production hardening & launch

- [ ] Load / soak tests (memory leaks, fragmentation under long sessions)
- [ ] Chaos: kill GPU plugin, OOM VRAM, network partition → CPU fallback holds
- [ ] Security review: sandbox, path traversal, SSRF via tools, secret exfil, **training poisoning**
- [ ] Red-team: plant injection in session history; assert not in train corpus / not in promoted model
- [ ] Runbooks: deploy, rollback, model swap, escalate storms, **train-job failure / eval gate fail**
- [ ] Public/private docs: architecture, operators, contributors, **data retention & redaction**
- [ ] Versioned model + binary release process (semver + model card + dataset lineage)
- [ ] Legal: model licenses, training data policy, customer data isolation, GDPR-class controls
- [ ] Developer productivity / cognitive-debt framing: ECVM-style human oversight expectations documented
- [ ] Production launch checklist sign-off against success metrics table

---

## Research questions → engineering tickets map

| RQ / theme | Outcome ticket |
|------------|----------------|
| RQ1 Model selection | Phase 3B eval + lock Qwen2.5-Coder-3B for CPU |
| RQ2 LoRA stability | Phase 7; r=8 default; CL methods in 3C |
| RQ3 Distillation size / method | Phase 3B (bootstrap) + 3C (continual) |
| RQ4 Heap under offload | Phase 1 experiment + Phase 6 matrix |
| RQ5 Perf / PCIe penalty | Phase 6; binary offload only |
| RQ6 Graceful GPU fail | Phase 1 `try_load_gpu` |
| RQ7 Hybrid routing / TCO | Phases 5 + 8 |
| Multi-provider enterprise teachers | Phases 2.1, 4, 5.0 |
| Capture all sessions → train org models | Phases 2.9, 3A, 3C |
| Credit ladder (small cheap / heavy expensive) | Phase 5.0–5.2 |
| Agentic telemetry corpus | Phase 2.9 + 3C.1 |
| Gateway PII / poisoning | Phase 3A |
| FIM hard negatives (Delulu) | Phase 3C.2 |
| Catastrophic forgetting | Phase 3C.4 |
| Scheduled LLMOps | Phase 3C.5 |

---

## Deliverables checklist

### Inference / agent (original research)

- [ ] Benchmark report (3B candidates under CPU; 7B-class under GPU)
- [ ] Rust crate: dual-mode infer + GPU auto-detect (+ later hot-swap adapters)
- [ ] Performance matrix (sizes × offload modes)
- [ ] TCO spreadsheet (CPU / GPU / hybrid + train amortization)
- [ ] Routing engine prototype (Rust)
- [ ] Reference architecture (IaC + wasmCloud/K8s)
- [ ] Optional Candle/WASI-NN metrics patch

### Continual learning + multi-provider (new)

- [ ] Versioned **TelemetryEvent** + session export schema (`provider`, tokens, credits)
- [ ] Multi-provider client pack (Claude, OpenAI, Kimi, … + wasmai)
- [ ] Redaction gateway (Presidio + secrets scanners) with latency report
- [ ] DPO/ORPO dataset builder from **all-provider** implicit feedback
- [ ] FIM hard-negative synthesizer (Delulu taxonomy; enterprise teachers for negatives)
- [ ] Continual FT job → **org-small / org-large** GGUF (CURLoRA and/or O-LoRA + optional EWC/LwF)
- [ ] Kubeflow (or equiv.) scheduled pipeline with security + eval + **credit-sim** gates
- [ ] Dashboards: forgetting, FIM, **credit burn**, **% org-model traffic**
- [ ] Training data policy + model card template (teacher mix lineage)

---

## Explicit anti-goals / risks to avoid

| Risk | Mitigation |
|------|------------|
| Full agent inside WASM 4 GB | Keep harness native; WASM = inference only |
| Partial GPU offload | Force 0 or 99 |
| Over-build router before loop works | Phase 2 before Phase 5 sophistication |
| Hot-swap adapters before base path | Phase 7 after Phases 1–4 |
| Compaction drops critical state | Always re-inject project memory + todos |
| Small model alone for multi-file hard tasks | Escalation + large pool; monitor escalate % |
| Secrets in git / prompts / train set | .gitignore, gateway redaction, CI scanners |
| DPO on Sub-3B causing collapse | Bootstrap with SFT/KL; DPO when pairs mature; prefer 7B for heavy DPO |
| **Train on raw chat history** | Phase 3A mandatory before 3C |
| **Chat-history poisoning → trained trigger** | Strict schemas + injection filters + red-team gate |
| **Naive LoRA monthly FT** | O-LoRA / CURLoRA / EWC / FIP; regression gates |
| **FT before prompting/RAG exhausted** | Policy gate + ≥500 quality pairs + lift metric |
| **Cognitive debt from generic AI** | Continual alignment + human ECVM-style oversight |
| **Always default to heavy Claude/OpenAI/Kimi** | Credit ladder: org-small first; heavy = escalate/teacher |
| **Capture only one vendor’s sessions** | Single gateway corpus; all providers tagged and trained |
| **Train org models but never shift traffic** | Post-promote router re-bias + credit dashboards |
| **Ignore enterprise ToS / employee notice** | Legal review for training-on-usage per provider contract |
---

## Suggested first week (immediate next steps)

| Day focus | Work |
|-----------|------|
| Days 1–2 | **Phase 0** scaffold + **Phase 1.1–1.2** memory-separation experiment |
| Days 2–4 | **Phase 1.3** minimal `try_load_gpu` + confidence probe CLI |
| Parallel | **Phase 2.1–2.2** agent loop + 5 tools against a strong API model |
| Early design spike | **TelemetryEvent** schema + redaction gateway strawman (3A) — no full pipeline yet |
| End of week | Demo: (A) heap proof write-up, (B) agent edits a toy repo via tools |

---

## Source index

| Doc / source | Role |
|--------------|------|
| `docs/deepseek.md` / `docs/grok.md` | Research prompt v3 (constraints, RQs, metrics, risks) |
| `docs/gemini.md` | Dual-mode architecture deep-dive (heap, LoRA, distillation, TCO) |
| `docs/meta.md` | Validated research report (model choice, routing, snippets) |
| `docs/deepseek-code.md` | 7-step agent blueprint |
| `docs/gemini-code.md` | ReAct harness steps (Python-oriented) |
| `docs/grok-code.md` | **Recommended 10-step build plan + two-layer architecture** |
| `docs/meta-code.md` | 14-step Claude Code reverse-engineering + WASM-mapped 10 steps |
| *Continual Learning Pipelines for Enterprise Coding Models* (2026-07-23) | Telemetry corpus, gateway redaction, DPO/FIM hard negatives, CF mitigations, Kubeflow LLMOps |

Update this file as phases complete: mark checkboxes, note owners/dates, and link PRs.
