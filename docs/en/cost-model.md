# Cost model & model per role

Pricing facts are taken from the official DeepSeek API docs (checked 2026-09-18). All prices are
per 1M tokens.

## Official DeepSeek-V4.1-Flash pricing (effective 2026-09-10)

| | Input (cache hit) | Input (cache miss) | Output |
|---|---|---|---|
| **Off-peak** | $0.003 | $0.15 | $0.60 |
| **Peak** | $0.006 | $0.30 | $1.20 |

- **Peak hours**: 01:00–04:00 and 06:00–10:00 UTC on weekdays. Everything else is off-peak,
  including weekends and public holidays.
- Off-peak is **50% of peak**. Scheduling the batch (night cron) makes the day ~half price.
- **Cache note**: agent pipelines repeat the same context (AGENTS/architecture, spec, repo map).
  A cache-hit input token costs $0.003–0.006 — **50× cheaper than a miss**. Dédalo is designed to reuse
  context blocks so the cache does the heavy lifting.

## Big picture: what the provider offers today

| Model name | Status (2026-09-18) | Verdict |
|---|---|---|
| `deepseek-flash` | **DeepSeek-V4.1-Flash** — the current model. 552B MoE, 8B active input / 16B output, native multimodal. Benchmarks ahead of V4-Pro "per multiple independent tests" | **The worker model.** Default everywhere in Dédalo |
| `deepseek-v4-pro` | **Retired.** Since 2026-09-14 all `deepseek-v4-pro` requests route to V4.1-Flash at Flash rates; DeepSeek states it is phasing out V4-Pro | Do **not** configure it: it is not a separate model anymore |
| `deepseek-v4-flash` (legacy alias) | Still accepted, routed to V4.1-Flash, billed at Flash price | Fine as alias; prefer the canonical `deepseek-flash` |
| `deepseek-v4-flash-vision-exp` (legacy alias) | Routed to V4.1-Flash | Same note |
| V4.1-Pro (announced) | Not yet on the API ("until V4.1-Pro launches") | The natural **future** premium option for Prometeo — revisit when it ships |
| Claude / GPT (other providers) | Requires its own key and explicit human OK per standing rule | Only as a reference row — never switched to on our own |

## Model per role (default assignments)

| Entity | Role | Model | Reasoning mode |
|---|---|---|---|
| Prometeo | Planner | `anthropic/claude-opus-5` (OpenRouter) | premium; fallback: flash `max` |
| Pythia | Spec | `deepseek-flash` | `thinking` enabled, `high` |
| Themis | Tests | `deepseek-flash` | `high` |
| Hefesto | Builders (×N) | `deepseek-flash` | `high` |
| Minos | Inspector (PR-Agent) | `deepseek-flash` | `high` |
| Alétheia / Argos / Cerbero / Plutus / Ariadna | Tooling | — (no LLM) | — |

**Total LLM roles: exactly 5** (plan, spec, tests, build×N, inspect). Everything else is deterministic
tooling that cannot be fooled. Only **Prometeo** runs a premium model, one call per unit.

## The premium planner — decision (2026-09-18, operator-approved)

The idea: a **much better model only for planning**, `deepseek-flash` on the workers, and clear
instructions from the planner so the workers don't reinterpret.

**Decision**: Prometeo runs **`anthropic/claude-opus-5`** (Anthropic) **via OpenRouter**.

| Aspect | Value (checked 2026-09-18) |
|---|---|
| Model | `anthropic/claude-opus-5` (1M context) |
| Route | **OpenRouter** (key `OPENROUTER_API_KEY` already exists) — direct Anthropic is the same list price and needs a new key/account; both are valid, OpenRouter is the zero-friction default |
| Price (OpenRouter) | $5.00 / 1M input · $25.00 / 1M output · `:batch` variant at half ($2.50 / $12.50) |
| Cost per plan (est. 10k in / 2k out) | ≈ $0.05 + $0.05 = **≈ $0.10** (≈ $0.05 with `:batch`) |
| Daily impact (5–10 units/day) | **+$0.50 – $1.00** over an all-flash pipeline |

Why this is safe:
- Prometeo is **one call per unit** — the premium cost does not multiply across iterations (the review
  loop only re-runs Hefesto + Minos, both flash).
- The plan is **executable instructions** for the flash workers (domain breakdown, acceptance criteria,
  exact files, forbidden behaviors), so the premium pays once and the cheap workers execute.
- Fallback: `deepseek-flash` with `reasoning_effort=max` if the OpenRouter route is down or the budget
  is exceeded; the pipeline degrades, never stops.
- If/when **DeepSeek V4.1-Pro** ships, re-evaluate it for the same role as a cheaper in-provider option.

## Estimated cost per unit (off-peak, cache-assisted)

| Role | Tokens in (est.) | Tokens out (est.) | Cost (est.) |
|---|---|---|---|
| Prometeo (plan, Opus 5) | 10k | 2k | ≈ $0.10 |
| Pythia (spec) | 8k | 1.5k | ≈ $0.002 |
| Themis (tests) | 10k | 1.5k | ≈ $0.002 |
| Hefesto (build, 1 iteration) | 40k | 8k | ≈ $0.011 |
| Minos (inspect, 1 round) | 50k | 4k | ≈ $0.010 |
| **Unit total, 1 build + 2 review rounds** | — | — | **≈ $0.14–0.16** (~$0.09–0.11 with Opus `:batch`) |

A full working day (10–20 units, 10–40k tokens per call, off-peak, cache-assisted):
**≈ $1.00–2.50** — the all-flash day was ~$0.50–1.50; the premium planner adds the Opus cost above.

## Budget mechanics (Plutus)

- Caps are **per run**: hard limit declared before the run; 80% consumed → warning to the operator;
  100% → hard cut, state saved, Telegram alert.
- Every call logs: requested model vs effective model, tokens, cost, handoffs, verdicts — so a role
  that silently uses a paid model (a real incident in the past) is caught by the ledger.
- Free models are **not** used for development or delegation (measured: five free subagents produced
  0 commits in 30 minutes; the free pool was retired because it stalls, can't read files and ships
  credential bugs).