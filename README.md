# app19 — Local Character Model on the Spare Card

**Status:** plan. Nothing in this repo trains or serves a model yet. The only thing live is this page at <https://app19.nextaura.us>.

## What

Run a small open model with a LoRA adapter on the **spare GPU card**. You give it a plain-language character brief ("heavy-set cop, mustache, aviators, navy slacks"). It returns **validated `lowpoly_character` params**, the same JSON the Blender low-poly builder already accepts. Blender renders those params. That render is the **reference**. The model gets no other source of truth.

The nightly loop:

1. Train or refresh the adapter on the spare card.
2. Run the fixed eval set. Brief → params → Blender render.
3. Score each render against the Blender reference render for that brief.
4. Promote the adapter only if it beats the current one. If it doesn't, keep the current one. Rolling back is one action.

The builder GUI for all of this is **app9.nextaura.us**. See [docs/app9-builder.md](docs/app9-builder.md).

## Why

- The params schema is closed: enums, ranges, and `#RRGGBB` colours, and unknown keys are rejected. That makes "did the model get it right" a measurable question, not a vibe.
- Blender already produces a good character from good params (front, 3/4, and side views, plus a rigged GLB). The model only has to learn **brief → params**. It never has to learn geometry.
- The spare card is otherwise idle. The main card and the main workloads are never touched.

## Architecture

```
brief (text)
   │
   ▼
[spare card] base model + LoRA adapter ──► params JSON
   │                                          │
   │                              schema validator (reject = score 0)
   │                                          │
   │                                          ▼
   │                               Blender builder (headless)
   │                                          │
   │                         render views + GLB + stats.json
   │                                          │
   ▼                                          ▼
eval harness ◄──────── Blender reference renders (gold params)
   │
   ▼
scorecard ──► promote / hold / roll back   (driven from app9)
```

| Part | Runs where | Notes |
| --- | --- | --- |
| Base model + adapter | Spare GPU only | Pinned to that one device. Never falls back to the main card. |
| Schema validator | CPU | The same rules as the Blender builder. Unknown keys are rejected. |
| Blender builder | CPU (Cycles CPU) or the existing private Blender node | Reached only through the authenticated gate. Never exposed directly. |
| Eval harness | CPU | Deterministic seeds and a fixed camera, light, and resolution. |
| Control plane | app9.nextaura.us | Status, start/stop, logs, evals, and rollback. Session-token auth. |

## VRAM budget

The spare card has a **hard budget**. The runner refuses to start when the estimate is over budget, and it stops the job when the measured peak crosses the stop line.

```
estimate = weights + adapter + optimizer_state + activations(seq_len, batch) + kv_cache + runtime_overhead
headroom = card_total − estimate          (must stay ≥ 15% of card_total)
```

| Mode | Base size | Precision | Approx. weights | Training extra (LoRA r=16, seq 1024, batch 4, grad ckpt) | Fits a ~12 GB spare card? |
| --- | --- | --- | --- | --- | --- |
| Serve | 0.5B | bf16 | ~1.0 GB | — | Yes, with lots of room |
| Serve | 1.5B | bf16 | ~3.1 GB | — | Yes |
| Serve | 3B | 4-bit | ~2.0 GB | — | Yes |
| Train (QLoRA) | 1.5B | 4-bit base, bf16 adapter | ~1.0 GB | ~3–5 GB | Yes, the default |
| Train (LoRA) | 3B | bf16 | ~6.2 GB | ~5–7 GB | Tight. Only if headroom stays ≥ 15% |

Rules:

- **Default:** 1.5B base, QLoRA training, bf16 serving. Start at 0.5B to prove the loop, then move up.
- **Serve and train never overlap.** The scheduler stops serving before a training run and restarts it after.
- **Thresholds:** green below 70% of card total, amber 70–85%, red above 85%. At red, the runner stops the job cleanly and writes a checkpoint.
- The budget numbers (card total, thresholds, max seq/batch) live in a config file, not in code. app9 shows estimated vs measured usage live.

## Data

