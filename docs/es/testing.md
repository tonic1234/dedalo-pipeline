# Tests del pipeline — la aceptación de Dédalo

El pipeline se prueba igual que hace trabajar a sus unidades: **tests de integración contra el destino
real**, no unit tests de helpers. Un test del pipeline en verde significa que una unidad real recorrió
todo el flujo y los artefactos verifican.

## Suite de tests

### T1 — Piloto: un caso real de punta a punta
Una feature chica real recorre `spec → red → build → prove → review` hasta **5/5 con cero pendientes**
sin intervención manual salvo el merge de Zeus.

- **Aceptación**: un PR mergeado, todos los artefactos de handoff presentes, verify.sh en verde, score
  del inspector 5/5, bitácora completa.
- **Es el gate de todo el pipeline**: hasta que T1 no pasa, sigue viva la versión anterior.

### T2 — Anti-mentira: fix falso pre-sembrado
A una unidad se le siembra un **fix falso** (un stub que "funciona" sin hacer el trabajo) en el build.
El pipeline debe comportarse exactamente así:

- El inspector puntúa por debajo de 5, y
- la unidad **vuelve a build**, nunca se presenta para merge.
- **Aceptación**: el fix falso se detecta y la unidad re-loopa al menos una vez.

### T3 — Calibración del prompt del inspector
El inspector revisa **5 PRs ya mergeados del historial real**. Expectativa: los PRs buenos puntúan alto
(4–5), los débiles más bajo. Si un PR claramente bueno puntúa < 4 sistemáticamente, el **prompt está
mal calibrado — se arregla el instrumento, no el código** (regla: instrumento antes que veredicto).

### T4 — Watchdog y límites
- Timeout por etapa, tope duro de iteraciones de revisión, corte de corrida al 100% del presupuesto.
- Cada límite se ejercita una vez: una traba escenificada debe producir **alerta por Telegram** y estado
  reanudable.

### T5 — Regresión
`verify.sh` en verde (tests, lint, tipos, escaneo de secretos), gate Argos en verde, sin secretos en el
repo, doc en sincronía con el código (EN/ES).

## Detalle del test anti-mentira

Las mentiras que la suite debe detectar:

| Mentira | Cómo se detecta |
|---|---|
| "Funciona" sin haberlo corrido | T1/T2: artefactos validados por la etapa receptora, nunca auto-reportados |
| Fix falso (stub) | T2: el inspector revisa el diff contra la spec, no la palabra del agente |
| Salto silencioso | Cualquier `untested` sin motivo falla el chequeo de assertions de Alétheia |
| "Funciona pero es basura" | Reglas del estándar code-structure del inspector: duplicación, dead code, funciones dios |
| Evidencia floja pero plausible | El humano (Zeus) lee el par antes/después en el momento del merge |

## Instrumento antes que veredicto

Cada vez que un score o una tasa de fallos parece rara, **auditar primero el instrumento**: ¿la medición
corre el pipeline completo con sus capas y precedencias, o una vista pelada? Un instrumento equivocado
ya produjo dos "bugs" falsos y decenas de "no entendió" falsos — la regla es verificar el instrumento
contra una unidad conocida-buena antes de tocar el motor.