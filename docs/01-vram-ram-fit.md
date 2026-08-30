# 01 — VRAM / RAM static fit (4 x CMP 170HX, 64 GiB each)

**All math in this document is inferred.** No allocation has ever been
measured on this platform; every memory overhead is **untested until
measured**. Inputs that are measured (checkpoint bytes) come from
`docs/00-pinned-checkpoint.md`.

## Inputs

| Quantity | Value | Label |
| --- | --- | --- |
| Cards | 4 x 64 GiB = **256 GiB** aggregate | given platform target |
| Usable VRAM fraction | 92% → **58.88 GiB/card**, **235.52 GiB** aggregate | assumed planning constant |
| Checkpoint weight bytes | **175.35510186851025 GiB** (188,286,106,928 B) | measured (Hub API/tree) |
| PLE bf16 portion | **≈ 95 GiB** | community-reported; verify from safetensors headers before any run |
| AWQ INT4 residual | ≈ 175.355 − 95 ≈ **80.36 GiB** | inferred |
| Full-attention layers | 12 of 48 (every 4th layer) | measured (config.json) |
| KV geometry | 2 KV heads x head_dim 256, BF16 | measured (config.json) |

## Formulas

- Usable VRAM/card = 64 GiB x 0.92 = **58.88 GiB**
- Aggregate usable = 4 x 58.88 = **235.52 GiB**
- Per-card KV bytes (one full-attention layer) =
  2 (K+V) x kv_heads x head_dim x dtype_bytes
  = 2 x 2 x 256 x 2 = **2,048 B/token/layer** (2 KiB)
- All-layers KV bytes = 2 KiB x 12 full layers = **24 KiB/token**
- TP4 KV placement: 2 KV heads cannot be divided across 4 ranks, so KV heads
  are replicated per rank (inferred, standard vLLM TP behavior when
  `num_kv_heads < tp_size`) → **per-card KV = total KV** (no division by 4).

## Equal-shard weight split is INVALID for production

Naive split: 175.355 GiB / 4 = **43.84 GiB/card** by shard files. This is
invalid for production:

1. Safetensors shards are arbitrary cuts of the parameter stream, not
   semantic units; shard boundaries do not align with layers or TP shards.
2. 38 shards do not even divide by 4 cards (9.5 shards/card).
3. Runtimes distribute tensors by tensor-parallel rules, not by file
   boundaries. Any "one shard per card" layout breaks weight loading.

Production must let the runtime shard weights (TP4 or PP4), not the file
system.

## Scenario A — no PLE offload (fit probe only)

```
aggregate: 175.355 (weights) + non-PLE bf16/GUARD + KV + activations
           <= 235.52 GiB usable
slack:     235.52 - 175.355 = 60.16 GiB for everything else
```

- After weights, only **60.16 GiB aggregate** remains for non-PLE bf16/GUARD
  tensors (vision tower, unquantized/ignored tensors), KV cache, and
  activations. Expected **tight**; no PLE offload leaves all bf16 tensors on
  GPU. Inferred: likely infeasible at useful context; treat as a bounded
  **fit probe**, never a serving configuration.
- **Explicit runt-card failure risk:** TP4 requires every card to hold
  ≈ 43.84 GiB of weights plus KV and activations. A single card with less
  usable VRAM (reserved carve-outs, ECC, clock/thermal throttling) fails the
  entire launch even when the aggregate fits. Do not interpret an aggregate
  fit as a per-card fit.

## Scenario B — CPU PLE offload (RECOMMENDED target)

```
GPU:  residual AWQ weights ≈ 80.36 GiB  (≈ 20.09 GiB/card average)
      + KV cache + activations + CUDA graphs + NCCL + temporaries
CPU:  PLE bf16 ≈ 95 GiB (pinned host memory)
RAM:  95 GiB x 1.25 = 118.75 GiB   (1.25x overhead ASSUMED, unmeasured)
      + pinned transfer buffers + runtime temporaries
```

- PLE bf16 ≈ 95 GiB is **community-reported**; it must be re-derived from
  safetensors header sizes before any run.
- Host RAM: until measured, budget **1.25x** the PLE size ≈ **118.75 GiB**,
  i.e. plan for ≥ 128 GiB host RAM and prefer more. Untested.
- Per-card average weight load ≈ 20.09 GiB → "20 GiB/card average."
- Pinned-memory transfers, CUDA graph pools, NCCL buffers, and temporary
  activation buffers are all **untested** on this platform.

## KV cache estimates (GQA full-attention layers, BF16)

Per token, all 12 full-attention layers: **24 KiB**. With TP4 KV-head
replication, each card carries the full amount (inferred). Linear-attention /
mamba recurrent state and the light-indexer state (`indexer_budget: 2048`)
are **additional and unmeasured** — not included in this table.

| Context (tokens) | KV bytes, all full layers | Per card (TP4, replicated) |
| ---: | ---: | ---: |
| 8,192 | 192 MiB (201,326,592 B) | 192 MiB |
| 32,768 | 768 MiB (805,306,368 B) | 768 MiB |
| 65,536 | 1.5 GiB (1,610,612,736 B) | 1.5 GiB |
| 131,072 | 3 GiB (3,221,225,472 B) | 3 GiB |
| 262,144 | 6 GiB (6,442,450,944 B) | 6 GiB |

KV is small relative to weights because only 12 of 48 layers are
full-attention; the dominant unknowns are PLE placement and unmeasured
recurrent/indexer state.

## Worked check at 32,768 context (Scenario B, TP4)

- Full-attn KV bytes: 32,768 x 24 KiB = **805,306,368 B ≈ 0.75 GiB per card**
  (replicated, inferred).
- Residual VRAM per card before overheads:
  20.09 (weights) + 0.75 (KV) = **20.84 GiB**
- Usable per card: **58.88 GiB**
- Unmeasured overhead allowance (activations + CUDA graphs + NCCL +
  temporaries): assume a generous **6 GiB estimate** (untested).
- Post-allocation headroom: 58.88 − 20.84 − 6 = **32.04 GiB = 54.4% of
  usable**.
- 15% headroom threshold: 0.15 x 58.88 = **8.83 GiB**. Overheads would have
  to exceed ≈ 29.2 GiB/card to break the rule.

**Verdict: Scenario B has at least 15% post-allocation headroom at 32k
context — inferred PASS**, contingent on the ≈ 95 GiB PLE figure and the 1.25x
host-RAM multiplier being confirmed. Scenario A is not carried forward.

## Status

Everything on this page is static math. The first leased run must replace
every "untested" label above with measured numbers captured in a run manifest
(`eval/configs/run-manifest.schema.json`).
