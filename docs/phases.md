# Phases

## Phase 1: inference + VRAM enforcement

**Goal:** the 1.5B model serves app18-only requests from the spare W7000, and it provably never touches system RAM.

- [ ] Pin llama.cpp **b8393** (Windows Vulkan x64 build). Record its hash.
- [ ] Card pinning: map slot → LUID → Vulkan index at each boot. `GGML_VK_VISIBLE_DEVICES` = spare card only.
- [ ] Preflight: budget math against the 3,200 MiB ceiling, a load-log check (no CPU model buffer), a ±5% check against the estimate, and the coherence prompt.
- [ ] Launcher with a Job Object (2,304 MiB commit cap, kill-on-close).
- [ ] Watchdog on the Windows GPU counters (shared and dedicated usage, Blender's card, working set). Kill and restart on a trip; stay stopped after 2 trips in an hour.
- [ ] Tool proxy: allowlist of app18 endpoints, attaches the per-session token that Grok Bot mints, and gives the model no credentials.
- [ ] System prompt + app18 tool schema. Prompt memory slot (empty for now).

**Done when:**

- A 24-hour soak passes with zero watchdog trips.
- Windows dedicated usage stays within 5% of 1,461 MiB, and shared usage stays at baseline.
- Blender renders on GPU B at the same time with no slowdown from the model.
- 20 scripted app18 jobs pass end to end.

## Phase 2: logging

**Goal:** every app18 job becomes a clean training record on the Linux storage box, and nowhere else.

- [ ] The tool proxy writes job records using the schema in [learning-loop.md](learning-loop.md).
- [ ] Scrubber: tokens, IPs, hostnames, usernames and paths are removed before a record leaves the PC. Unit tests cover each one.
- [ ] Append-only transfer to the storage box over an authenticated private link. A local spool holds records until receipt.
- [ ] Validator auto-labels each job `pass` or `fail`, and Marco can add his own label.
- [ ] Prompt memory: the top-k verified examples go into the context (it lives in the KV cache, on the card).
- [ ] Freeze the held-out eval set (~150 tasks) and run the gold jobs through Blender to get the reference outputs.

**Done when:** 7 days of jobs are on the storage box, the scrubber tests pass, and the eval set and Blender references are frozen.

## Phase 3: nightly adapter + eval gate

**Goal:** a nightly LoRA that only replaces the current one when it beats it on Blender-referenced app18 tasks, with instant rollback.

- [ ] 2-hour spike: the qvac-fabric LoRA trainer on the spare W7000. Record whether its output is clean and its peak usage. Expected outcome: blocked by the driver.
- [ ] Plan of record: LoRA training on the storage box's CPU from the current adapter, then convert to GGUF and hash.
- [ ] Eval harness: candidate vs current on the frozen set, with Blender executing the jobs. Scorecard as in [learning-loop.md](learning-loop.md#5-eval-gate-blender-is-the-reference).
- [ ] Gate: +1.0 point, no metric down more than 2, zero forbidden calls, coherence passes.
- [ ] Hot-swap through `/lora-adapters` scales, with current and previous both loaded. One-call rollback.
- [ ] Morning scorecard in app9.

**Done when:** 14 nights run unattended, at least one adapter promotes on merit, and a forced rollback takes under 1 second.

## Iteration rule (this repo)

One push per iteration, with an annotated tag counting up (`good-0`, `good+1`, …). The tag message states the goal, what changed, how it was verified, and the base commit. Existing tags are never moved.
