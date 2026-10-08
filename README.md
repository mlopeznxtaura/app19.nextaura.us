# app19: an app18-only model that lives on one spare 4 GB card

**Status:** plan. The only thing live today is this page at <https://app19.nextaura.us>. Nothing in this repo serves or trains a model yet.

## What

One Windows PC runs two jobs:

- **app18.nextaura.us**: the public site in front of a **Blender 5.2.2** modeling and render node.
- **app19**: a small LLM that is an expert in exactly one thing, driving app18. It turns a request into the right app18 jobs, reads the results, and fixes its own mistakes. Every night it gets better from app18's own job logs.

| Part | What it is |
| --- | --- |
| CPU | AMD Ryzen 9 7950X (16 cores) |
| GPU A, the **spare card** | AMD FirePro W7000, 4 GB, GCN 1.0 (Pitcairn). **The model lives here and only here.** |
| GPU B | AMD FirePro W7000, 4 GB. **Dedicated to Blender.** The model never touches it. |
| OS | Windows 11 Pro |
| Blender | 5.2.2. Cycles GPU (HIP) does not support GCN 1.0, so Cycles renders on the CPU and EEVEE/Workbench use GPU B. |

## The hard rule

**The model never spills into system RAM.** Weights, KV cache and compute buffers all stay in the spare card's 4 GB of VRAM. The OS reports **3,452 MiB free** on that card (measured).

How the rule is enforced, all detailed in [docs/vram-budget.md](docs/vram-budget.md):

1. **llama.cpp's Vulkan backend.** It is the only option on this card: there is no CUDA, and ROCm/HIP does not support GCN 1.0.
2. **Everything on the card.** Full offload (`-ngl 999`), the token embedding forced onto the card (`-ot token_embd.weight=Vulkan0`), no memory-mapped model file, KV cache on the GPU, fixed context and batch sizes, and no automatic resizing.
3. **Card pinning.** The server only sees the spare card, selected by Vulkan device index plus a LUID check (LUID is Windows' per-boot GPU adapter ID). Blender's card is invisible to it.
4. **Preflight check.** It adds up the real file sizes, KV cache, compute buffers and driver overhead, and refuses to start if the total is over the **3,200 MiB** ceiling.
5. **Windows Job Object.** It caps the server process's memory, so a runaway allocation kills the process instead of paging.
6. **Watchdog.** It reads the Windows GPU counters (dedicated vs shared usage) for the server process every 500 ms and kills and restarts the server on any spill.

## Chosen model

**Qwen2.5-Coder-1.5B-Instruct, Q4_K_M**, with context 8192, micro-batch 256 and llama.cpp **b8393** on Vulkan.

| Measured on the spare W7000 (b8393, `--no-mmap`, ctx 8192, ub 256) | MiB |
| --- | --- |
| Model buffer, all on the card (GGUF file is 1,117,320,768 B) | 1,059.89 |
| KV cache, f16 | 224.00 |
| Compute buffer on the card | 151.38 |
| llama.cpp total on the card | **1,435.27** |
| Windows "Dedicated Usage" peak for the process (includes driver overhead) | **1,461** |
| Room left for a LoRA adapter, a second adapter for instant rollback, and headroom under the 3,200 MiB ceiling | ~1,739 |

Generation speed is about **43 tokens/s**. The full table of candidates (0.5B, 1B, 3B), with sources, is in [docs/vram-budget.md](docs/vram-budget.md).

### The driver problem, plainly

The W7000s run AMD's **last** driver for this generation: 27.20.21026.6 from June 2021, with a Vulkan ICD (the driver's Vulkan implementation) reporting **1.2.170**. llama.cpp's Vulkan needs 1.2 plus 16-bit storage buffers. Both cards pass that check: `--list-devices` shows them with `fp16: 0` and `int dot: 0`.

The catch: **llama.cpp builds after b8393 produce corrupted text on this driver.** The tested build b11512 doubled words ("and and", "These These"). Upstream declined a fix ([llama.cpp PR #21787](https://github.com/ggml-org/llama.cpp/pull/21787)) and told users to switch to the Linux open-source driver (Mesa RADV).

b8393 produced clean output in the same test. **So Phase 1 pins b8393.** Fallbacks are listed in [docs/vram-budget.md](docs/vram-budget.md#if-the-driver-path-fails).

## The learning loop, in one paragraph

Every app18 job (the request, the model's plan, the API calls, Blender's result, pass/fail) is logged as a training example. Logs are stored **only on a separate Linux storage box**. Updating weights in real time is not possible on this card. A **nightly LoRA adapter** is possible, but only trained **off the card**.

The honest verdict on training *on* the 4 GB card without using RAM: **upstream llama.cpp cannot fine-tune the 1.5B or even a 0.5B model inside 4 GB.** Its trainer is full-parameter and FP32 only, which limits it to roughly a 150M-parameter model with AdamW. See [docs/learning-loop.md](docs/learning-loop.md) for the numbers and for the alternatives, each labelled as violating or not violating the rule.

A new adapter replaces the old one **only** if it beats it on a held-out set of app18 tasks scored against Blender's reference output. Rollback is instant: both adapters stay loaded and a single call switches between them.

## Security

- The model reaches app18 **only** with short-lived, per-session tokens that Grok Bot mints. There are no hardcoded tokens, nothing long-lived, and nothing in this repo or on this site.
- The model never holds a credential. A local tool proxy attaches the session token and allows only an approved list of app18 endpoints.
- No extra Cloudflare services. This site is a static Worker that serves files only. app18 keeps its existing setup.
- The PC, the Blender node and the storage box have no new public exposure.

## Plan docs

| Doc | What's in it |
| --- | --- |
| [docs/vram-budget.md](docs/vram-budget.md) | Budget table with real GGUF sizes and on-card measurements, server flags, preflight, Job Object, watchdog, card pinning, driver fallbacks |
| [docs/learning-loop.md](docs/learning-loop.md) | Logging, the storage box, the nightly adapter, the on-card training verdict and alternatives, eval gate, rollback |
| [docs/phases.md](docs/phases.md) | Phase 1: inference + VRAM enforcement. Phase 2: logging. Phase 3: nightly adapter + eval gate |
| [docs/app9-builder.md](docs/app9-builder.md) | app9.nextaura.us as the agent-ready building GUI: JSON API, MCP-style tools, per-session tokens |

## Deploy this site

```
npm ci
npm run deploy
```

This builds the markdown into `site/` and runs `wrangler deploy` on Node 22. Any API-token environment variables are unset first, so the existing Cloudflare login is used. The Worker serves static files only.

## License

MIT. See [LICENSE](LICENSE).
