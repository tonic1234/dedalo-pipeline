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

## v1.1 — planificador premium (DECIDIDO 18-sep-2026)
**Prometeo corre `anthropic/claude-opus-5` vía OpenRouter** (la key ya existe — cero fricción; Anthropic
directo tiene el mismo precio de lista si después se agrega una key). Exactamente una llamada premium por
unidad: el planificador escribe **instrucciones ejecutables** (desglose, criterios de aceptación,
archivos exactos, comportamientos prohibidos) para los obreros deepseek-flash. Costo: ≈ $0.10/plan
($0.05 con `:batch`), +$0.50–1.00/día sobre todo-flash. Fallback: flash `reasoning_effort=max`
(degrade, nunca se detiene). Reevaluar **DeepSeek V4.1-Pro** como opción más barata dentro del proveedor
cuando salga.

## No-objetivos (declarados)
- **Sin auto-merge** — el humano es la única autoridad que mergea, siempre.
- **Sin modelos gratis para desarrollo ni delegación** — retirados por causa (0 commits en 30 min medido).
- **Sin servicio pago sin OK** — los revisores SaaS (Greptile/CodeRabbit) quedan afuera; el inspector es
  PR-Agent local + deepseek-flash.
- **Sin romper lo que corre** — todo es aditivo; la versión vieja sigue viva hasta que pase el gate.