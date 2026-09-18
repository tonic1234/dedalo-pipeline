# Anti-hallucination layers — how Dédalo survives LLM lies

LLM agents can be pushed to lie, to believe false things, or to silently skip work
("agents can't pinky promise"). Dédalo doesn't try to make agents honest; it makes **lying
structurally expensive and visible**. Seven layers:

## 1. Nobody is their own source
Every "done / green / works" is a raw artifact (log, listing, counts, score) **validated by the
receiving stage**, which re-runs the check itself. An agent never reports its own result.

## 2. The inspector is a stranger to the builder
Minos sees only the **diff and the spec** — fresh context, no shared conversation, different memory.
A lie cannot be coordinated with the judge, and a mistake made while building does not contaminate the
review. The judge reads the code, not the builder's claims.

## 3. Omission is visible, never silent
`assertions.md` marks every check `passed / failed / untested + reason`.
"What I didn't run but said I did" stops being silence: it is an annotated gap that blocks the handoff.

## 4. The score must justify itself
A low score with an empty findings list is **rejected as a verdict**. The judge has to account for the
score; the ledger records both.

## 5. "It works but it's sloppy" is a category the judge checks
Models will do the job in the sloppiest acceptable way if they can. Minos reviews against a written
code-structure standard (service layer, no god functions, no duplication, no dead code) — the things a
model won't judge well on itself without a written guide.

## 6. Sustained lying is expensive
Hard iteration cap, stage timeouts, budget cap with a hard cut at 100%, and a watchdog alerting on
stall. A lie chain burns iterations and budget, trips the alert and lands in the ledger — the system
turns lies into visible noise instead of absorbing them.

## 7. The merge is human
The only stage without automation. Zeus doesn't read code line by line: he reads the before/after
evidence pair and the score, then clicks merge — or sends the unit back.

## What these layers do NOT do (honest limits)

- They don't make the models perfect reviewers — the calibration test (T3) sets the acceptable floor.
- They don't replace real tests: `verify.sh` and integration tests against the real destination are
  still the final truth for "does it work".
- The review loop catches structural and consistency lies; the evidence pair catches "the change isn't
  what I claimed". The human remains the last line for judgment calls.