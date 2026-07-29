# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso — BLOQUEADA esperando un archivo

**Feature/tarea:** excepción de atribución para la autocuración indirecta de "Pulgas" (Laik).
Ver PROJECT.md, sección "Excepción de atribución: autocuración indirecta de 'Pulgas'".

**Objetivo:** cuando un jugador "lleva pulgas" (debuff aplicado por Laik) y recibe una curación
pasiva sin nombre de efecto al terminar su turno, esa curación debería atribuirse a Laik (quien
aplicó Pulgas), no al lanzador activo del motor actual.

**Hecho hasta ahora:** investigación completa contra un log real de OTRA conversación (ya no
adjunto aquí): se descartó la petición original tal cual (atribuir toda cura luz/fuego a Laik
habría robado curaciones legítimas a otros 7 jugadores), se confirmó con el usuario que el caso
real es más concreto (proc de Pulgas), y se aislaron 67 líneas candidatas — pero con una duda sin
resolver (ver PROJECT.md) que necesita el log real para separar curaciones directas legítimas de
procs de Pulgas. Nada de código tocado todavía, nada commiteado en `wakfu_calculadora_1.html`.

**Falta por hacer:** pedirle a quien retome esto (o a la persona) que adjunte `wakfu_chat.log`
(o un log real equivalente con Pulgas) en la conversación, confirmar el formato exacto de la
línea de aplicación de Pulgas, resolver la duda de curación-directa-vs-proc línea a línea contra
ese log, diseñar e implementar la excepción puntual en `parseCombat`, y validarla con Node antes
de entregar.

**Dónde tocar:** `wakfu_calculadora_1.html` — `parseCombat`, rama `pdvRe` con signo `+`
(curación), y el sistema de `stateOwners`/`resolveSource` como referencia de patrón a reutilizar
(no reutilizar literalmente, la curación directa no pasa por ahí hoy).

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto — no se ha tocado código todavía,
precisamente para no adivinar el regex de Pulgas sin el log real.

**Log de prueba usado:** ninguno disponible en esta conversación. Se usó `wakfu_chat.log` en una
conversación anterior (grep pegado en el chat), pero el archivo en sí no está adjunto aquí —
hace falta volver a subirlo para continuar.

## Plantilla a rellenar cuando se deja algo a medias

**Feature/tarea:** (nombre corto)

**Objetivo:** (1-2 frases, qué se quiere conseguir)

**Hecho hasta ahora:**
- (lista concreta de sub-pasos ya completados y commiteados)

**Falta por hacer:**
- (lista concreta, lo más específica posible — "arreglar el regex de X" mejor que "seguir con X")

**Dónde tocar:** (archivo + función/sección aproximada, ej. "wakfu_calculadora.html, función resolveSource, buscar 'papmRe'")

**Último commit relevante:** (hash o mensaje)

**Cualquier cosa rota/a medio probar ahora mismo:** (ej. "el debug mode no filtra bien por categoría todavía")

**Log de prueba usado:** (si se ha usado un log real para validar, indicar cuál/qué combate, para no pedir otro innecesariamente)

---

## Protocolo (para cualquier sesión/cuenta que entre)

1. `git pull` primero, siempre.
2. Lee este archivo. Si dice "Nada en curso", trabaja normal desde PROJECT.md.
3. Si hay algo en curso y es tuyo continuarlo: sigue desde "Falta por hacer".
4. Trabaja en commits pequeños con prefijo `WIP:` según completes cada sub-paso —
   no esperes a terminar toda la feature para hacer commit/push.
5. Actualiza este archivo (o vacíalo a "Nada en curso" si terminaste) en el MISMO
   commit final antes de que acabe la sesión. Nunca dejar código sin pushear.
6. Rama: si la feature es grande o puede chocar con otro trabajo en paralelo, usar
   `feature/nombre-corto` en vez de tocar `main` directamente.
