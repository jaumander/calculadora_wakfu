# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** Clic en un dato (Daño hecho/recibido, Curación hecha/recibida, Derribos,
Resurrecciones) → abre el Modo debug ya filtrado a los eventos exactos detrás de ese número, con
opción de incluir también los lanzamientos (casts) de ese jugador para verificar la atribución.

**Objetivo:** ver PROJECT.md, sección "Clic en un dato → modo debug filtrado (en curso — ver
ALCANCE)", con el detalle completo.

**Hecho hasta ahora:** nada commiteado todavía (acabo de escribir el alcance en PROJECT.md).

**Falta por hacer (milestones, ver PROJECT.md):**
1. `eventsForMetric`/`debugRowMatches` (funciones puras) + `data-source`/`data-target` en filas
   de debug + selects "Objetivo"/"Atribuido a" + checkbox "incluir lanzamientos".
2. Tooltips que faltan en las 4 columnas agregadas + clic-a-debug en las 6 columnas +
   `openDebugFiltered` + estilo visual de "esto es clicable".
3. Pulido + cerrar PROJECT.md.

**Dónde tocar:** `wakfu_calculadora_1.html` — función `renderResults` (tabla principal y sección
Modo debug están dentro de la misma función, buscar `debugFilterCat`/`applyDebugFilters`).

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto.

**Log de prueba usado:** ninguno necesario, no se toca el parser.

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
