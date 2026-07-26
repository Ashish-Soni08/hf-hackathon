# AIFoundry + OpenHW CORE-ET Hackathon — Journey Log

Personal / team working log for contributions by **Ashish-Soni08** to
[`aifoundry-org/hf-hackathon`](https://github.com/aifoundry-org/hf-hackathon).

This file is the **canonical narrative** of what we submitted, what merged, what
failed, and what is blocked on maintainer/infra. It supersedes the older
scattered notes under `docs/vision_models/PR_TRACKER.md` (closed PR #40) and
ad-hoc agent handoffs.

Last updated: **2026-07-26**.

---

## TL;DR

| Outcome | Items |
|---------|--------|
| **Merged** | #11 Qwen3-8B · #26 SmolVLM-256M · #27 SmolVLM-500M · #112 VLM identity contracts |
| **Open (auth green; board score blocked)** | #29 SmolVLM2-2.2B · #30 ZwZ-4B · #73 Qwen3-VL-2B |
| **Closed without merge** | #8 LFM2.5-350M · #24 Qwen3-8B batch revert · #25 Whisper fidelity · #40 vision tracker docs |
| **Participant work is done** on open VLM PRs | Vision harness configs, mmproj, COCO oracles, #112 contracts, **no protected CI edits** |
| **Cannot finish alone** | aifoundry3 glog/sys-emu prebuild · protected `ignore_eos` / KV-metadata runner flags · ET kernels for `CONCAT`/`ROPE`/`UPSCALE` |

---

## Timeline (high level)

1. **Early llama.cpp-et ports** — Qwen3-8B (#11 merged). LFM2.5-350M (#8) closed.
2. **Vision pivot** — SmolVLM-256M (#26) and SmolVLM-500M (#27) merged after moving onto the trusted `smolvlm2_video` path.
3. **CI identity work** — #112 merged so `qwen3vl.*` / `B`-unit params / optional model names validate.
4. **Larger VLM ports opened** — #29 SmolVLM2-2.2B, #30 ZwZ-4B, #73 Qwen3-VL-2B.
5. **Maintainer redesign ask (Jul 22)** — stop text-only `llama_server` for VLMs; require mmproj + image oracles + offload gates.
6. **Redesign implemented** — all three open PRs reworked to `smolvlm2_video` + contracts.
7. **Protected-file mistake** — agents briefly edited `run_smolvlm2_video_benchmark.py`; Authorize failed. **Reverted** so auth passes again.
8. **Whisper fidelity (#25)** — closed Jul 26; Whisper board workload removed from `main`; not a new model-port credit; assets not fork-actionable.
9. **Current state** — Authorize ✅ on #29/#30/#73; Leaderboard gate ❌ because board jobs abort in ~22s with no hardware-epoch score (aifoundry3 infra).

---

## PR inventory

### Merged

| PR | Title | Merged | Notes |
|----|-------|--------|-------|
| [#11](https://github.com/aifoundry-org/hf-hackathon/pull/11) | Qwen3-8B Q8_0 llama.cpp-et | 2026-07-08 | Text LLM; board credit |
| [#26](https://github.com/aifoundry-org/hf-hackathon/pull/26) | SmolVLM-256M-Instruct | 2026-07-09 | First VLM port |
| [#27](https://github.com/aifoundry-org/hf-hackathon/pull/27) | SmolVLM-500M-Instruct | 2026-07-22 | Pattern for later VLMs (`smolvlm2_video`) |
| [#112](https://github.com/aifoundry-org/hf-hackathon/pull/112) | VLM identity architecture contracts | 2026-07-23 | Unblocks qwen3vl identity validation |

### Open (as of 2026-07-26)

| PR | Branch (fork tip) | What we landed | Still blocked |
|----|-------------------|----------------|---------------|
| [#29](https://github.com/aifoundry-org/hf-hackathon/pull/29) | `feat/smolvlm2-2.2b` @ `3daab3d` | `smolvlm2_video` + mmproj + COCO + `pmc_cycles` + #112 contract; **no** protected runner edits | Board empty score (infra); identity may need maintainer opt-out for missing GGUF `head_count_kv` (GQA=1) |
| [#30](https://github.com/aifoundry-org/hf-hackathon/pull/30) | `feat/zwz-4b` @ `291b4bc` | Same vision pattern; PPL contract consistent; `ignore_eos` in **JSON only** | Same infra; then likely qwen3vl ET fallbacks |
| [#73](https://github.com/aifoundry-org/hf-hackathon/pull/73) | `feat/qwen3vl-2b` @ `1e603ab` | Vision harness + #112 contract; prior healthy board had accuracy 1.0 | Infra now; then 2/3 tokens without runner `ignore_eos`; `CONCAT`/`ROPE`/`UPSCALE` CPU fallbacks until llama.cpp-et kernels |

**Files each open PR may touch (hackathon-safe):**

- `.github/ci/benchmark_config.json` (register model key only)
- `.github/ci/reference/<model>.json` (new identity/oracle contract)
- `ported_models/llama_cpp_et/{artifacts,benchmarks,docs}/…`
- `docs/HF_REFERENCES.md` row
- `docs/vision_models/<model>.md` plan

**Must not touch (Authorize blocks):**  
`.github/ci/scripts/run_smolvlm2_video_benchmark.py` and other protected runners/oracles listed in the board authorize job.

### Closed (not merged)

| PR | Why closed |
|----|------------|
| [#8](https://github.com/aifoundry-org/hf-hackathon/pull/8) LFM2.5-350M | Not pursued / closed early |
| [#24](https://github.com/aifoundry-org/hf-hackathon/pull/24) Qwen3-8B batch revert | Experiment cleanup; closed |
| [#25](https://github.com/aifoundry-org/hf-hackathon/pull/25) Whisper fidelity | Whisper removed from `main`; not a new port credit; needs maintainer assets + restored workload |
| [#40](https://github.com/aifoundry-org/hf-hackathon/pull/40) Vision PR tracker docs | Tracker PR closed; narrative moved here |

---

## What “done on our side” means for #29 / #30 / #73

### Implemented (participant-legal)

- [x] Trusted vision runner (`smolvlm2_video`), not text-only `llama_server`
- [x] Pinned mmproj loaded via benchmark config
- [x] Deterministic COCO image cases + host oracle / order-sensitive checks in reference JSON
- [x] `require_full_offload` + `require_zero_vision_fallbacks` + `pmc_cycles`
- [x] #112-style identity contracts (`parameter_count` with `B` unit, `n_vocab`, `metadata_key_prefix` / `require_model_name` as needed)
- [x] Rebased onto current `main`
- [x] Protected CI scripts **identical to `main`** (empty diff)
- [x] Local checks: JSON parse, `benchmark_config_helpers.py --models …`, unit tests where applicable

### Not implementable from a fork PR

| Gap | Owner |
|-----|--------|
| aifoundry3 ELF/sys-emu **glog link** failure → ~22s board job, synthetic fail score, “hardware epoch” gate | Infra / maintainers |
| `performance.ignore_eos` actually honored by `run_smolvlm2_video_benchmark.py` | Maintainer (protected file) |
| Optional `head_count_kv` when GGUF omits key (SmolVLM2-2.2B Q4 GQA=1) | Maintainer (protected file) |
| ET coverage for qwen3vl_merger ops `CONCAT`, `ROPE`, `UPSCALE` | llama.cpp-et / maintainers |

Further solo config churn will not clear the Leaderboard gate until those move.

---

## Lessons learned

1. **Participant PRs cannot edit protected runners.** Even “helpful” flags (`ignore_eos`, KV metadata) fail Authorize. Put keys in contract JSON; ask maintainers to wire them on `main`.
2. **VLM submissions must exercise vision.** Text PPL alone was rejected (Jul 22 reviews on #29/#30).
3. **Board job “success” ≠ valid score.** ~22s green jobs with `benchmark_device: unknown` still fail the gate.
4. **Healthy-board unrankable ≠ infra.** On aifoundry2, #73 had correct cat answers but CPU vision fallbacks and 2/3 tokens → no `pmc_cycles`.
5. **Whisper fidelity is a separate track.** Quality findings (FP32 convs, per-dim token scales) are real, but need a restored board workload + hosted weights.

---

## Workspace layout (local)

Canonical clone used for docs / cleanup:

```text
D:/Github/ai-foundry-hf-hackathon/repo/     # main working clone (fork + origin)
```

Disposable / agent leftovers (safe to delete after tips are on fork):

```text
D:/Github/ai-foundry-hf-hackathon/zwz-work/   # best-of-n worktree clone
D:/Github/ai-foundry-hf-hackathon/clone.log
```

Fork branches of record:

| Branch | Purpose |
|--------|---------|
| `feat/smolvlm2-2.2b` | PR #29 |
| `feat/zwz-4b` | PR #30 |
| `feat/qwen3vl-2b` | PR #73 |
| `docs/hackathon-journey` | This document |
| `docs/vision-models-pr-tracker` | Obsolete tracker (superseded by this file) |
| `feat/whisper-fidelity-improvements` | Closed PR #25 (historical) |

---

## Maintainer ask (next external action)

Please (Discord `#Lab` or comment on #73 with links to #29/#30):

1. Confirm / fix **aifoundry3** glog/sys-emu prebuild so board jobs produce real scores.
2. Land tiny protected-runner support for `performance.ignore_eos` and optional `head_count_kv`.
3. Clarify ownership of **qwen3vl** ET vision ops (`CONCAT`/`ROPE`/`UPSCALE`).
4. After (1)–(2), we will request fresh ET runs on #29 / #30 / #73.

---

## Related in-tree docs

| Doc | Role |
|-----|------|
| [`docs/SUBMISSION_GUIDE.md`](./SUBMISSION_GUIDE.md) | Official PR checklist |
| [`docs/HF_REFERENCES.md`](./HF_REFERENCES.md) | Pinned HF artifacts |
| [`docs/vision_models/smolvlm-256m.md`](./vision_models/smolvlm-256m.md) | Merged #26 plan |
| [`docs/vision_models/smolvlm-500m.md`](./vision_models/smolvlm-500m.md) | Merged #27 plan |
| Per-PR plans on feature branches | `docs/vision_models/{smolvlm2-2.2b,zwz-4b,qwen3vl-2b}.md` |
| Per-PR recipes | `ported_models/llama_cpp_et/docs/<model>.md` on feature branches |

---

## Decision log (2026-07-26)

- **Close #25** — Whisper not actionable; focus on VLMs.
- **Stop editing protected runners** — revert and keep model-port-only diffs.
- **Stop solo churn on #29/#30/#73** until infra + maintainer runner/kernel work; document and wait.
