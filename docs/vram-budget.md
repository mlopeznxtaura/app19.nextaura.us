# VRAM budget and enforcement

**The rule:** weights, KV cache and compute buffers live in the spare W7000's VRAM and nowhere else. If the model would spill into system RAM, it does not start. If it starts spilling while running, it is killed.

## What the card actually offers

| Fact | Value | How we know |
| --- | --- | --- |
| Card | AMD FirePro W7000, GCN 1.0 (Pitcairn), 4,096 MiB GDDR5 | Windows device list on the PC |
| Driver | 27.20.21026.6, dated 2021-06-01. This is AMD's last driver for GCN 1.0. | Windows device list |
| Vulkan ICD (the driver's Vulkan implementation) | API 1.2.170 | The driver's own Vulkan manifest |
| Free VRAM reported to llama.cpp | **3,452 MiB** of 4,096 | `llama-cli --list-devices` on the PC (b8393 and b11512 agree) |
| llama.cpp Vulkan device flags | `uma: 0`, `fp16: 0`, `bf16: 0`, `warp size: 64`, `int dot: 0`, `matrix cores: none` | Same command |
| CUDA / ROCm / HIP | Not available: AMD hardware, and GCN 1.0 is not supported by ROCm | — |

So the backend is **llama.cpp Vulkan**. Its hard requirements, from the source, are a Vulkan 1.2 instance and `storageBuffer16BitAccess`. Both W7000s pass them: they enumerate as llama.cpp devices without the `Unsupported device` error.

### Which llama.cpp build: pin b8393

| Build | Date | Result on the W7000 (temperature 0, same prompt) |
| --- | --- | --- |
| **b8393** | 2026-03-17 | Clean text. 43 tokens/s generation (1.5B Q4_K_M). |
| b11512 | 2026-10-08 | **Corrupted**: doubled words ("and and", "These These", "used used"). 41 tokens/s. |

The corruption is a known incompatibility between llama.cpp's newer Vulkan synchronization (from b8394 on) and AMD's final legacy driver (Vulkan 1.2.170). Upstream declined a compatibility patch and pointed users to the Linux open-source driver, Mesa RADV ([PR #21787](https://github.com/ggml-org/llama.cpp/pull/21787)). **Phase 1 pins b8393.** Any upgrade must pass the same coherence check first.

### If the driver path fails

| Fallback | Keeps the no-RAM rule? | Cost |
| --- | --- | --- |
| Stay on b8393 and backport only safe fixes | Yes | Frozen engine; no new model architectures |
| Linux on this PC with Mesa RADV, which supports GCN 1.0 through amdgpu (dual-boot, or a dedicated boot for the model) | Yes. Same 4 GB math. | Blender and app18 must move or share the boot. Known GCN 1.0 RADV quirks (`RADV_DEBUG=novm,syncshaders`, `-ub 256`) |
| Mesa "Dozen" (Vulkan on top of D3D12) | Yes, in theory | Immature for compute. Untested on this card. |
| CPU inference on the Ryzen | **No, violates the rule** | Uses system RAM by definition |
| Partial offload (`-ngl` < all layers) | **No, violates the rule** | The remaining layers run from RAM |

## Budget

Ceiling: **3,200 MiB** on the card. That is the 3,452 MiB reported free minus 252 MiB of safety margin, and it is tighter than the ~3.6 GB first proposed because the driver itself reserves about 644 MiB.

```
total = model_buffer + lora_adapters + kv_cache(ctx) + compute_buffer(ubatch) + driver_overhead
start only if total ≤ 3,200 MiB
```

KV cache size per token in f16 is `2 × layers × kv_heads × head_dim × 2 bytes`, taken from each model's `config.json`.

| Model (GGUF) | File size (source) | Model buffer | KV @ 4k | KV @ 8k | Compute (ub 256) | Driver overhead | **Total @ 8k** | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Qwen2.5-0.5B-Instruct Q4_K_M | 491,400,032 B / 468.6 MiB ([Qwen/Qwen2.5-0.5B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF)) | 462.96 (measured) | 48 | 96 (measured) | 150.12 (measured) | 37 (measured) | **746 (measured)** | Fits easily. Weaker reasoning. |
| **Qwen2.5-Coder-1.5B-Instruct Q4_K_M** | 1,117,320,768 B / 1,065.6 MiB ([Qwen/Qwen2.5-Coder-1.5B-Instruct-GGUF](https://huggingface.co/Qwen/Qwen2.5-Coder-1.5B-Instruct-GGUF)) | 1,059.89 (measured) | 112 (measured) | 224 (measured) | 151.38 (measured) | 26 (measured) | **1,461 (measured)** | **Chosen.** Code- and JSON-strong, with about 1.7 GiB of headroom. |
| Llama-3.2-1B-Instruct Q4_K_M | 807,694,464 B / 770.3 MiB ([bartowski/Llama-3.2-1B-Instruct-GGUF](https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF)) | ~765 (est.) | 128 | 256 | ~130 (est.) | ~30 | **~1,180 (est.)** | Fits. Weaker at code than Qwen-Coder. |
| Llama-3.2-3B-Instruct Q4_K_M | 2,019,377,696 B / 1,925.8 MiB ([bartowski/Llama-3.2-3B-Instruct-GGUF](https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF)) | ~1,918 (est.) | 448 | 896 | ~180 (est.) | ~30 | **~3,020 (est.)** | Under the ceiling by only ~180 MiB. No room for two adapters. **Rejected** (at 4k it would be ~2,580). |

"Measured" means llama.cpp b8393's own allocation log plus the Windows `GPU Process Memory\Dedicated Usage` peak, from a run on the spare W7000 with the flags below. Other measured points for the chosen model:

| Context | Model + KV + compute (llama.cpp log) | Windows dedicated peak |
| --- | --- | --- |
| 4,096 | 1,059.89 + 112 + 151.38 = 1,323.27 MiB | 1,349 MiB |
| **8,192 (default)** | 1,059.89 + 224 + 151.38 = 1,435.27 MiB | **1,461 MiB** |
| 16,384 | 1,059.89 + 448 + 211.50 = 1,719.39 MiB | 1,746 MiB |

Adapter allowance: a rank-16 LoRA on all linear layers of the 1.5B model is about 18.5M parameters, which is ~37 MiB in f16 or ~74 MiB in f32. Two adapters stay loaded (current and previous) for instant rollback, so ≤ 150 MiB. **Default plan total: ~1,610 MiB, about half the ceiling.**

## Server flags (b8393)

```
GGML_VK_VISIBLE_DEVICES=<spare card index>     # the server sees ONLY the spare card; Blender's card is hidden
GGML_VK_DISABLE_HOST_VISIBLE_VIDMEM=1          # plain device-local VRAM only
# never set GGML_VK_ALLOW_SYSMEM_FALLBACK or GGML_VK_PREFER_HOST_MEMORY

llama-server -m qwen2.5-coder-1.5b-instruct-q4_k_m.gguf \
  -dev Vulkan0 -ngl 999 -ot "token_embd\.weight=Vulkan0" \
  --no-mmap -c 8192 -b 256 -ub 256 -np 1 -fa off -fit off \
  -ctk f16 -ctv f16 \
  --lora-init-without-apply --lora <current.gguf> --lora <previous.gguf> \
  --host 127.0.0.1 --port <local port>
```

| Flag | Why |
| --- | --- |
| `-ngl 999` | Every layer on the card |
| `-ot token_embd.weight=Vulkan0` | llama.cpp normally keeps the input embedding on the CPU. This moves it to the card. The load log must show **no** CPU model buffer line. |
| `--no-mmap` | The model file is not memory-mapped, so the OS cannot keep weight pages in RAM. (Builds after b8393 renamed this to `--load-mode none`.) |
| KV offload (default on, never `-nkvo`) | KV cache on the card |
| `-c 8192 -b 256 -ub 256 -np 1` | Fixed context, batch and slots, so the memory size never changes after start |
| `-fit off` | Stops llama.cpp from quietly resizing anything |
| `-fa off` | Measured configuration. GCN 1.0 has no fast fp16 path. |
| `GGML_VK_VISIBLE_DEVICES` + `-dev Vulkan0` | Pins the model to one card by index. Without it, llama.cpp also opens a context on the other cards. |

In ggml-vulkan (b8393 and later), a device buffer that does not fit **fails**. It falls back to host memory only if `GGML_VK_ALLOW_SYSMEM_FALLBACK` is set, which we never do. An over-budget load therefore errors out instead of silently spilling.

**What does stay in host RAM, measured and named:**

- the 0.58 MiB logits output buffer and the 6–18 MiB host compute staging buffer (I/O only, no weights or KV)
- the executable, the Vulkan driver and the tokenizer: working set 195–283 MiB

## Card pinning

Both cards report the same name ("AMD FirePro W7000"), so a name match is not enough.

1. At every boot, preflight lists the Vulkan devices and reads each one's `deviceLUID` (Vulkan 1.1 ID properties). This is the same LUID Windows uses in the GPU performance counters.
2. The spare card is recorded once by its physical slot. Preflight maps slot → LUID → Vulkan index for this boot, then sets `GGML_VK_VISIBLE_DEVICES` to that index.
3. If the mapping is ambiguous, or the chosen LUID has Blender's process on it, preflight refuses to start.

## Preflight (refuses to start over budget)

1. Check the build is b8393 or a build that passed the coherence check. Refuse otherwise.
2. Resolve the spare card (above) and read its free VRAM. Refuse if it is under 3,452 MiB minus 64 MiB, which means something else is already on the card.
3. Compute `total` from the real GGUF sizes, the KV formula at the configured context, and the adapter sizes, plus the compute and driver allowance for this config (table above). Refuse if it is over 3,200 MiB.
4. Start the server inside the Job Object and parse the load log. Refuse, and kill the server, if any buffer line names a CPU or host model buffer, or if the Vulkan0 model + KV + compute total differs from the estimate by more than 5%.
5. Run a 48-token coherence prompt at temperature 0. Refuse if it repeats words.

## Windows Job Object cap

A small launcher creates a Job Object and starts `llama-server` suspended inside it. It sets:

- `JOB_OBJECT_LIMIT_PROCESS_MEMORY`, a commit cap of **2,304 MiB** for the default config (budget + 512 MiB)
- `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`, so the server dies if the launcher dies
- `JOB_OBJECT_LIMIT_DIE_ON_UNHANDLED_EXCEPTION`

Why the cap is above the VRAM figure: Windows (WDDM) charges **commit** for VRAM allocations so it can evict them in an emergency. Measured commit was 978 MiB (0.5B at 8k), 1,671 MiB (1.5B at 4k), 1,814–1,836 MiB (1.5B at 8k) and 2,080 MiB (1.5B at 16k). That is commit charge, not resident RAM; the working set stayed at 195–283 MiB. The cap means any real host-side growth, such as a CPU fallback or a mapped model file, hits the limit and kills the process instead of paging.

## Watchdog (kill on any spill)

The watchdog samples the Windows counters for the server's process ID on the spare card's LUID every 500 ms:

| Counter | Baseline (measured) | Trip |
| --- | --- | --- |
| `\GPU Process Memory(pid_*_luid_<spare>_phys_0)\Shared Usage` | 10–28 MiB | > baseline at start + 32 MiB, or > 64 MiB, for 2 samples in a row |
| `\GPU Process Memory(pid_*_luid_<spare>_phys_0)\Dedicated Usage` | 1,461 MiB (default config) | Drops by more than 64 MiB while running, meaning the OS evicted VRAM to RAM |
| `\GPU Process Memory(pid_*_luid_<blender card>_phys_0)\Dedicated Usage` | ~14 MiB seen during probes, still to be traced | > 32 MiB, meaning the model is touching Blender's card |
| Process working set | 195–283 MiB | > 384 MiB |

**On a trip:** kill the job, log the counter snapshot, run preflight again, and restart with the same config. Two trips within an hour → stay stopped and raise an alert in app9.

## Verification checklist (every start, shown in app9)

- [ ] Load log: `offloaded 29/29 layers to GPU`, `Vulkan0 model buffer`, `Vulkan0 KV buffer`, `Vulkan0 compute buffer`, and no CPU model buffer line
- [ ] Windows dedicated usage within 5% of the estimate; shared usage at baseline
- [ ] Coherence prompt passes
- [ ] Blender card shows no new allocation from the model process
