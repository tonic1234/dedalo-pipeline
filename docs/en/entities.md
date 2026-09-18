# Entities — the Greek cast

Identifiers are Greek-mythology names chosen by **what the myth means for the role**. This keeps the
same identifiers valid in English and Spanish documentation and code, and gives every component a
mental anchor: you instantly know what Minos does because Minos is the judge.

## Registry

| Entity | Myth | Meaning for the role | Pipeline role | Model (default) | Artifact |
|---|---|---|---|---|---|
| **Dédalo** | Daedalus, master craftsman of Crete, builder of the Labyrinth | The one who designs and builds the whole factory | The pipeline itself | — | docs, schemas, code |
| **Prometeo** | Prometheus "the Forethinker", who planned before acting and gave humanity the arts | Thinks ahead, plans the work before anyone touches code | **Planner** | `deepseek-flash` + `thinking` + `reasoning_effort=max` | plan: unit breakdown, effort, budget estimate |
| **Pythia** | The Oracle of Delphi, whose prophecies declare what must happen | Declares the spec: the prophecy the factory must fulfill | **Spec writer** | `deepseek-flash` | `spec.md` |
| **Themis** | Goddess of divine law and order | The tests are the law: nothing ships until the law is satisfied | **Test writer (RED)** | `deepseek-flash` | failing tests + raw output |
| **Hefesto** | Hephaestus, blacksmith of the gods, forged in isolation in his forge | The workers who build, each at his own anvil (worktree) | **Builder** (×N parallel) | `deepseek-flash` | code in `agent/<unit>` branch, tests green |
| **Alétheia** | Goddess of revealed truth (a-letheia = "un-forgetting") | Evidence reveals what is true; nothing is taken on faith | **Prover** | scripts, no LLM | before/after pair, `assertions.md` |
| **Minos** | Judge of the dead, who weighs every soul | The external judge whose score decides if work is finished | **Inspector** | PR-Agent + `deepseek-flash` | score 0–5 + findings (file:line) |
| **Argos** | Argus Panoptes, the hundred-eyed guardian | Sees everything before anything moves | **Preflight gate** | no LLM | pass/fail on plan, tests, docs, git |
| **Cerbero** | Cerberus, the three-headed watchdog of Hades | Watches while everyone sleeps; barks if something stalls | **Watchdog** | no LLM | alerts (Telegram) |
| **Ariadna** | Ariadne, whose thread guided Theseus out of the Labyrinth | The thread through the Dédalo labyrinth: evidence, lessons, state | **Ledger / resumable state** | no LLM | per-unit log, changelog, state file |
| **Plutus** | God of wealth | Counts every coin; the budget is a cap, not a suggestion | **Budget controller** | no LLM | spend ledger, warnings, cuts |
| **Zeus** | King of Olympus | The human: the only authority allowed to merge | **Human operator** | human | merge click |

## Naming notes

- Names are **neutral identifiers** — engine code, logs and docs use `prometeo`, `minos`, `hefesto`
  regardless of the language of the surrounding text.
- Spanish and English docs both use the Greek names (not translations).
- Legacy internal names (ARGOS gate, HILO thread) keep their meaning: Argos → **Argos**, HILO → **Ariadna**.

## Per-entity contract detail

### Prometeo (Planner) — the only premium-reasoning role
- Inputs: the goal, the current codebase map, budget, active constraints.
- Outputs: ordered unit breakdown, effort estimate per unit, risk notes, proposed model per role.
- The plan is a **proposal**: Pythia turns it into a spec, and Themis can falsify it.

### Pythia (Spec)
- Specs are complete prose, point by point, connected to what already exists (current fields, modals,
  UI). Explicit statements of what existing behavior **must not** break.
- "Saca X" means delete, not relocate. Instructions are law.

### Themis (Tests)
- Tests come **before** the code. The RED is saved as a file (raw output with the failure).
- The valuable test is the **integration test against the real destination**, not a unit test of a helper.
- If a test contradicts the spec, the **spec wins**: the test is fixed with explicit OK and the reason
  is logged.

### Hefesto (Builders)
- Multiple builders run in parallel, each in an isolated worktree, no shared branch.
- Every builder commits in slices (progress survives timeouts).
- They write code to the service-layer standard (see `code-structure`, vendored from Ras Mic's kit).

### Alétheia (Prove)
- Evidence is never the agent's word: raw logs, file sizes, listings, counts at the destination.
- The "before" is captured **while reproducing the failure**, prior to fixing it — when it is cheapest.
- `assertions.md` marks every check `passed / failed / untested + reason`; silent skips are forbidden.

### Minos (Inspector)
- Runs on code it never discussed with the builder (fresh context: diff + spec only).
- Emits score + findings; findings must justify the score (a low score with an empty list is rejected
  as a verdict).
- Loop: score < 5 → back to Hefesto with the findings as the next spec. Cap: N iterations (default 4).
- At 5/5 with zero unresolved, the unit is presented to Zeus.

### Argos, Cerbero, Plutus, Ariadna
- Deterministic tooling: gates, timers, budgets and ledgers are plain scripts and files, no LLM — they
  are the machinery that the LLM stages cannot fake.