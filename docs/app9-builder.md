# app9.nextaura.us — Builder GUI for app19 (plan)

**Status:** plan only. Nothing here is deployed. The current app9 stays exactly as it is until Marco approves the cutover (see [Rollout](#rollout)).

app9 becomes the **control panel** for the app19 local character model: the person or agent at the panel sees the VRAM budget, starts and stops the model on the spare card, reads logs, triggers and compares nightly adapter evals against the Blender reference, and rolls back. **Marco and agents use the same API.** The web UI is just one client of it.

## Panels

| Panel | Shows | Actions |
| --- | --- | --- |
| **VRAM budget** | Spare card total, estimated vs measured usage, the green/amber/red band, and the current job. The main card shows only as "not managed". | None. This panel is read-only. |
| **Model** | State (`stopped`, `starting`, `serving`, `training`, `stopping`, `error`), current adapter hash, uptime, and tokens/s | Start, stop, restart |
| **Logs** | Live tail of runner, trainer, and eval logs, filterable by job and level | Download a log bundle for a job |
| **Evals** | Nightly scorecards over time. Per-metric trend lines for schema-valid, field accuracy, silhouette IoU, palette ΔE, and SSIM. | Trigger an eval on any adapter |
| **Compare** | Two adapters side by side, or one adapter vs the Blender reference. Per-brief render grid, worst delta first. | Promote (if the gate passes) |
| **Adapters** | The last 10 promoted adapters plus candidates: hash, created, data manifest, score, and who promoted it | Roll back to any listed adapter |
| **Audit** | Every state change: actor (Marco or agent session), action, args, and result | — |

## State machine (spare card runner)

```
stopped ──start──► starting ──ok──► serving ──stop──► stopping ──► stopped
   ▲                   │                │
   │                 fail             train (scheduled or manual)
   │                   ▼                ▼
   └──────reset────── error ◄──fail── training ──done──► serving (current adapter)
```

- Only one job runs on the spare card at a time. `train` stops serving first.
- Every start is checked against the VRAM budget. If the estimate is over, the start is rejected with `409 over_budget`.
- When the measured peak goes above the red line, the runner checkpoints, moves to `stopping`, and writes an audit event.

## JSON API

Base: `https://app9.nextaura.us/api/v1`. JSON in, JSON out. Every request carries `Authorization: Bearer <session token>`. Every mutating request also carries an `Idempotency-Key` header.

| Method | Path | Scope | Purpose |
| --- | --- | --- | --- |
| GET | `/status` | `read` | Runner state, current adapter, VRAM snapshot, current job |
| GET | `/vram` | `read` | Card total, estimated, measured, peak, band, thresholds |
| POST | `/model/start` | `operate` | `{ "adapter": "current" \| "<hash>", "mode": "serve" }` |
| POST | `/model/stop` | `operate` | `{ "reason": "..." }` |
| GET | `/logs` | `read` | `?job=<id>&level=info&since=<cursor>&limit=500` returns lines + next cursor |
| GET | `/logs/stream` | `read` | Server-sent events, same filters |
| GET | `/adapters` | `read` | Promoted and candidate adapters with scores |
| POST | `/evals` | `operate` | `{ "adapter": "<hash>", "set": "frozen" }` returns `{ "eval_id" }` |
| GET | `/evals/{id}` | `read` | Status + scorecard once done |
| GET | `/evals/compare` | `read` | `?a=<hash>&b=<hash or "reference">` returns per-metric deltas + per-brief render pairs |
| POST | `/adapters/promote` | `admin` | `{ "adapter": "<hash>", "eval_id": "<id>" }`. Refused unless that eval passed the gate |
| POST | `/adapters/rollback` | `admin` | `{ "to": "<hash>", "reason": "..." }` |
| GET | `/audit` | `read` | `?since=<cursor>` |
| GET | `/tools` | none | The MCP-style tool list below, as JSON |

Errors use one shape: `{ "ok": false, "error": "<code>", "detail": "..." }`. Codes: `unauthorized`, `forbidden_scope`, `token_expired`, `over_budget`, `busy`, `not_found`, `gate_failed`, `invalid_args`.

### Example

```http
POST /api/v1/evals
Authorization: Bearer <session token>
Idempotency-Key: 7f1c...
Content-Type: application/json

{ "adapter": "a1b2c3d4", "set": "frozen" }
```

```json
{ "ok": true, "eval_id": "ev_20261009_0200", "state": "queued" }
```

## Agent tool list (MCP-style)

The same operations as tools, so any agent can drive app9 without the browser. Served at `GET /api/v1/tools` and over an MCP endpoint behind the same auth.

| Tool | Args | Returns | Scope |
| --- | --- | --- | --- |
| `get_status` | — | runner state, adapter, VRAM snapshot | read |
| `get_vram_budget` | — | total, estimated, measured, peak, band | read |
| `start_model` | `adapter?`, `mode?` | new state | operate |
| `stop_model` | `reason` | new state | operate |
| `tail_logs` | `job?`, `level?`, `since?`, `limit?` | lines, cursor | read |
| `list_adapters` | — | adapters with scores | read |
| `run_eval` | `adapter`, `set?` | eval_id | operate |
| `get_eval` | `eval_id` | status, scorecard | read |
| `compare_evals` | `a`, `b` (hash or `reference`) | deltas, render pairs | read |
| `promote_adapter` | `adapter`, `eval_id` | new current | admin |
| `rollback_adapter` | `to`, `reason` | new current | admin |
| `get_audit` | `since?` | events | read |

Agents get `read` + `operate` by default. `admin` (promote and rollback) is granted per session only when Marco asks for it.

## Auth

- **Grok Bot mints a short-lived token for each session.** Nothing is hardcoded in the repo, the site, the worker, or the agents' prompts.
- The token is a signed claim set: `sub` (Marco or agent session id), `scope` (`read` / `operate` / `admin`), `exp` (default 1 hour, max 8 hours), `jti` (unique id), and `aud` = `app9`.
- The signing key lives only in the gate's secret store. It's never in git, never in the static assets, and never in logs.
- The gate verifies signature, expiry, audience, and scope on every call. Revoking a `jti` takes effect on the next request.
- The browser UI gets its token from the same minting flow. It keeps the token in memory only, with no long-lived cookies or localStorage.
- Every call is written to the audit log with `sub`, `jti`, the action, and the result. Token values themselves are never logged.
- The gate is the only public surface. The spare-card runner and the Blender node have no public address.

## Rollout

1. **Now:** plan only (this doc). The current app9 is unchanged.
2. Build the gate + runner API against a mock runner. All panels work with fake data.
3. Hook up the real runner on the spare card, read-only first (`status`, `vram`, `logs`).
4. Turn on `operate` (start, stop, eval), then `admin` (promote, rollback).
5. **Cutover:** replace the current app9 page only after Marco approves. Keep the old static mirror's files in the repo history so it can be restored.

## Out of scope

- Any Cloudflare service beyond what the gate needs. Each one gets its own decision when we get to it.
- Exposing the main GPU card, the GPU box, or the Blender node to the internet.
- Marketing app9. It's an internal builder tool.
