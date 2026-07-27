# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** Clasificación aliado/enemigo por combate del historial, para dejar de
amalgamar sus totales en la comparativa (y en la línea resumen del historial).

**Objetivo:** ver PROJECT.md, sección "Clasificación aliado/enemigo (en curso — ver ALCANCE)",
que tiene el detalle completo del problema, la solución y qué toca/no toca.

**Hecho hasta ahora:**
- 1/3: `subsetPlayers`, `splitByRole` (helpers puros, probados con Node), modal reutilizable
  `openRoleModal(entryIds, onSaved)` con overlay + toggles Aliado/Enemigo, botón "Clasificar"
  por entrada del historial que lo abre para ese combate. Persiste en `entry.roles`.
- 2/3: `renderComparison` ya no amalgama — "Total del grupo" se divide en dos tablas (Aliados /
  Enemigos), aviso "N sin clasificar" con enlace que abre el modal para los 2 combates
  comparados a la vez, y etiqueta de bando junto al nombre en "por jugador" (con aviso si el
  mismo nombre tiene bando distinto en cada combate). Probado con Node reproduciendo el caso
  exacto del screenshot original (daño hecho == daño recibido a nivel de grupo) y confirmado que
  ya no ocurre tras separar por bando.

**Falta por hacer (milestones, ver PROJECT.md):**
3. Pulido: `roles: {}` por defecto al guardar un combate nuevo, aviso de sin clasificar también
   en la línea resumen del historial (no solo en la comparativa), actualizar PROJECT.md a
   estado terminado.

**Dónde tocar:** `wakfu_calculadora_1.html` — bloque `btnSaveHistory` (para `roles: {}`) y
`renderHistoryList` (para el aviso de sin clasificar en la línea resumen).

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto.

**Log de prueba usado:** ninguno necesario, no se toca el parser.

---

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
