# 10 — Intel Qwen3.8-Flash-Next W4A16 RTN AutoRound (candidate pin)

Status: PIN + PREFETCHED. Verified bytes for this candidate now exist on
shared storage (transfer receipt: `docs/11-intel-prefetch-receipt.md`); no
build, model load, or GPU use has happened for it. Compute remains blocked
until #52 grants #56 the four-card lane after a fresh authoritative live
preflight.

## Pinned identity (all measured from Hub metadata APIs, 2026-08-31)

- Repo: `Intel/Qwen3.8-Flash-Next-W4A16-RTN-AutoRound` @ `a729382b72baabf11a6c10f35e9042b98cc06ef3`
- License: Qwen Community 1.0 (LICENSE + README metadata)
- Files: 146 total, 168.84 GiB (181,243,078,643 B); 131 safetensors shards =
  168.74 GiB (181,195,588,648 B), every shard carries an LFS SHA256 in the
  Hub tree manifest
- Quantization: AutoRound 0.14.2, `quant_method: auto-round`, `model_free: true`,
  `iters: 0` (pure RTN), W4A16, group 128, symmetric, packing
  `auto_round:auto_gptq`
- Architecture: `Qwen4ExpForConditionalGeneration` (same family as baseline);
  48 layers = 12 full-attention + 36 linear-attention (config layer_types);
  512 routed experts, top-10, MTP 1 layer

## Quantized vs preserved tensors (from quantization_config.json)

- 2,684 tensors are explicitly overridden to BF16 (bits 16):
  - 2,514 in the language core: all router gates (49), shared-expert linears
    (196), hyper-connection mixer weights/norms (398), linear-attention
    in/out projections (216), indexer/attention projections and norms in the
    12 full-attention layers (117), lm_head (1)
  - 165 in the vision tower (all blocks), 4 PLE projections/norms,
    1 embed/head entry
- Notably the entire ngram PLE embedding is NOT in the override list and
  therefore falls under the default 4-bit group-128 scheme. This differs from
  the cyankiwi AWQ baseline, which leaves the 95.43 GiB PLE in BF16 and
  excludes 1,083 tensors from quantization. Static-byte consequences below.
- Baseline comparison (identical base model): cyankiwi AWQ INT4 = 175.32 GiB
  with 95.43 GiB PLE in BF16; Intel RTN W4A16 = 168.74 GiB total including a
  PLE quantized at 4 bits. Same-BF16-PLE accounting for the Intel artifact
  would be 168.74 + (95.43 - ~11.93) = 252.24 GiB-equivalent, i.e. it does
  NOT fit 4x64 GiB under the baseline's memory layout; the fit claim depends
  on the quantized PLE path being loadable and correctly served. This is the
  primary technical risk for the A/B.

## Runtime requirements (gated, not yet run)

- Same serving path as docs/02: vLLM PR #53899 stack (re-verify state/head at
  run time; last measured 2026-08-31: OPEN, dirty, 935728b4).
- vLLM AutoRound loading: mainline support for `quant_method: auto-round`
  (W4A16 path) is expected via the compressed-tensors/AutoRound loaders;
  exact loader path on the pinned stack is UNTESTED on CMP until the lease.
- Launch examples in the candidate README are CUDA-generic; no SM80-specific
  claims exist. Treat all as untested until measured.

## Static fit / topology (inferred, planning only)

- Weights: 168.74 GiB total = 42.19 GiB/card at TP4 (balanced), before KV,
  activations, CUDA graphs, NCCL and temporaries (all unmeasured).
- KV geometry unchanged from baseline: 2 KV heads x head_dim 256, BF16,
  2 KiB/token/layer, 24 KiB/token all 12 full-attn layers, replicated per
  rank (2 heads < 4 ranks).
- At 32k context: ~0.75 GiB KV/card; pre-KV headroom 58.88 - 42.19 =
  16.69 GiB/card. Tighter than the baseline's 38.91 GiB/card pre-KV; CUDA
  graph + NCCL + activation overheads are the unknowns. The safe-fallback
  8k no-CUDA-graph cell from eval/configs/matrix.yaml applies.

## A/B plan (locked to eval/configs/matrix.yaml at run time)

- Candidates: cyankiwi AWQ INT4 @ d39638a0 (baseline) vs Intel RTN W4A16 @
  a729382b. One bounded round each, same harness/prompts/context/concurrency,
  one terminal verdict (best recipe or reproducible negative).
- Gate: only after #52 explicitly grants #56 the four-card lane following the
  fresh authoritative live preflight (per #52 comment 5474393282 and owner
  command 5475201650).
