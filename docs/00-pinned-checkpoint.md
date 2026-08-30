# 00 — Pinned checkpoint

Status of this document: research prep. **No weights have been downloaded.**
Download remains prohibited until the upstream gate clears and an exclusive
four-CMP lease is granted (see `README.md`).

## Pinned revision

| Item | Value |
| --- | --- |
| Checkpoint | <https://huggingface.co/cyankiwi/Qwen3.8-Flash-Next-AWQ-INT4> |
| Checkpoint revision | `d39638a0e740fccb3e24ae0ea5cab34c15371ae6` |
| Base model | <https://huggingface.co/Qwen/Qwen3.8-Flash-Next> |
| Base revision | `de4b8e4d43b917e7706784d8bb445c9af86a3540` |
| License | Qwen Community 1.0 — <https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE> |

All work in this repository must reference exactly the revisions above. A
re-pin requires a new document revision and a new review.

## File inventory (measured)

Measured from the Hugging Face Hub API / repo tree on 2026-08-30:

- **38 safetensors shards**, totaling **exactly 188,286,106,928 bytes**
  (= **175.35510186851025 GiB**, dividing by 2^30).
- Average shard size ≈ 4.615 GiB (188,286,106,928 / 38 ≈ 4,954,897,551 B).
- Plus small metadata files (`config.json`, tokenizer, generation config,
  index JSON). Shard byte totals and the file inventory are **measured** from
  the Hub API/tree; they are not reproduced file-by-file here to keep this
  document small.

These byte totals drive the static-fit math in `docs/01-vram-ram-fit.md`.

## Quantization format (measured from `config.json`)

Architecture/config facts below are **measured from `config.json` at the
pinned revision** (`quantization_config`):

- Format: **compressed-tensors, pack-quantized** (`format: pack-quantized`).
- Scheme: AWQ INT4 — `num_bits: 4`, weight `type: int`, `strategy: group`,
  **group size 32**, **asymmetric** (`symmetric: false`, int8 zero points),
  observer `mse`.
- Targets: **Linear** layers only (`targets: ["Linear"]`, `dynamic: false`).
- **1083 ignored tensors** are listed in `quantization_config.ignore`. The
  full ignore list is deliberately **not reproduced** here (bulk generated
  content); it is dominated by `model.visual.*` entries.
- **Vision tensors are unquantized** — the ignore list excludes the vision
  tower from quantization, so vision weights remain bf16.
- Top-level architecture: `Qwen4ExpForConditionalGeneration`, model type
  `qwen4_exp`, multimodal (`vision_config` present, `image_token_id` /
  `video_token_id` set), `tie_word_embeddings: false`.

## Text architecture facts (measured from `config.json`)

From `text_config`, relevant to fit and KV math (`docs/01-vram-ram-fit.md`):

- 48 layers; `layer_types` pattern repeats 3x `linear_attention` + 1x
  `full_attention` (`full_attention_interval: 4`) → **12 full-attention
  layers**, 36 linear-attention layers.
- Full-attention GQA: **24 query heads, 2 KV heads, head_dim 256**, dtype
  **bfloat16**.
- Linear attention: 16 key heads / 48 value heads, key/value head_dim 128,
  conv kernel dim 4.
- Light indexer: `indexer_budget: 2048`, 4 heads, head_dim 128, 1 KV head,
  compress ratio 4.
- `hidden_size: 2560`, `vocab_size: 248320`,
  `max_position_embeddings: 262144`.

## Checkpoint provenance caveats

- The checkpoint is published by a community account (`cyankiwi`), not the
  `Qwen` organization. The base link above is the upstream model; equivalence
  of the quantized weights to the pinned base revision is **community-reported**
  and unverified until a checksum comparison is possible (post-lease).
- Run-success claims tied to this checkpoint are **community-reported**
  (see `docs/02-runtime-compatibility.md`); nothing on CMP 170HX is
  **measured** yet.

## Handling rules

- No weights, shards, or large artifacts in this repository — ever.
- Download only into the configured model library, only after the gates in
  `README.md` clear.
- Obey the Qwen Community 1.0 license for any derived artifacts.
