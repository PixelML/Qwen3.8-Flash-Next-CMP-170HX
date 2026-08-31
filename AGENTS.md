# Repository instructions for agents

Public repository. Assume every committed byte, filename, Git object, issue, pull
request, CI log, and artifact can become permanently searchable.

## Scope

This is the canonical home for **every** Qwen3.8-Flash-Next attempt on
CMP 170HX (SM80): AWQ INT4, NVFP4, GPTQ, EXL3, FP8/BF16, alternative sizes,
conversions, runtime ports, successful benchmarks, incompatibilities, and
negative results. One repository per model family/platform — never per
quantization, checkpoint, runtime, or machine. Do not create separate
`...-AWQ`, `...-NVFP4`, or runtime-specific CMP repositories; future
quantizations and runtimes land in this repository.

The DGX reference repositories remain the DGX reference implementations; CMP
results never belong there.

## Publication boundary

Never publish:

- passwords, tokens, cookies, API keys, private keys, credential filenames, or secret-manager paths;
- private or Tailscale IPs, public rental IPs, hostnames, MAC addresses, serial numbers, physical locations, or exact PCI maps tied to private infrastructure;
- account IDs, instance IDs, billing details, purchase records, vendor conversations, customer data, private prompts, or personal information;
- unredacted logs, shell history, environment dumps, SSH configuration, `.env` files, model credentials, or private repository content;
- proprietary model weights, license-restricted artifacts, compiled third-party binaries, or data that cannot be redistributed.

Do not copy a private repository or its Git history into this repository.
Reconstruct public documentation from verified facts. Use generic labels such as
"four-card CMP node" and "shared model storage." Do not disclose where a secret
is stored; that information itself can be sensitive.

If a value is not necessary for reproducing the result, omit it. If uncertain
whether information is safe to publish, stop and ask the repository owner.

The base model and each quantization carry their own license terms; obey them,
and record which license applies to each checkpoint you document (see
`docs/00-pinned-checkpoint.md`).

## Evidence rules

- Label statements as **measured**, **inferred**, **community-reported**, or **untested**.
- Link primary sources for external technical claims.
- Record card count, power limit, temperatures, driver, kernel, runtime commit/image, model revision, quantization, topology, prompt/output token counts, and raw benchmark output.
- For streaming APIs, derive generated tokens from the final usage object, never from stream-event counts.
- Preserve negative results. State exactly what was tested and why it failed; do not overstate what a single failure rules out.
- Never invent a measurement, version, citation, or successful test.
- Never paste an entire quantization ignore list or other bulk generated content into a document; link or summarize instead.

## Infrastructure safety

By default, agents may inspect files and run read-only checks. They must not
start or stop VMs, reboot or power-cycle hosts, change passthrough, change GPU
power limits, flash firmware, install drivers, build software, download model
weights, or launch GPU workloads without explicit user authorization for that
action.

For this repository, checkpoint download, software builds, and any GPU use are
additionally prohibited until the upstream gate clears (see `README.md`) and an
exclusive four-CMP lease is granted in `seanphan/pixelml#52` after
`seanphan/pixelml#57` is resolved. Do not request the lease before the
decision gate in `docs/02-runtime-compatibility.md` is met.

For any later authorized GPU work: confirm forced airflow before load; stop at
80 C core or 85 C memory temperature; stop on Xid errors, GPU disappearance,
memory errors, or unsafe storage conditions; run destructive memory tests only
on GPUs not serving other workloads; store model weights only in the configured
model library, never in the repository.

## Repository hygiene

- Keep every file plain text, under 300 KB, with no binaries, symlinks, or model weights.
- Use Conventional Commits. Work on feature branches; open pull requests; never push directly to `main`.
- Do not add AI attribution to commits, pull requests, or issues.
