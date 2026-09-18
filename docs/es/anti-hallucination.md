# Capas anti-alucinación — cómo Dédalo sobrevive a las mentiras de los LLM

A los agentes LLM se los puede empujar a mentir, a creer cosas falsas o a saltarse trabajo en silencio
("los agentes no pueden pinky-promise"). Dédalo no intenta que los agentes sean honestos; hace que
**mentir sea estructuralmente caro y visible**. Siete capas:

## 1. Nadie es su propia fuente
Todo "listo / verde / funciona" es un artefacto crudo (log, listado, conteos, score) **validado por la
etapa receptora**, que re-corre el chequeo ella misma. Un agente nunca reporta su propio resultado.

## 2. El inspector es un desconocido para el constructor
Minos ve solo el **diff y la spec** — contexto fresco, sin conversación compartida, memoria distinta.
Una mentira no puede coordinarse con el juez, y un error cometido al construir no contamina la
revisión. El juez lee el código, no las afirmaciones del constructor.

## 3. La omisión es visible, nunca silenciosa
`assertions.md` marca cada chequeo `passed / failed / untested + motivo`.
"Lo que no corrí pero dije que corrí" deja de ser silencio: es un hueco anotado que bloquea el handoff.

## 4. El score tiene que justificarse
Un score bajo con lista de observaciones vacía se **rechaza como veredicto**. El juez tiene que dar
cuenta del score; la bitácora registra ambos.

## 5. "Funciona pero es basura" es una categoría que el juez chequea
Los modelos hacen el trabajo de la forma más sucia aceptable si pueden. Minos revisa contra un estándar
de arquitectura escrito (capas de servicio, sin funciones dios, sin duplicación, sin dead code) — las
cosas que un modelo no se auto-juzga bien sin guía escrita.

## 6. Mentir sostenidamente es caro
Tope duro de iteraciones, timeout por etapa, tope de presupuesto con corte al 100%, y un watchdog que
avisa si se traba. Una cadena de mentiras quema iteraciones y presupuesto, dispara la alarma y queda en
la bitácora — el sistema convierte las mentiras en ruido visible en vez de absorberlas.

## 7. El merge es humano
La única etapa sin automatización. Zeus no lee código línea por línea: lee el par de evidencia
antes/después y el score, y hace click — o devuelve la unidad.

## Lo que estas capas NO hacen (límites honestos)

- No vuelven perfectos a los modelos como revisores — el test de calibración (T3) fija el piso aceptable.
- No reemplazan los tests reales: `verify.sh` y los tests de integración contra el destino real siguen
  siendo la verdad final de "¿funciona?".
- El loop de revisión detecta mentiras estructurales y de consistencia; el par de evidencia detecta "el
  cambio no es lo que afirmé". El humano sigue siendo la última línea para los juicios de valor.