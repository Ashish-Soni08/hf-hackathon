# Plan: VLM Runner Extension (`--mmproj` vision benchmarking)

**Status:** Proposed — scoping only, no code changes in this doc/branch
**Owner:** (maintainer — runner is a protected file)
**Scope boundary:** This is **separate** from the already-landed baseline model PRs
(SmolVLM-256M #26, SmolVLM-500M #27, SmolVLM2-2.2B #29, ZwZ-4B #30, Qwen3-VL-2B #73).
Those stay as-is; this plan adds the shared capability that upgrades all of them
from text-only decode to actual vision benchmarking.

---

## 1. Problem statement

The vision-language models we ported are currently benchmarked **text-only**. The
`llama_server` runner (`​.github/ci/scripts/run_llama_server_benchmark.py`) only ever:

- materializes the **LLM GGUF** (`model_artifact`),
- launches `llama-server -m <llm.gguf> ...` (no `--mmproj`),
- POSTs a **text** prompt to `/completion`,
- scores decode tok/s + WikiText-2 perplexity.

The vision projector (`mmproj`) GGUF is pinned in `artifacts.json` for every VLM but
is **never materialized, never passed to the binary, and no image ever reaches the
API**. So the differentiating capability of these models — image understanding — is
not exercised or scored.

**Goal:** extend the shared runner + config schema so a VLM benchmark loads its
`mmproj`, sends an image + prompt, and scores a vision task, while remaining fully
backward-compatible with every existing text-only LLM benchmark.

---

## 2. Current state (verified against the code)

| Concern | File / location | Today |
|--------|------------------|-------|
| Runner entrypoint | `.github/ci/scripts/run_llama_server_benchmark.py` `main()` (~L549–803) | LLM-only |
| Server CLI assembly | same file (~L624–648) | no `--mmproj` |
| Artifact resolution | `materialize_artifact()` (~L156–193) | generic — reusable for mmproj/image |
| Request payload | (~L652–669) `completion` / `chat` text only | no image content |
| Scoring | decode tok/s (`timings.predicted_per_second`), PPL via `llama-perplexity` | text metrics only |
| Board execution | `.github/ci/platform/deploy/soc3-benchmark.sh` (~L153–174 env forwarding) | no `*_MMPROJ_PATH` vars |
| Config validation | none (no JSON schema); `ci_preflight.sh` only checks JSON parses | unknown keys silently ignored |
| Runner protection | `benchmark-board.yml` (~L139), `changed_benchmark_models.py` (~L44) | runner is a **protected** file → maintainer PR only |
| CI model selection | `changed_benchmark_models.py` `collect_artifact_refs()` (~L185–198) | any `*_artifact` key already tracked |

Key consequence: adding a `mmproj_artifact` key is automatically picked up by
change-detection, and unknown config keys are non-breaking — so the rollout can be
incremental and opt-in per model.

---

## 3. Design

### 3.1 Config schema additions (per-model `llama_server` block)

Opt-in, additive keys — absent = current text-only behavior:

```jsonc
"llama_server": {
  "model_artifact": "smolvlm_256m_q8_gguf",
  "mmproj_artifact": "smolvlm_256m_mmproj_q8",   // NEW: enables vision load
  "vision": {                                     // NEW: vision request+scoring
    "image_artifact": "vlm_eval_image_coco_dog",  // small pinned test image
    "prompt": "What animal is in this image? Answer with one word.",
    "expect_substring": "dog",                    // scoring: case-insensitive contains
    "max_tokens": 32
  }
}
```

- `mmproj_artifact`: key into `artifacts.json` (same schema as `model_artifact`).
- `vision.image_artifact`: a new **image** artifact (small, license-clean, pinned by
  SHA256, same source/cache mechanism as GGUFs).
- `vision.expect_substring`: minimal, deterministic correctness check (temperature 0).
- If `vision` is absent but `mmproj_artifact` is present → load projector but keep the
  text prompt (still a valid "VLM loads on ET" signal).

### 3.2 Runner changes (`run_llama_server_benchmark.py`)

1. **Materialize mmproj** — after `model_path` is resolved (~L576):
   ```python
   mmproj_id = lcfg.get("mmproj_artifact")
   mmproj_path = materialize_artifact(mcfg, mmproj_id) if mmproj_id else None
   ```
2. **CLI arg** — in cmd assembly (~L648):
   ```python
   if mmproj_path:
       cmd += ["--mmproj", str(mmproj_path)]
   ```
   (Confirm exact flag against the pinned ET revision — see Open Questions.)
3. **Vision request** — new branch in payload build (~L652). Use the
   `/v1/chat/completions` multimodal contract with a base64 data-URI image:
   ```python
   img_b64 = base64.b64encode(image_path.read_bytes()).decode()
   content = [
     {"type": "text", "text": vision_prompt},
     {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{img_b64}"}},
   ]
   ```
4. **Scoring** — reuse decode tok/s; add optional `expect_substring` check for the
   vision response. Keep it additive to existing `success_substring` logic.
5. **Perplexity** — leave text PPL as-is (projector not needed for PPL). See §3.5.

### 3.3 Board env forwarding (optional, perf only)

`soc3-benchmark.sh` (~L153–174): add `*_MMPROJ_PATH` vars mirroring `*_MODEL_PATH`
so pre-staged mmproj blobs skip download. Not required for correctness (falls back to
`local_cache` + HF URL), so this can be a follow-up.

### 3.4 Test image artifact

- Add one (or a tiny set of) small, permissively-licensed image(s) to `artifacts.json`
  (`kind: "image"`), pinned by SHA256 with an HF/`local_cache` source.
- Prefer a public-domain / CC0 image with an unambiguous answer (single object).
- Keep <200 KB; no image blob committed to the repo (download by URL like GGUFs).

### 3.5 Perplexity / leaderboard policy for VLMs

- Text WikiText-2 PPL stays enabled and unchanged (decode path is identical).
- Decide whether the **vision correctness** result is a hard gate or reported-only for
  the first iteration. Recommendation: **report-only** initially (pass/fail logged in
  score JSON, not enforced by `leaderboard_gate.py`), then promote to a gate once
  stable. `leaderboard_gate.py` (~L318–320, L525–540) already keys PPL off
  `perplexity.enabled`; vision scoring would be a new, separate signal.

---

## 4. Milestones

| # | Milestone | Deliverable | Depends on |
|---|-----------|-------------|-----------|
| M0 | **Confirm ET vision support** | Notes: does the pinned ET `llama-server` build include `libmtmd`? Do CLIP/vision ops run on `GGML_ET`? Exact `--mmproj` flag + HTTP image contract | submodule / Modal smoke test |
| M1 | **Config schema** | `mmproj_artifact` + `vision` block documented; one model JSON updated (SmolVLM-256M, smallest) | M0 |
| M2 | **Runner: load mmproj** | Runner materializes + passes `--mmproj`; server loads projector; text path unchanged | M1 |
| M3 | **Runner: vision request+score** | Base64 image → chat API; `expect_substring` scoring; score JSON carries vision result | M2 |
| M4 | **Test image artifact** | Pinned CC0 image in `artifacts.json` | — |
| M5 | **Board plumbing** | Optional `*_MMPROJ_PATH` env forwarding; board run green on SmolVLM-256M vision | M2–M4 |
| M6 | **Roll out to all VLMs** | Add `mmproj_artifact`+`vision` to 500M / 2.2B / ZwZ-4B / Qwen3-VL-2B | M3–M5 |
| M7 | **Promote to gate (optional)** | Enforce vision correctness in `leaderboard_gate.py` | M6 stable |

M1–M4 are one focused maintainer PR against `main` (runner is protected). M6 is small
per-model config PRs that can reuse the existing branches or new ones.

---

## 5. Risks & open questions

| Risk / question | Impact | Mitigation |
|-----------------|--------|-----------|
| **ET backend may not implement CLIP/vision ops** (`GGML_ET`) | High — vision could fall back to CPU or fail | M0 Modal/board smoke test before any config rollout; if CPU-only, still valid but note kernel-wait metric |
| Exact `--mmproj` flag / HTTP image contract differs on pinned ET revision | Med | M0 verifies against `13da971…` `et` branch build; adjust flag/endpoint |
| Runner is a **protected file** | Process | Land runner change as a maintainer PR on `main`; participant model PRs only touch config/artifacts |
| No config schema → typos silently ignored | Low/Med | Add a light `ci_preflight.sh` check that if `mmproj_artifact`/`vision.image_artifact` are set, the keys exist in `artifacts.json` |
| Vision scoring flakiness (non-deterministic captions) | Med | temperature 0, single-object image, `expect_substring` (not exact match); report-only first |
| Board time budget grows (extra load + request) | Low | mmproj is small; reuse same server process; keep one image |
| Metric ambiguity: is the headline still decode tok/s? | Med | Keep decode tok/s as leaderboard metric; vision correctness is a separate pass/fail signal, not a speed metric |

---

## 6. Backward compatibility

- All new keys are optional; every existing benchmark (all text LLMs + current
  text-only VLMs) runs byte-for-byte identically when `mmproj_artifact`/`vision` are
  absent.
- Change-detection (`collect_artifact_refs`) already tracks `*_artifact` keys, so no CI
  selection changes are needed.
- Rollout is per-model and reversible (remove the keys → back to text-only).

---

## 7. Validation plan

1. **M0 smoke**: on Modal/board, run pinned ET `llama-server -m <llm> --mmproj <proj>`
   and one `/v1/chat/completions` image request; confirm a coherent answer + that ET
   offload log lines still appear.
2. **Runner unit path**: dry-run `run_llama_server_benchmark.py` with `--model
   smolvlm_256m` and vision keys on a dev box (sysemu) to exercise materialize + payload
   build without silicon.
3. **Board E2E**: PR on `main` → `board (smolvlm_256m)` runs vision path, score JSON
   shows `vision.passed=true`, decode tok/s + PPL unchanged.
4. **Regression**: confirm a pure-LLM model (e.g. `qwen3_8b`) is unaffected.

---

## 8. Out of scope (explicitly)

- The **ggonnx** ONNX vision track (D-FINE / RF-DETR / EfficientViT / TinyViT) — separate
  runner, separate plan.
- Video input (Qwen3-VL / SmolVLM2 support video; start with single image).
- Multi-image / interleaved prompts.
- Changing the leaderboard **primary metric** (stays decode tok/s).
