# 02 — Runtime compatibility

Status: research prep. Nothing here has been run on CMP 170HX. Search and API
observations below are labeled with how they were measured and when.

## Upstream runtime support for `qwen4_exp` (measured 2026-08-30)

| Runtime | `qwen4_exp` support on main | How measured |
| --- | --- | --- |
| vLLM | **absent** from main | measured via registry/code search on 2026-08-30 |
| SGLang | **absent** from main | measured via GitHub code search on 2026-08-30 |
| Hugging Face transformers | **present** (`qwen4_exp` modules exist) | measured via code search on 2026-08-30 |

transformers can load the architecture for inspection, but it is not a serving
path for this platform. Serving on CMP 170HX requires the vLLM gate below.

## Required runtime gate: vLLM PR #53899 (measured via API on 2026-08-30)

- PR: <https://github.com/vllm-project/vllm/pull/53899>
  — "Support PLE-Offload for Qwen3.8-Flash-Next"
- Head commit: `935728b4a95e110d91a41ab4e02b6bed06ec66ab`
- State: **OPEN**, `mergeable_state`: **dirty** (measured via GitHub API on
  2026-08-30; re-check before citing).
- Scope of the PR: model support for `qwen4_exp`, **PLE CPU offload env
  vars**, MTP support, tests, and CUDA/Triton GDN kernels.

**SM80 / CMP 170HX support is UNTESTED.** The only run-success confirmation
associated with this stack is **community-reported on SM120, TP1, NVFP4** —
that is a different GPU architecture, a different topology, and a different
quantization from our target (SM80, TP4, AWQ INT4). It must not be cited as
evidence that SM80 works.

### At-lease rule

During the single AWQ INT4 baseline round (see `docs/03-evaluation-plan.md`):

- Use the **minimum pinned upstream/runtime delta** required to attempt
  startup — no speculative rebases, no cherry-picks beyond what launch needs.
- Record **every patch and commit** (vLLM head, kernels, any local diffs) in
  the run manifest (`eval/configs/run-manifest.schema.json`).
- **No optimization patches** during the baseline round; optimization work is
  parked per
  <https://github.com/seanphan/pixelml/issues/52#issuecomment-5468770811>.

## Auxiliary-GPU PLE offload (optional path)

- Feature request: <https://github.com/vllm-project/vllm/issues/53908>
  — open request for auxiliary-GPU PLE offload. Relevant only as a future
  alternative to CPU PLE offload; **open, not merged, not usable today**.

## Community-reported recipes (unverified)

1. <https://github.com/vektorprime/qwen38-flash-next-pp2> — community recipe
   repository for running Qwen3.8-Flash-Next (pipeline-parallel style).
   **Community-reported; untested here.**
2. <https://huggingface.co/cyankiwi/Qwen3.8-Flash-Next-AWQ-INT4/discussions/1>
   — HF discussion attached to the pinned checkpoint; contains the
   community-reported SM120/TP1/NVFP4 run notes referenced above.
   **Community-reported; untested here.**

**Pinned-commit TODO:** before any build attempt (post-lease only), both
recipes must be reduced to exact pinned commits for vLLM, kernels, and any
patches, recorded in this document. Until then they are not actionable.

## Decision gate (binding)

The upstream situation must be **decision-ready** before a four-CMP lease is
requested. Decision-ready means at least one of:

1. vLLM PR #53899 is **merged** into a released or main state compatible with
   our harness, or
2. PR #53899 is **rebased** to a clean, mergeable state we can pin, or
3. a **pinned branch** exists with concrete **SM80 + AWQ INT4 + GDN + PLE**
   evidence attached (CI or reproducible report).

Until one of these holds, status stays **WAITING_RESOURCE** and no lease is
requested. Re-measure the PR state (state, `mergeable_state`, head SHA) on
every review of this document and update the table above.
