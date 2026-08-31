# 03 — Evaluation plan (Qwen3.8-Flash-Next AWQ INT4 on 4 x CMP 170HX)

Status: plan only. **Untested** — no command in this plan has ever run on the
target platform. Execution requires the gates in `README.md` to clear.

## Baseline round scope

Per the owner's scope recorded in internal tracking, the next execution is
**ONE fixed baseline round** — not the full suite below. The round consists of
exactly one fixed measurement set on the two ACTIVE matrix cells
(`eval/configs/matrix.yaml`):

1. **Startup / compatibility verdict** — launch the primary cell; record
   launch success/failure and a classified verdict
   (`infrastructure` / `compatibility` / `capacity` / `stability`).
2. **Fixed speed + latency set** — the metrics listed under "Metrics to
   capture per run" below, with **three repetitions where practical**
   (skip rules per the "Seeds and sampling" section, justified in the
   manifest).
3. **Bounded quality/reliability smoke** — about 20–50 deterministic text
   prompts; 10–20 structured tool-call checks; **1** long-context retrieval
   check at 32k; 5–10 vision/chart checks.
4. **Sanitized hardware receipts** — per-card power/thermal peaks, driver,
   runtime commit, and topology, redacted per the publication boundary in
   `AGENTS.md`.
5. **Negative-verdict protocol** — if startup or a safety stop condition
   fails, dump logs/thermals per `scripts/preflight-checklist.md`, classify
   the run in the manifest, file the negative result, and **stop**: a
   classified negative verdict completes the round.

**PARKED:** the full five-family evaluation suite (task families 1–5 below,
the split policy, and the cost model) is parked until **all four model
baselines publish** per master policy. The methodology text below is kept
unchanged for those later rounds; nothing in it is scheduled for the
baseline round beyond the bounded smoke set above.

## Hypothesis and predicted result

**Hypothesis (H1):** Qwen3.8-Flash-Next-AWQ-INT4 with PLE CPU offload serves
on 4 x CMP 170HX (SM80, 256 GiB aggregate) at ≥ 32k context while keeping
≥ 15% post-allocation VRAM headroom per card, with AWQ INT4 quality within
noise of the unquantized checkpoint on text and tool-calling tasks and
unchanged vision quality (vision tensors are unquantized; see
`docs/00-pinned-checkpoint.md`).

**Predicted result (inferred, from `docs/01-vram-ram-fit.md`):** Scenario B
fits at 32k with ≈ 54% headroom (inferred); Scenario A (no offload) is a
fit-probe expected to fail or thrash. If H1's capacity leg fails, the run is
classified `capacity`, not `quality`.

## Split policy: development vs held-out

- **Development split:** used to debug launch, PLE offload, and harness
  mechanics. May be iterated on freely. Results are labeled
  development-quality and never cited as headline numbers.
- **Held-out split:** frozen before the first leased run; touched only after
  the configuration is fixed. Every held-out number requires a run manifest
  (`eval/configs/run-manifest.schema.json`). No parameter, prompt, or sampling
  change may be made after seeing held-out results; violations are recorded
  as development data.
- Splits are disjoint by task/prompt ID; source datasets and the exact split
  lists get pinned in `eval/` before the first run.

## Seeds and sampling

- Fixed seeds for every run, recorded in the run manifest
  (`seed` field). Greedy decoding (temperature 0) where the runtime supports
  it; otherwise temperature/top-p fixed and pinned per matrix cell.
- **Three-repetition minimum where practical** (always for benchmark-style
  scoring; skipped only for long-context monolith runs where one repetition
  exceeds the lease window — the skip must be justified in the manifest).
- Report median and spread across repetitions, plus per-repetition raw output
  references.

## Task families

1. **Text quality** — standard knowledge/reasoning suites with deterministic
   extraction (exact-match or rule-graded), plus fixed prompt sets scored
   consistently across runs.
2. **Structured tool-calling** — JSON-schema-constrained calls: schema
   validity, argument correctness, tool-selection accuracy; graded
   programmatically.
3. **Long-context retrieval** — needle-style retrieval and multi-hop lookup
   at 8k/32k/64k/128k (262,144 only if capacity allows); report recall vs
   context length.
4. **Agentic coding / task completion** — repository-level fix tasks and
   multi-step tool loops in a sandbox; pass/fail by test execution
   (deterministic), plus step-count and retry statistics.
5. **Vision / chart / document checks** — chart QA, document VQA, and
   image-understanding checks from fixed sets; vision weights are
   unquantized, so this doubles as a regression guard for the PLE/offload
   plumbing.

## Scoring policy

- **Deterministic, programmatic scoring is preferred** for every task family;
  exact-match, test-execution, schema-validation, and rule-graded extraction
  first.
- **Pinned-judge policy (only if unavoidable):** if a task cannot be graded
  deterministically, use a judge model pinned to an exact revision with a
  frozen rubric and temperature 0, recorded in the manifest. Judge identity
  and revision must never change mid-comparison. Community or unpinned judges
  are prohibited.
- No human grading in the first pass; it may be added later as an audited
  secondary check.

## Cost model

Cost per successful task:

```
cost_per_success = (gpu_hours x cards_in_use x runs) / successful_tasks
```

- `gpu_hours` = wall-clock GPU hours per run x repetitions.
- `successful_tasks` counted at the held-out split, final configuration.
- Record `gpu_hours` and `cards_in_use` in every manifest so the formula is
  computable from raw data.

## Metrics to capture per run (all into the run manifest)

- **Latency:** TTFT (ms), ITL (ms), per-stream tok/s.
- **Throughput:** aggregate tok/s **derived from the final usage object** of
  each request (never from stream-event counts).
- **Memory:** per-card VRAM peak and post-run residual; host RAM peak.
- **Power and thermals:** per-card power (W), core and memory temperature
  peaks; stop conditions per `scripts/preflight-checklist.md`.
- **Stability:** completed/failed/cancelled requests, retries, resets, Xid
  errors, PLE-offload transfer errors.
- **Context/concurrency:** context tokens and concurrency actually used.

Every claimed result carries a **claim label** (`measured`, `inferred`,
`community-reported`, `untested`) in its manifest. Until the first leased
run, everything in this plan is **untested**.
