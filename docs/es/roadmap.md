# Roadmap — de la fundación 0.9 al 1.0 en producción

## v0.9 (este repo) — fundación ✅
Arquitectura, catálogo de entidades (mitología griega), contratos de handoff, tests del pipeline, capas
anti-alucinación, modelo de costos, esquemas. Plan de nacimiento:
`/root/.hermes/plans/2026-09-18_pipeline-v2-inspector-externo.md` (privado).

## v0.9.1 — calibración del inspector (primer hito ejecutable)
- Instalar PR-Agent local (The-PR-Agent/pr-agent, MIT) con `deepseek-flash`.
- Escribir el prompt del inspector: veredicto JSON `{score: 0-5, findings: [{file, line, issue, fix}], justification}`.
- Test de calibración (T3): correrlo sobre **5 PRs ya mergeados** del historial real.
- **Gate**: un PR claramente bueno debe puntuar ≥ 4. Si no, se arregla el instrumento (el prompt), no el código.
- Este hito **no toca nada** de lo que corre hoy — es solo lectura sobre el historial.

## v0.9.2 — piloto: un caso real (T1 + T2)
- Una feature chica real recorre el flujo completo hasta 5/5 sin intervención manual salvo el merge.
- La unidad con fix falso pre-sembrado (T2) debe ser detectada y re-encolada.
- **Gate**: T1 y T2 en verde → se empieza a poder reemplazar la versión anterior del pipeline.
- Decisión requerida: programación del lote (noche/off-peak para mitad de precio) y cableado del
  watchdog (Cerbero).

## v1.0 — integración al orquestador
- Agregar el estado `review` + loop de re-encolado al orquestador; bitácora de presupuesto/demora en
  Ariadna; registro de modelos por rol enforced (modelo pedido vs efectivo logueado).
- Suite completa T1–T5 en CI con `verify.sh` + escaneo de secretos.
- Todo documentado dentro del producto (`<details>` bilingüe), según la convención permanente.

## v1.1 — experimento del planificador con Claude (condicional)
**Actual: Prometeo planifica con `deepseek-flash` + `reasoning_effort=max`** (decisión 18-sep-2026).
Experimento planificado: A/B de las mismas 3 unidades contra `anthropic/claude-opus-5` vía OpenRouter
(key existente, ≈$0.30 en total) — comparar completitud de la spec, score del inspector al primer pase,
vueltas hasta 5/5, tokens de retrabajo. El resultado decide el default; el pipeline nunca bloquea por
eso. Reevaluar **DeepSeek V4.1-Pro** como premium más barato dentro del proveedor cuando salga.

## No-objetivos (declarados)
- **Sin auto-merge** — el humano es la única autoridad que mergea, siempre.
- **Sin modelos gratis para desarrollo ni delegación** — retirados por causa (0 commits en 30 min medido).
- **Sin servicio pago sin OK** — los revisores SaaS (Greptile/CodeRabbit) quedan afuera; el inspector es
  PR-Agent local + deepseek-flash.
- **Sin romper lo que corre** — todo es aditivo; la versión vieja sigue viva hasta que pase el gate.