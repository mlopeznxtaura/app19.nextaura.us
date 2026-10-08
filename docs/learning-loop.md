# Learning loop

The model gets better at app18 from app18's own job logs. Blender's output is the reference for what is correct.

```
request ─► model (spare W7000) ─► tool proxy ─► app18 API ─► Blender node ─► result
                                     │                                         │
                                     └──────── job record (scrubbed) ◄─────────┘
                                                      │
                                         Linux storage box (only copy)
                                                      │
                                    nightly: build dataset ─► train adapter ─► eval vs Blender
                                                      │
                                     gate passes? ─► hot-swap adapter on the card
                                     gate fails?  ─► keep current, file the report
```

## 1. Logging (Phase 2)

Every app18 job the model drives becomes one JSON line. The **tool proxy** writes the record, not the model, so the model cannot edit its own history.

| Field | Content |
| --- | --- |
| `job_id`, `ts` | IDs and time |
| `request` | What was asked, in plain language |
| `prompt_version`, `adapter` | Which system prompt and which adapter hash produced the plan |
| `plan` | The model's output: the app18 calls and their arguments |
| `calls[]` | Each app18 request and response (status, body summary, timing) |
| `blender` | Blender's result: mesh/rig stats, render hashes, export file sizes, errors |
| `outcome` | `pass` / `fail` from the validator. Optional `human` label from Marco. |
| `fix` | If the model retried, the corrected plan. These are the most valuable training pairs. |

**Scrubbing happens before anything leaves the PC.** Auth headers and session tokens are dropped, and so are IPs, hostnames, usernames and local file paths. Only the job content stays.

**Storage happens only on the separate Linux storage box.** Records are append-only, sent over an authenticated private link. Nothing is stored on Cloudflare, in this repo, or in any third-party service. The PC keeps a small spool only until the box confirms receipt.

## 2. Why not real-time learning

Updating weights after every job is not feasible on this card. Even inference uses a quantized (Q4_K_M) model, and gradient steps on it cannot run on the card within the no-RAM rule (see section 3). What *can* change in real time without training is **prompt memory**: a small, curated set of recent verified `request → plan` examples that the server injects into the context. It lives in the KV cache on the card and counts against the 8,192-token context.

Weight changes happen **nightly**, as a LoRA adapter.

## 3. Can training run on the 4 GB card without spilling into RAM?

**Verdict: no, not with upstream llama.cpp on this driver. Not for the 1.5B model, and not even for a 0.5B model.**

**Two blockers, either one enough:**

1. **Missing GPU ops in the only build that works here.** b8393, the last build that gives clean output on this driver, has no Vulkan `OUT_PROD` or `CROSS_ENTROPY_LOSS` / `_BACK`. llama.cpp's trainer needs them, so they would run on the CPU, which means RAM. Vulkan `OUT_PROD` only landed on 2026-07-15 (llama.cpp #23997), and that is only in builds that corrupt output on this driver.
2. **Memory, even on a fixed driver** (for example Linux + Mesa RADV). Upstream `llama-finetune` is full-parameter only, needs FP32 weights, and never trains the token embedding.

| Model | Weights f32 | Trainable (non-embedding) | + grads | + AdamW moments | Activations (ctx 512) | Total vs 3,200 MiB ceiling |
| --- | --- | --- | --- | --- | --- | --- |
| Qwen2.5-0.5B, SGD | 1,885 MiB | ~358M | 1,366 MiB | — | ~600 MiB | **~3,850 MiB: does not fit** |
| Qwen2.5-0.5B, AdamW | 1,885 MiB | ~358M | 1,366 MiB | 2,731 MiB | ~600 MiB | **~6,580 MiB: does not fit** |
| Qwen2.5-Coder-1.5B, any | 5,880 MiB | ~1.31B | — | — | — | **Weights alone do not fit** |

**The concrete limit, with a working driver:** full fine-tune of about **150M parameters with AdamW** (16 B per trained parameter) or about **300M with SGD** (8 B per parameter), at context 512. That is small-model territory (for example a 135M model). It is not the 1.5B model that serves app18.

