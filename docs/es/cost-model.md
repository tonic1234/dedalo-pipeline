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
| Prometeo | Planificador | `deepseek-flash` | `thinking` + `reasoning_effort=max` |
| Pythia | Spec | `deepseek-flash` | `thinking` activado, `high` |
| Themis | Tests | `deepseek-flash` | `high` |
| Hefesto | Obreros (×N) | `deepseek-flash` | `high` |
| Minos | Inspector (PR-Agent) | `deepseek-flash` | `high` |
| Alétheia / Argos / Cerbero / Plutus / Ariadna | Herramientas | — (sin LLM) | — |

**Roles LLM totales: exactamente 5** (planificar, spec, tests, construir×N, inspeccionar). Todo lo demás
es herramienta determinística que no se puede engañar.

## El modelo del planificador — decisión vigente y el experimento planificado

Decisión (18-sep-2026): **Prometeo planifica con `deepseek-flash` + `thinking` + `reasoning_effort=max`**
por ahora. La opción premium se evaluó y quedó como *experimento medido*:

| Aspecto | Valor (verificado 18-sep-2026) |
|---|---|
| Candidato | `anthropic/claude-opus-5` (contexto 1M), vía OpenRouter (la key `OPENROUTER_API_KEY` ya existe; Anthropic directo = mismo precio de lista y pide key/cuenta nueva) |
| Precio (OpenRouter) | $5.00 / 1M input · $25.00 / 1M output · variante `:batch` a mitad ($2.50 / $12.50) |
| Costo por plan si se adopta (est. 10k in / 2k out) | ≈ $0.10 (≈ $0.05 con `:batch`) — impacto ≈ +$0.50–1.00/día |

Por qué DeepSeek primero: ningún modelo premium es *necesario* para que el pipeline funcione — flash con
razonamiento máximo es la base medida; el premium solo tiene sentido si reduce el retrabajo aguas abajo.

**Protocolo A/B (cuando se corra)**: las mismas 3 unidades se planifican con ambos modelos (flash-max vs
opus-5). Comparar por unidad: completitud de la spec, score del inspector al primer pase, vueltas hasta
5/5, tokens de retrabajo. Costo del experimento: 3 unidades × ≈$0.10 = ≈$0.30. El resultado decide el
default; el pipeline nunca bloquea por eso. Reevaluar **DeepSeek V4.1-Pro** como premium dentro del
proveedor cuando salga.

## Costo estimado por unidad (off-peak, con caché)

| Rol | Tokens in (est.) | Tokens out (est.) | Costo (est.) |
|---|---|---|---|
| Prometeo (plan, flash max) | 10k | 2k | ≈ $0.003 |
| Pythia (spec) | 8k | 1.5k | ≈ $0.002 |
| Themis (tests) | 10k | 1.5k | ≈ $0.002 |
| Hefesto (build, 1 iteración) | 40k | 8k | ≈ $0.011 |
| Minos (inspección, 1 vuelta) | 50k | 4k | ≈ $0.010 |
| **Total por unidad, 1 build + 2 vueltas de revisión** | — | — | **≈ $0.04–0.06** |

*Si el experimento A/B de Prometeo adopta `claude-opus-5`, sumar ≈ $0.10 por unidad (≈ $0.05 con `:batch`).*

Un día completo de trabajo (10–20 unidades, 10–40k tokens por llamada, off-peak, con caché):
**≈ $0.50–1.50**.

## Mecánica de presupuesto (Plutus)

- Los topes son **por corrida**: límite duro declarado antes de la corrida; 80% consumido → aviso al
  operador; 100% → corte duro, estado guardado, alerta por Telegram.
- Cada llamada registra: modelo pedido vs modelo efectivo, tokens, costo, handoffs, veredictos — así un
  rol que toque un modelo pago en silencio (incidente real del pasado) queda atrapado en la bitácora.
- Los modelos gratis **no** se usan para desarrollo ni delegación (medido: cinco subagentes gratis
  produjeron 0 commits en 30 minutos; el pool free se retiró porque se cuelga, no lee archivos y trae
  bugs de credenciales).