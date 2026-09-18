# Entidades — el elenco griego

Los identificadores son nombres de la mitología griega elegidos por **lo que el mito significa para el
rol**. Así los mismos identificadores valen en la documentación y el código en español y en inglés, y
cada componente gana un anclaje mental: sabés al instante qué hace Minos porque Minos es el juez.

## Registro

| Entidad | Mito | Qué significa para el rol | Rol en el pipeline | Modelo (por defecto) | Artefacto |
|---|---|---|---|---|---|
| **Dédalo** | Daedalus, artesano maestro de Creta, constructor del Laberinto | El que diseña y construye toda la fábrica | El pipeline mismo | — | docs, esquemas, código |
| **Prometeo** | Prometheus "el que piensa antes", que planeó antes de actuar y dio las artes a la humanidad | Piensa adelante: planifica el trabajo antes de que nadie toque código, y deja instrucciones claras para todas las demás etapas | **Planificador** | `claude-opus-5` (Anthropic vía OpenRouter) | plan: desglose de unidades, esfuerzo, presupuesto |
| **Pythia** | El Oráculo de Delfos, cuyas profecías declaran lo que debe pasar | Declara la spec: la profecía que la fábrica debe cumplir | **Escritora de la spec** | `deepseek-flash` | `spec.md` |
| **Themis** | Diosa de la ley divina y el orden | Los tests son la ley: nada sale hasta satisfacerla | **Escritora de tests (RED)** | `deepseek-flash` | tests que fallan + salida cruda |
| **Hefesto** | Hephaestus, herrero de los dioses, que forjaba aislado en su fragua | Los obreros que construyen, cada uno en su yunque (worktree) | **Constructor** (×N en paralelo) | `deepseek-flash` | código en rama `agent/<unidad>`, tests en verde |
| **Alétheia** | Diosa de la verdad revelada (a-letheia = "des-olvido") | La evidencia revela lo verdadero; nada se acepta por fe | **Probador** | scripts, sin LLM | par antes/después, `assertions.md` |
| **Minos** | Juez de los muertos, que pesa cada alma | El juez externo cuyo score decide si el trabajo terminó | **Inspector** | PR-Agent + `deepseek-flash` | score 0–5 + observaciones (archivo:línea) |
| **Argos** | Argus Panoptes, el guardián de los cien ojos | Todo lo ve antes de que algo se mueva | **Gate de preflight** | sin LLM | pasa/no pasa sobre plan, tests, doc, git |
| **Cerbero** | Cerberus, el perro guardián de tres cabezas del Hades | Vigila mientras todos duermen; ladra si algo se traba | **Watchdog** | sin LLM | avisos (Telegram) |
| **Ariadna** | Ariadne, cuyo hilo guió a Teseo fuera del Laberinto | El hilo por el laberinto de Dédalo: evidencia, lecciones, estado | **Bitácora / estado reanudable** | sin LLM | log por unidad, changelog, archivo de estado |
| **Plutus** | Dios de la riqueza | Cuenta cada moneda; el presupuesto es tope, no sugerencia | **Control de presupuesto** | sin LLM | registro de gasto, avisos, cortes |
| **Zeus** | Rey del Olimpo | El humano: la única autoridad autorizada a mergear | **Operador humano** | humano | click de merge |

## Notas de nomenclatura

- Los nombres son **identificadores neutros**: el código, los logs y la doc usan `prometeo`, `minos`,
  `hefesto` sin importar el idioma del texto que los rodea.
- La doc en español y en inglés usa los nombres griegos (no traducciones).
- Los nombres internos previos conservan su sentido: ARGOS → **Argos**, HILO → **Ariadna**.

## Contrato por entidad

### Prometeo (Planificador) — el único rol premium
- Entradas: el objetivo, el mapa del código actual, el presupuesto, las restricciones activas.
- Salidas: desglose ordenado en unidades, esfuerzo estimado por unidad, riesgos, modelo propuesto por
  rol — y sobre todo, **instrucciones lo bastante claras para que los obreros deepseek-flash las
  ejecuten sin reinterpretar**: criterios de aceptación explícitos, archivos exactos, comportamientos
  prohibidos.
- Modelo: `claude-opus-5` (Anthropic, vía OpenRouter — la key ya existe). Fallback:
  `deepseek-flash` con `reasoning_effort=max` si la ruta premium no está disponible o excede presupuesto.
- El plan es una **propuesta**: Pythia lo convierte en spec y Themis puede falsificarlo.

### Pythia (Spec)
- Specs completas en prosa, punto por punto, conectadas con lo que ya existe (campos, modales, UI actual).
- Declaración explícita de qué comportamiento existente **no** se puede romper.
- "Saca X" significa borrar, no reubicar. Las instrucciones son ley.

### Themis (Tests)
- Los tests van **antes** del código. El RED se guarda como archivo (salida cruda con el fallo).
- El test que vale es el de **integración contra el destino real**, no el unit test de un helper.
- Si un test contradice la spec, **manda la spec**: se corrige el test con OK explícito y queda anotado
  el porqué.

### Hefesto (Obreros)
- Varios obreros en paralelo, cada uno en su worktree aislado, sin rama compartida.
- Cada obrero commitea por tramos (el progreso sobrevive a los timeouts).
- Escriben código contra el estándar de capas de servicio (`code-structure`, del kit de Ras Mic).

### Alétheia (Prueba)
- La evidencia nunca es la palabra del agente: logs crudos, tamaños de archivo, listados, conteos en el
  destino.
- El "antes" se captura **reproduciendo el fallo**, antes de arreglarlo — cuando es más barato.
- `assertions.md` marca cada chequeo `passed / failed / untested + motivo`; los saltos silenciosos
  están prohibidos.

### Minos (Inspector)
- Corre sobre código del que nunca habló con el constructor (contexto fresco: solo diff + spec).
- Emite score + observaciones; las observaciones deben justificar el score (un score bajo con lista
  vacía se rechaza como veredicto).
- Loop: score < 5 → vuelve a Hefesto con las observaciones como próxima spec. Tope: N iteraciones (4).
- Con 5/5 y cero pendientes, la unidad se le presenta a Zeus.

### Argos, Cerbero, Plutus, Ariadna
- Herramientas determinísticas: gates, timers, presupuesto y bitácora son scripts y archivos, sin LLM —
  son la maquinaria que las etapas con LLM no pueden falsear.