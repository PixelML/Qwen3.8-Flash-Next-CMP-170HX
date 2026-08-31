# 11 — Intel RTN W4A16 prefetch receipt (sanitized)

Status: TRANSFER_COMPLETE_PASS. One resumable lane, shared storage only.
No build, no model load, no GPU use. Compute remains gated on an exclusive
four-card lease after a fresh live preflight.

## Pinned transfer (2026-08-31)

- Candidate: `Intel/Qwen3.8-Flash-Next-W4A16-RTN-AutoRound`
  @ `a729382b72baabf11a6c10f35e9042b98cc06ef3`
- Destination: `/library/models/qwen38/Intel-Qwen3.8-Flash-Next-W4A16-RTN-AutoRound`
  (canonical shared model storage; guest root never used for weights)
- Trigger: GLM lane integrity PASS + bounded boot start, recorded in internal
  tracking.

## Verification gates (all measured)

| Gate | Result |
| --- | --- |
| Pre-flight | shared storage read-write OK, 7.9 TiB free, 0 reusable Intel bytes present |
| File inventory | 146/146 files exact-size match vs Hub tree manifest |
| Byte total | 181,243,078,643 / 181,243,078,643 B (exact) |
| Safetensors | 131/131 shards, 181,195,588,648 B (exact) |
| LFS SHA256 | 133/133 pinned files OK (all 131 shards + 2 large non-shard files) |
| Guest root impact | bounded transient ~1 GiB staging cache; released after |

## Lane telemetry (measured)

- Start 09:44:45Z, PASS 10:53:12Z on 2026-08-31: 68.5 min end-to-end.
- Download phase ~45 min (~80 MB/s sustained, 2 workers), hash-verify phase
  ~22 min (~20 s per 1.38 GiB shard).
- One lane only; no re-downloads; no resumable restarts needed.
- Storage I/O guard polled the active GLM lane every 60 s: zero measurable
  impact reported, zero pause events.
- Baseline remains untouched: cyankiwi AWQ INT4 (38/38 shards) reused, not
  re-downloaded.