- **Seed set:** a handful of hand-checked character specs (for example a Belizean man, a Trinidadian woman, a police officer, and a street character), each with a written brief.
- **Expansion:** sample valid params from the schema (body, build, height, hair, top, bottom, shoes, colours, accessories ≤ 5). Write 3–5 briefs per spec in different voices: terse, descriptive, slang. Keep the briefs a human would actually type.
- **Gold renders:** Blender renders every gold spec once: front, 3/4, and side views at a fixed resolution and sample count. The image hashes are stored with the spec.
- **Split:** train / val / **frozen eval** (about 200 briefs). The frozen eval set never changes between nights, so the scores stay comparable.
- No scraped faces and no real people. Every character is synthetic.

## Training

- QLoRA on the spare card. The target is **params JSON only**, with no prose.
- Constrained decoding against the schema's JSON grammar at inference time, so invalid enums or keys can't be emitted.
- Nightly: train on whatever new data arrived since the last run, starting from the last *promoted* adapter, with a capped step count and a capped wall clock.
- Every adapter is saved as an immutable, content-hashed artifact with its config, data manifest, and scorecard.

## Nightly eval vs the Blender reference

For each frozen-eval brief:

| Metric | What it checks | Weight |
| --- | --- | --- |
| Schema-valid rate | Params pass the validator | Gate. Must be ≥ 99% |
| Field accuracy | Exact match on enums, ±0.03 m on height, ΔE ≤ 10 on colours vs gold params | 40% |
| Silhouette IoU | Mask IoU of front, 3/4, and side renders vs the reference renders | 25% |
| Palette distance | Mean ΔE of the dominant colours per body region vs the reference | 20% |
| Perceptual similarity | SSIM on the fixed-camera renders vs the reference | 15% |

**Promotion gate:** the candidate's weighted score must be at least the current adapter's score + 0.5 points, with no metric regressing by more than 2 points, and a schema-valid rate of ≥ 99%. If any check fails, the current adapter stays and the candidate is kept for comparison.

**Compare view:** side-by-side reference and candidate renders per brief, sorted by worst delta first. This lives in app9.

## Rollback

- `current` is a pointer to one adapter hash. Promotion and rollback both just move the pointer.
- The last 10 promoted adapters are kept. Rollback swaps the pointer and restarts serving on the spare card. The target is under 1 minute.
- Every pointer move is logged with who did it (Marco or an agent session), when, and why.

## Security

- **No secrets in this repo or on the site.** No tokens, keys, internal addresses, hostnames, usernames, or file paths.
- Control actions go through app9 with **short-lived per-session tokens** minted by Grok Bot for each session. Nothing is hardcoded and nothing is long-lived. See [docs/app9-builder.md](docs/app9-builder.md#auth).
- The GPU box and the Blender node are **never** exposed publicly. They're reached only through the authenticated gate.
- This site is a static Cloudflare Worker with assets only: no backend, no storage, and no other Cloudflare services.

## Milestones

| # | Milestone | Done when |
| --- | --- | --- |
| M0 | Plan public | This page is live at app19.nextaura.us |
| M1 | Eval harness | Gold renders exist for the seed set, and the scorer runs on CPU against hand-written params |
| M2 | Baseline | 0.5B base model with constrained decoding and no adapter is scored on the frozen eval |
| M3 | First adapter | A QLoRA adapter on the spare card beats the baseline through the promotion gate |
| M4 | app9 builder | Status, start/stop, logs, evals, compare, and rollback all work from app9 for Marco and for agents |
| M5 | Nightly | Unattended nightly train → eval → promote/hold, with a morning scorecard |

## Repo layout

```
README.md              this plan (rendered as the site)
docs/app9-builder.md   app9 builder GUI + agent API plan
site/                  built static site (generated)
scripts/build.mjs      renders the markdown into site/
wrangler.toml          static-assets Worker for app19.nextaura.us
```

## Deploy (site only)

```
npm ci
npm run build
npm run deploy
```

`npm run deploy` runs `wrangler deploy` (Node 22) with your existing Cloudflare login. Any API-token environment variables are unset first so the login is used.
