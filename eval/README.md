# Evaluation harness

Design for running the evaluation plan (`docs/03-evaluation-plan.md`) against
the pinned checkpoint (`docs/00-pinned-checkpoint.md`) on 4 x CMP 170HX.

**Status: untested.** Every command below is a design sketch; none has been
executed on the target platform. Update this file with measured reality after
the first leased run.

## Design

- One runnable cell = one entry in `eval/configs/matrix.yaml`. All cells are
  `status: untested` until a manifest exists for them.
- Every run produces exactly one manifest conforming to
  `eval/configs/run-manifest.schema.json`, including the outcome class
  (`infrastructure` / `compatibility` / `capacity` / `stability` /
  `quality`) and a claim label. Failed launches get manifests too — negative
  results are first-class.
- Metrics come from the runtime's own logs and the final usage object of each
  request (never stream-event counts), plus `nvidia-smi` sampling for VRAM,
  power, and thermals.
- Raw outputs (usage JSON, truncated logs, score tables) are stored out of
  band; the manifest references them by path/URI. `raw_output_references`
  must be non-empty.

## Commands (UNTESTED — do not run until gates clear)

```bash
# 0) Gates first: README.md upstream + resource gates, scripts/preflight-checklist.md
#    Checkpoint download, builds, and GPU use are prohibited before both gates clear.

# 1) Fetch the pinned checkpoint into the model library (NOT this repo)
#    hf download cyankiwi/Qwen3.8-Flash-Next-AWQ-INT4 \
#      --revision d39638a0e740fccb3e24ae0ea5cab34c15371ae6

# 2) Launch the runtime per the matrix cell (exact flags TBD post-gate;
#    requires the vllm#53899 gate from docs/02-runtime-compatibility.md)

# 3) Run the task suites with pinned seeds; write eval/results/<run_id>.manifest.json

# 4) Validate the manifest
#    check-jsonschema --schemafile eval/configs/run-manifest.schema.json \
#      eval/results/<run_id>.manifest.json
```

## Repository hygiene

- **No model weights, no shards, no large binaries in this repository** —
  weights live only in the configured model library.
- Keep everything plain text under 300 KB; raw outputs referenced, never
  committed wholesale.
- Every committed number keeps its evidence label
  (`measured` / `inferred` / `community-reported` / `untested`).
