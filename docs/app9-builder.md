# app9.nextaura.us: the building GUI for app19 (plan)

**Status:** plan only. Nothing is deployed. Today app9 serves a static "NextAura Multimodal Fine-tune" mirror, and it stays that way until Marco approves the cutover (see [Rollout](#rollout)).

app9 becomes the **control panel** for the app18-only model on the spare W7000. **Marco and agents use the same API.** The web page is just one client of it.

## Panels

| Panel | Shows | Actions |
| --- | --- | --- |
| **VRAM** | Spare card: Windows dedicated vs shared usage (live), budget vs the 3,200 MiB ceiling, the latest preflight result, watchdog trips. Blender's card: the model process's usage on it (must stay ~0). | Read-only |
| **Model** | State, llama.cpp build (must be the pinned b8393), model + context, active adapter, tokens/s, uptime | Start, stop, restart (each start runs preflight) |
| **Logs** | Live tail of the launcher, preflight, server, watchdog, tool proxy and nightly jobs. Filter by job and level. | Download a log bundle |
| **Jobs** | Recent app18 jobs driven by the model, each with its outcome and its Blender result | Label pass/fail (feeds the learning loop) |
| **Evals** | Nightly scorecards: success, match to the Blender reference, plan correctness, efficiency, and the forbidden-call gate | Trigger an eval of any adapter |
| **Compare** | Two adapters, or one adapter vs the Blender reference, per task: renders side by side, stats diff, worst first | Promote (only if the gate passed) |
| **Adapters** | Current, previous, the last 10 promoted, and candidates: hash, size on the card, dataset manifest, score, who promoted it | Roll back (instant scale swap) |
| **Audit** | Every state change: actor (Marco or agent session), action, arguments, result | — |

## Runner states (spare card)

```
stopped ──start──► preflight ──pass──► serving ──stop──► stopped
                      │                   │
                    fail               watchdog trip ──► stopped (+ alert after 2 trips/hour)
                      ▼
                   refused (reason shown: over_budget, wrong_build, card_ambiguous, spill_detected, incoherent)
```

## JSON API

Base: `https://app9.nextaura.us/api/v1`. JSON in and out. Every request carries `Authorization: Bearer <session token>`. Mutating requests also need an `Idempotency-Key`.

| Method | Path | Scope | Purpose |
| --- | --- | --- | --- |
| GET | `/status` | read | State, build, model, adapter, VRAM snapshot, last preflight |
| GET | `/vram` | read | Dedicated/shared usage on the spare card, budget, ceiling, watchdog trips, usage on the Blender card |
| GET | `/preflight/last` | read | Full preflight report (budget lines, load-log check, coherence result) |
| POST | `/model/start` | operate | `{ "adapter": "current" }`. Runs preflight first. Returns `409` with the reason if refused. |
| POST | `/model/stop` | operate | `{ "reason": "..." }` |
| GET | `/logs` | read | `?source=server&level=info&since=<cursor>&limit=500` |
| GET | `/logs/stream` | read | Server-sent events (live stream), same filters |
| GET | `/jobs` | read | Recent app18 jobs with outcomes |
| POST | `/jobs/{id}/label` | operate | `{ "label": "pass" \| "fail", "note": "..." }` |
| GET | `/adapters` | read | Current, previous, history, candidates |
| POST | `/evals` | operate | `{ "adapter": "<hash>" }` → `{ "eval_id" }` |
| GET | `/evals/{id}` | read | Status + scorecard |
| GET | `/evals/compare` | read | `?a=<hash>&b=<hash or "reference">` |
| POST | `/adapters/promote` | admin | `{ "adapter": "<hash>", "eval_id": "<id>" }`. Refused unless that eval passed the gate. |
| POST | `/adapters/rollback` | admin | `{ "to": "previous" \| "<hash>", "reason": "..." }` |
| GET | `/audit` | read | `?since=<cursor>` |
| GET | `/tools` | none | The tool list below, as JSON |

Errors: `{ "ok": false, "error": "<code>", "detail": "..." }`. The codes are `unauthorized`, `token_expired`, `forbidden_scope`, `over_budget`, `wrong_build`, `card_ambiguous`, `spill_detected`, `incoherent`, `busy`, `not_found`, `gate_failed` and `invalid_args`.

## Agent tools (MCP-style)

These are the same operations, served at `GET /api/v1/tools` and over an MCP endpoint behind the same auth.

| Tool | Args | Scope |
| --- | --- | --- |
| `get_status` | — | read |
| `get_vram` | — | read |
| `get_preflight` | — | read |
| `start_model` | `adapter?` | operate |
| `stop_model` | `reason` | operate |
| `tail_logs` | `source?`, `level?`, `since?`, `limit?` | read |
| `list_jobs` | `since?`, `outcome?` | read |
| `label_job` | `id`, `label`, `note?` | operate |
| `list_adapters` | — | read |
| `run_eval` | `adapter` | operate |
| `get_eval` | `eval_id` | read |
| `compare_evals` | `a`, `b` (hash or `reference`) | read |
| `promote_adapter` | `adapter`, `eval_id` | admin |
| `rollback_adapter` | `to`, `reason` | admin |
| `get_audit` | `since?` | read |

Agents get `read` + `operate` by default. `admin` (promote and roll back) is granted per session, only when Marco asks.

## Auth: per-session tokens

- **Grok Bot mints a short-lived token for each session.** No token is ever hardcoded in the repo, the site, the worker, an agent prompt, or the model.
- A token is a signed claim set: `sub` (Marco or an agent session), `scope` (`read` / `operate` / `admin`), `aud` (`app9` or `app18`), `exp` (1 hour by default, 8 hours max) and a unique `jti`.
- The signing key lives only in the gate's secret store. It is never in git, static assets or logs. Logs record the `jti`, never the token value.
- **The model itself** gets an `app18`-audience token through the local tool proxy. The proxy attaches it to allowlisted app18 calls. The model never sees it.
- Revoking a `jti` takes effect on the next request. The browser UI keeps its token in memory only, with no long-lived cookies.
- No extra Cloudflare services. The gate is the only public surface. The PC, the Blender node and the storage box have no new public exposure.

## Rollout

1. **Now:** plan only (this doc). app9 is unchanged.
2. Build the API against a mock runner, so every panel works with fake data.
3. Connect the real runner read-only first: `status`, `vram`, `logs`, `preflight`.
4. Turn on `operate` (start, stop, eval, label), then `admin` (promote, rollback).
5. **Cutover:** replace today's app9 page only after Marco approves. Keep the old mirror in git so it can be restored.

## Out of scope

- Any Cloudflare service beyond what the gate needs.
- Exposing the PC, the GPUs, the Blender node or the storage box to the internet.
- Marketing app9. It is an internal building tool.