**LoRA on the card:** upstream removed its LoRA trainer in November 2025. The [qvac-fabric](https://huggingface.co/blog/qvac/fabric-llm-finetune) fork does LoRA on a quantized base on Vulkan. On paper it fits: 1.06 GiB base + ~35 MiB rank-8 attention adapter with gradients and Adam state + ~0.6–0.9 GiB activations ≈ 2 GiB. **But** it tracks upstream, so it carries the synchronization change that corrupts output on this driver, and it is verified on Qwen3/Gemma3 only. **Status: unproven, likely blocked.** Phase 3 gives it one 2-hour spike; it is not the plan of record.

### Alternatives, labelled

| Option | Violates the no-RAM rule? | Notes |
| --- | --- | --- |
| **Train the LoRA off the PC, on the Linux storage box (CPU, PEFT on the full-precision weights), ship only the adapter file** | **No.** The served model never leaves the card. Training RAM is on a different machine. | **Plan of record.** The adapter is converted to GGUF and loaded on the card next to the base. ~37–74 MiB is budgeted. |
| Prompt memory: retrieve verified examples, no training | **No** | Ships in Phase 2. Works even if training never does. |
| qvac-fabric LoRA on the spare W7000, at night | No, if it works | Expected to hit the driver bug. Time-boxed spike only. |
| Linux + RADV on this PC, then on-card training of a ≤150M model | No | Too small to replace the 1.5B for driving app18. Useful only as a router or classifier. |
| Train at night on GPU B (Blender's card) | No (VRAM only), but it **breaks "GPU B is Blender's"** | Same 4 GB limit and same driver bug, so it gains nothing |
| Train on the Windows PC's CPU and RAM (64 GB) | **Yes, violates** | Puts model state in this PC's RAM |
| Partial offload or CPU fallback during training | **Yes, violates** | The same thing, hidden |

## 4. Nightly adapter (Phase 3)

1. **02:00 PT:** the storage box builds the dataset. It takes records since the last run where `outcome = pass` or a human-approved `fix`, de-duplicates them, and holds out anything that matches the eval set.
2. **Train** a LoRA (rank 16, all linear layers) on the storage box's CPU, starting from the current promoted adapter. Step count and wall-clock time are capped.
3. **Convert** to a GGUF LoRA and hash it.
4. **Eval** (below). If the gate passes, **promote** by moving the `current` pointer.

## 5. Eval gate (Blender is the reference)

- **Held-out set:** ~150 app18 tasks, frozen, split across job types: build a character, edit one, rig, turntable render, GLB export. Each task has a **gold job** that has already been run through Blender. Those outputs (mesh and rig stats, renders, export sizes) are the reference.
- **Run:** the candidate and the current adapter each drive app18 on every task, using an eval-only session token. Blender actually executes the jobs, at night, on GPU B and the CPU.

| Metric | Weight |
| --- | --- |
| Task success (job completes and the validator passes) | 40% |
| Match to the reference: mesh/rig stats within tolerance, silhouette IoU, palette ΔE, SSIM of renders | 35% |
| Plan correctness vs the gold job (endpoint and arguments exact-match rate) | 15% |
| Efficiency (retries, calls per task) | 10% |
| Forbidden or out-of-allowlist calls | **Gate: must be 0** |
| Coherence check on the card | **Gate: must pass** |

**Promote only if:** the candidate's score is at least the current score + 1.0 point, no metric regresses by more than 2 points, and both gate checks pass. Otherwise the current adapter stays and the report is filed.

## 6. Rollback, instantly

- `llama-server` starts with **both** the current and the previous adapter loaded (`--lora-init-without-apply`). They cost ≤ 150 MiB on the card in total.
- **Promote or roll back** = one local `POST /lora-adapters` call that sets the scales (new 1, old 0, or the reverse). No restart and no reload, so it takes well under a second.
- The last 10 promoted adapters stay on the storage box. Rolling back further loads the older adapter at the next restart after preflight.
- Every pointer move is logged with who did it (Marco or an agent session), when, and why.
