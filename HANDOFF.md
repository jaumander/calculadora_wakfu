# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** IA que analiza el historial y da consejos (botón "Analizar con IA")

**Objetivo:** En el panel de resultados de un combate recién calculado, un botón que llama a
la API de Claude (fetch directo desde el HTML, sin API key — usa el uso normal de la cuenta,
igual que el resto de "AI-powered artifacts") y devuelve 2-3 consejos concretos, comparando el
combate actual contra la media del historial guardado (no solo los totales sueltos).

**Alcance (qué toca / qué NO toca):**
- Toca: el panel de resultados (`renderResults`) — nueva tarjeta "Análisis con IA" con botón y
  área de resultado; nuevas funciones puras `buildAIPayload`/`aiDeltaSummary` para calcular
  medias históricas y deltas por jugador; una función que hace el `fetch` a
  `https://api.anthropic.com/v1/messages`.
- NO toca: `parseCombat`, `resolveSource`, la comparativa de 2 combates, la vista de tendencia,
  el modo debug, las etiquetas, ni el formato de guardado del historial (el resultado de la IA
  se guarda como campo opcional aparte, así que combates antiguos no se rompen).
- Limitación conocida a documentar: solo funciona dentro del panel de chat de Claude (con
  "Create AI-powered Artifacts" activo en Settings). Si se abre el `.html` suelto, el botón debe
  avisar en vez de fallar en silencio o lanzar un error confuso.

**Hecho hasta ahora:**
- (nada commiteado todavía, empezando milestone 1/2)

**Falta por hacer:**
- 1/2: `buildAIPayload` (media histórica + deltas por jugador, probado con Node) + tarjeta HTML
  del botón + wiring del `fetch` real a la API + parseo/render de la respuesta + manejo de error.
- 2/2: cachear el resultado por combate (`window.storage`), nota de limitación cuando no hay
  storage, actualizar `PROJECT.md` (marcar como hecha en el checklist).

**Dónde tocar:** `wakfu_calculadora_1.html` — tarjeta nueva junto a "Guardar este combate en el
historial" (buscar `btnSaveHistory`), funciones nuevas cerca de `combatTotals`.

**Último commit relevante:** 7140604 (vista de tendencia, feature anterior ya cerrada)

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto — el objetivo del milestone 1 es
dejar el botón funcionando de verdad, no un stub.

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
