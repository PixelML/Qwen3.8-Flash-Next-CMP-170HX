# Preflight checklist — Qwen3.8-Flash-Next AWQ INT4, four-CMP node

Run top to bottom before every leased session. Any unchecked box = no run.
Post-run section is mandatory before releasing the lease.

## 0. Lease and gate check (hard blockers)

- [ ] The preceding workload is resolved/cleared in internal tracking (record date).
- [ ] Lease order verified against the current private control-plane policy;
      record release evidence for the preceding workload before this lease.
- [ ] Exclusive **four-CMP** lease granted in the private control plane for
      this exact window; no other workload scheduled on the node.
- [ ] Upstream decision gate met per `docs/02-runtime-compatibility.md`
      (PR #53899 merged / rebased / pinned branch with SM80 AWQ/GDN/PLE
      evidence). Re-measure PR state, `mergeable_state`, head SHA today and
      record it.
- [ ] The lease window, cards, and purpose are recorded in internal tracking
      before starting.

## 1. Storage check

- [ ] Model library has ≥ **200 GiB free** (checkpoint is a measured
      **175.36 GiB** across 38 shards, plus scratch for logs/outputs).
- [ ] Target filesystem healthy: no errors in `dmesg`, adequate inode/file
      limits, write test passes.
- [ ] Weights will live ONLY in the configured model library — never inside
      the repository, never in `$HOME` sprawl.
- [ ] Checkpoint revision verified after download:
      `d39638a0e740fccb3e24ae0ea5cab34c15371ae6` and total byte size matches
      **188,286,106,928 bytes**.

## 2. GPU health (read-only checks first)

- [ ] `nvidia-smi` lists all four cards, correct IDs, no `Xid` in
      `dmesg`/event log.
- [ ] ECC errors: current volatile + aggregate counters at baseline; record
      them. Any uncorrected error = stop.
- [ ] Persistence mode on; no stale processes holding memory
      (`nvidia-smi --query-compute-apps=`).
- [ ] Idle power and clocks look symmetric across all four cards (a silent
      "runt card" fails TP4 — see `docs/01-vram-ram-fit.md`).
- [ ] `nvidia-smi -q -d PERFORMANCE` shows no throttling reasons at idle.

## 3. Driver and runtime

- [ ] Driver version recorded in the run manifest (same value on all cards).
- [ ] Runtime container/commit matches the pinned gate reference
      (vLLM PR #53899 head `935728b4a95e110d91a41ab4e02b6bed06ec66ab` or the
      documented successor).
- [ ] PLE CPU offload env vars set per upstream docs; host RAM ≥ budget from
      `docs/01-vram-ram-fit.md` (≥ 128 GiB planned; 1.25x overhead assumed
      until measured).
- [ ] Harness revision (this repo's commit) recorded.

## 4. Thermals and airflow

- [ ] Forced airflow confirmed before any load (fans respond, intake/exhaust
      unobstructed).
- [ ] Idle temperatures recorded per card (core + memory).
- [ ] Temperature sampling loop running (e.g. 1 Hz `nvidia-smi` logger)
      before first request.

## 5. Stop conditions (any one = immediate stop)

- [ ] Core temperature ≥ **80 C** on any card.
- [ ] Memory temperature ≥ **85 C** on any card.
- [ ] Any **Xid error**, GPU disappearance, or uncorrected ECC error.
- [ ] Host RAM exhaustion or swap thrash (PLE offload starvation).
- [ ] Storage full or write errors.
- [ ] Card power/thermal asymmetry suggesting a failing card.
- After a stop: dump logs, thermals, and `dmesg` tail into the run record;
  classify the run (`infrastructure` / `capacity` / `stability`) in the
  manifest; file the negative result.

## 6. Post-run release steps (mandatory, in order)

- [ ] Serving processes stopped; `nvidia-smi --query-compute-apps=` empty.
- [ ] Final thermals and power recorded; cards back near idle baselines.
- [ ] Run manifest completed and validated against
      `eval/configs/run-manifest.schema.json`; raw outputs referenced.
- [ ] Repository still contains **no weights and no large files**.
- [ ] Lease released in the private control plane with a short outcome note;
      internal tracking updated with the manifest link.
