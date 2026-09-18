# Schemas — Mermaid diagrams

GitHub renders Mermaid inside Markdown code blocks (see `README.md` and the docs). The standalone
`.mmd` files below are for editors and CLI rendering (`mmdc -i file.mmd -o file.svg`).

| File | Shows |
|---|---|
| `architecture.mmd` | The full flow: Zeus → Prometeo → Pythia → Themis → Hefesto → Alétheia → Argos → Minos, the review loop, and the tooling entities (Ariadna, Plutus, Cerbero) |
| `state-machine.mmd` | The unit state machine: spec → red → build → prove → review → merge_pending, with the requeue loop |
| `handoffs.mmd` | Every handoff as an edge with its artifact |

Render locally:

```bash
npx -y @mermaid-js/mermaid-cli -i schemas/architecture.mmd -o schemas/architecture.svg
```