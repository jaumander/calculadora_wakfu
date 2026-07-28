# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: EN CURSO

**Feature/tarea:** Sesiones de mazmorra

**Objetivo:** agrupar varios combates del historial en una "sesión" (mazmorra entera o varios
intentos al mismo jefe) y ver el total agregado, no solo combate a combate. Alcance completo en
PROJECT.md, sección "Sesiones de mazmorra — ALCANCE".

**Hecho hasta ahora:** nada todavía, empezando milestone 1/4.

**Falta por hacer:**
- Milestone 1/4: `loadSessions()`/`saveSessionsList()` + quitar el límite de 2 en las casillas
  del historial + botón "Agrupar seleccionados en una sesión" cuando hay 2+.
- Milestone 2/4: tarjeta "Sesiones de mazmorra" con total agregado y combates que la componen.
- Milestone 3/4: mejor/peor combate de la sesión + manejo de IDs huérfanos.
- Milestone 4/4: export/import de sesiones.

**Dónde tocar:** wakfu_calculadora_1.html — `renderHistoryList()`/`updateCompareUI()` (checkboxes
y límite de 2), cerca de `loadHistory()`/`saveHistoryList()` para añadir las funciones gemelas de
sesiones, y junto a `resolutionsCard`/`historyCard` en el HTML para la tarjeta nueva.

**Cualquier cosa rota/a medio probar ahora mismo:** nada, todavía no se ha tocado código.

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
