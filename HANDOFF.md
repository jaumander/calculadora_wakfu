# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: EN CURSO

**Feature/tarea:** Desglose de daño/curación hecha por objetivo (roadmap, punto 1)

**Objetivo:** Por cada jugador, ver contra qué objetivo (mob/jugador enemigo) hizo cada golpe de
daño/curación y cuánto, no solo el total agregado.

**Hecho hasta ahora:**
- `parseCombat`: cada jugador tiene ahora `targets: {}` (vía `ensureTarget`), rellenado en la
  rama `pdvRe` usando el `source` ya resuelto (respeta las excepciones de Pulgas/aura con
  nombre — no un `currentCaster` en crudo).
- Validado con Node contra el `wakfu_chat.log` real subido en esta conversación: la suma de
  `targets[*].dmg`/`.heal` por jugador coincide exactamente con `dmgDone`/`healDone` — cero
  discrepancia, 18 jugadores, 0 ambiguos.
- Commit: `72c5dc3` — "WIP: 1/2 desglose por objetivo - trackear targets{dmg,heal,hits}..."

**Falta por hacer:**
- Añadir sección de render (tabla plegable "Ver desglose por objetivo" por jugador), siguiendo
  el mismo patrón que `spellSections`/`toggleSpells`/`spellsWrap` (buscar esas variables en
  `wakfu_calculadora_1.html`).
- Decidir si se ordena por daño o se deja alfabético por objetivo.
- Actualizar el checklist de "Estado actual" en PROJECT.md marcando esto como `[x]` al terminar.

**Dónde tocar:** `wakfu_calculadora_1.html`, función `renderResults` — justo después del bloque
que genera `spellSections` (buscar `let spellSections = ''`), y en el HTML del template
literal, justo después del bloque `spellsWrap`.

**Último commit relevante:** `72c5dc3`

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto — el parser ya está validado, solo
falta la parte visual.

**Log de prueba usado:** `wakfu_chat.log` subido por el usuario en esta conversación (2648
líneas, sin palabras clave inicio/fin — se probó pasando el archivo completo a `parseCombat`
directamente en vez de vía el flujo normal de la UI).

Última feature completada: excepción de atribución de curas de Pulgas (luz)/(fuego) a Laik, y su
extensión posterior a curas de auras con nombre de efecto explícito ("El Gatallón"), generalizada
vía el dueño de estado ya registrado en `stateOwners` (no hardcodeada al nombre "El Gatallón").
Pulgas requirió dos rondas de aclaración con el usuario (la petición original habría robado
curaciones legítimas a otros 7 jugadores; la aclaración final ancló la regla exactamente a la
etiqueta de elemento, usando la señal de "Racha Sanadora" solo como comprobación interna, no como
la regla en sí). Validado con Node contra `wakfu_chat.log` real: 11 eventos reatribuidos,
conservación exacta (lo que pierden unos es exactamente lo que gana Laik), cero falsos positivos
confirmados explícitamente contra las curaciones de Hidori/Cunejo que comparten esa misma etiqueta
por coincidencia. La extensión de El Gatallón solo tenía 1 ejemplo real en el log y ese caso ya
estaba bien atribuido por coincidencia (Laik era target y lanzador activo a la vez), así que se
validó con un caso sintético construido a mano además de la comprobación de regresión (Pulgas y
el total de Laik no cambian). Documentado en PROJECT.md, secciones "Excepción de atribución: curas
de Pulgas (luz)/(fuego) a Laik" y "Extensión: curas de auras con nombre de efecto explícito...".
Nada roto, nada a medio probar.

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
