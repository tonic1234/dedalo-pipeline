# Modelo de costos y modelo por rol

Los datos de precios vienen de la documentación oficial de la API de DeepSeek (verificado 18-sep-2026).
Todos los precios son por 1M de tokens.

## Precio oficial DeepSeek-V4.1-Flash (vigente desde 2026-09-10)

| | Input (cache hit) | Input (cache miss) | Output |
|---|---|---|---|
| **Off-peak** | $0.003 | $0.15 | $0.60 |
| **Peak** | $0.006 | $0.30 | $1.20 |

- **Horas pico**: 01:00–04:00 y 06:00–10:00 UTC en días hábiles. Todo lo demás es off-peak, incluidos
  findes y feriados.
- Off-peak es **50% del peak**. Programar el lote (cron nocturno) deja el día a mitad de precio.
- **Nota de caché**: los pipelines de agentes repiten el mismo contexto (AGENTS/arquitectura, spec,
  mapa del repo). Un token de input con cache-hit cuesta $0.003–0.006 — **50× menos que un miss**.
  Dédalo está diseñado para reusar bloques de contexto para que la caché haga el trabajo pesado.

## Panorama: qué ofrece el proveedor hoy

| Modelo | Estado (18-sep-2026) | Veredicto |
|---|---|---|
| `deepseek-flash` | **DeepSeek-V4.1-Flash** — el modelo actual. 552B MoE, 8B activos de input / 16B de output, multimodal nativo. Benchmarks por delante de V4-Pro "según pruebas independientes" | **El modelo de los obreros.** Default en todo Dédalo |
| `deepseek-v4-pro` | **Retirado.** Desde el 14-sep-2026 todas las llamadas a `deepseek-v4-pro` se enrutan a V4.1-Flash a tarifa Flash; DeepSeek declaró que retira V4-Pro | **No configurarlo**: ya no es un modelo separado |
| `deepseek-v4-flash` (alias legacy) | Todavía aceptado, enrutado a V4.1-Flash, facturado a tarifa Flash | OK como alias; preferir el canónico `deepseek-flash` |
| `deepseek-v4-flash-vision-exp` (alias legacy) | Enrutado a V4.1-Flash | Igual |
| V4.1-Pro (anunciado) | Todavía no está en la API ("until V4.1-Pro launches") | La opción premium **futura** natural para Prometeo — reevaluar cuando salga |
| Claude / GPT (otros proveedores) | Requiere key propia y OK humano explícito (regla permanente) | Solo como fila de referencia — nunca se cambia de proveedor por decisión propia |

## Modelo por rol (asignaciones por defecto)

| Entidad | Rol | Modelo | Modo de razonamiento |
|---|---|---|---|
| Prometeo | Planificador | `deepseek-flash` | `thinking` activado + `reasoning_effort=max` |
| Pythia | Spec | `deepseek-flash` | `thinking` activado, `high` |
| Themis | Tests | `deepseek-flash` | `high` |
| Hefesto | Obreros (×N) | `deepseek-flash` | `high` |
| Minos | Inspector (PR-Agent) | `deepseek-flash` | `high` |
| Alétheia / Argos / Cerbero / Plutus / Ariadna | Herramientas | — (sin LLM) | — |

**Roles LLM totales: exactamente 5** (planificar, spec, tests, construir×N, inspeccionar). Todo lo demás
es herramienta determinística que no se puede engañar.

## La pregunta del "planificador muy bueno" — veredicto

La idea: poner un modelo **mucho mejor solo para planificar**, y `deepseek-flash` en los obreros.

- Hoy la API de DeepSeek **no tiene un modelo premium a la venta**: V4-Pro fue retirado y enruta a
  V4.1-Flash a tarifa Flash; V4.1-Pro todavía no salió.
- Por lo tanto, lo mejor disponible como "planificación premium" dentro del proveedor actual es:
  **flash con thinking + reasoning_effort=max solo en Prometeo** — al mismo precio que flash normal
  (~$0.003 por plan), y es el único rol que corre razonamiento máximo.
- Cuando salga **V4.1-Pro**, es el modelo natural de Prometeo (una llamada por unidad, así que el
  impacto de costo es marginal). Reevaluar entonces, con OK explícito antes de escalar a algo pago.
- El premium de otro proveedor (Claude/GPT) es solo opción de referencia: necesitaría key propia, rompe
  el default "un proveedor" y está vedado sin aprobación humana explícita.

## Costo estimado por unidad (off-peak, con caché)

| Rol | Tokens in (est.) | Tokens out (est.) | Costo (est.) |
|---|---|---|---|
| Prometeo (plan) | 10k | 2k | ≈ $0.002 |
| Pythia (spec) | 8k | 1.5k | ≈ $0.002 |
| Themis (tests) | 10k | 1.5k | ≈ $0.002 |
| Hefesto (build, 1 iteración) | 40k | 8k | ≈ $0.011 |
| Minos (inspección, 1 vuelta) | 50k | 4k | ≈ $0.010 |
| **Total por unidad, 1 build + 2 vueltas de revisión** | — | — | **≈ $0.04–0.06** |

Un día completo de trabajo (10–20 unidades, 10–40k tokens por llamada, off-peak, con caché):
**≈ $0.50–1.50** — consistente con la base histórica medida de ~US$1 por día entero de trabajo.
Los lotes semanales programados de noche/off-peak corren a la mitad.

## Mecánica de presupuesto (Plutus)

- Los topes son **por corrida**: límite duro declarado antes de la corrida; 80% consumido → aviso al
  operador; 100% → corte duro, estado guardado, alerta por Telegram.
- Cada llamada registra: modelo pedido vs modelo efectivo, tokens, costo, handoffs, veredictos — así un
  rol que toque un modelo pago en silencio (incidente real del pasado) queda atrapado en la bitácora.
- Los modelos gratis **no** se usan para desarrollo ni delegación (medido: cinco subagentes gratis
  produjeron 0 commits en 30 minutos; el pool free se retiró porque se cuelga, no lee archivos y trae
  bugs de credenciales).