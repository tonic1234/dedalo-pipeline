# Roadmap — from 0.9 foundation to 1.0 in production

## v0.9 (this repo) — foundation ✅
Architecture, entity catalog (Greek mythology), handoff contracts, pipeline tests, anti-hallucination
layers, cost model, schemas. Birth plan:
`/root/.hermes/plans/2026-09-18_pipeline-v2-inspector-externo.md` (private).

## v0.9.1 — inspector calibration (first executable milestone)
- Install PR-Agent locally (The-PR-Agent/pr-agent, MIT) with `deepseek-flash`.
- Write the inspector prompt: verdict JSON `{score: 0-5, findings: [{file, line, issue, fix}], justification}`.
- Calibration test (T3): run it over **5 already-merged PRs** from real history.
- **Gate**: a clearly good PR must score ≥ 4. If not, fix the instrument (the prompt), not the code.
- This milestone touches **nothing** that runs today — it's read-only against history.

## v0.9.2 — pilot: one real case (T1 + T2)
- One small real feature travels the full flow to 5/5 with zero manual intervention except merge.
- Seeded fake-fix unit (T2) must be caught and requeued.
- **Gate**: T1 and T2 pass → the previous pipeline version can start being replaced.
- Décision required: batch scheduling (night/off-peak to halve cost) and watchdog wiring (Cerbero).

## v1.0 — orchestrator integration
- Add `review` state + requeue loop to the orchestrator; budget/delay ledger in Ariadna; per-role model
  registry enforced (requested vs effective model logged).
- Full test suite T1–T5 in CI with `verify.sh` + secrets scan.
- Everything documented inside the product (bilingual `<details>`), per standing convention.

## v1.1 — planner experiment with Claude (conditional)
**Current: Prometeo plans with `deepseek-flash` + `reasoning_effort=max`** (decision 18-sep-2026).
Planned experiment: A/B the same 3 units against `anthropic/claude-opus-5` via OpenRouter (key exists,
≈$0.30 total) — compare spec completeness, first-pass inspector score, rounds to 5/5, rework tokens.
The result decides the default; the pipeline never blocks on it. Re-evaluate **DeepSeek V4.1-Pro** as a
cheaper in-provider premium when it ships.

## Non-goals (declared)
- **No auto-merge** — the human is the only authority allowed to merge, always.
- **No free models for development or delegation** — retired for cause (0 commits in 30 min measured).
- **No paid service without OK** — Greptile/CodeRabbit SaaS reviewers are out; the inspector is local
  PR-Agent + deepseek-flash.
- **No breaking changes while something runs** — everything is additive; the old version stays alive
  until the gate passes.