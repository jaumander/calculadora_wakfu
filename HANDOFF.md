# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** Hacer standalone las dos features pendientes (ver PROJECT.md, sección
"Principio de diseño: debe funcionar también fuera del panel de chat") — decisión ya tomada:
guardar la clave de la API en `localStorage` (opción cómoda, no la conservadora).

**Objetivo:**
1. Análisis con IA: fuera del panel, pedir una clave de Anthropic, guardarla en `localStorage`,
   y usarla en el `fetch` con las cabeceras BYOK (`x-api-key`, `anthropic-version`,
   `anthropic-dangerous-direct-browser-access`).
2. Correcciones aprendidas: fuera del panel, usar `localStorage` en vez de perderse en memoria.

**Alcance (qué toca / qué NO toca):**
- Toca: `runAIAnalysis` (cabeceras condicionales), nueva tarjeta de IA dividida en un contenedor
  `#aiCardBody` que se puede re-renderizar solo (sin re-pintar toda `renderResults`) al
  guardar/olvidar la clave; `loadResolutions`/`saveResolution`/`deleteResolution` con fallback a
  `localStorage`; nota junto a la tarjeta de correcciones igual que la del historial.
- NO toca: `parseCombat`, `resolveSource`, comparativa, tendencia, formato de guardado del
  historial, ni el comportamiento dentro del panel de chat (debe seguir funcionando exactamente
  igual que antes cuando `hasStorage()` es true).

**Hecho hasta ahora:** nada commiteado todavía.

**Falta por hacer:**
- 1/2: `hasLocalStorage()`, `getStoredApiKey`/`saveApiKey`/`clearApiKey`, `buildAIHeaders(apiKey)`
  (pura, testable con Node), `renderAICard()` como sub-render independiente, wiring del
  formulario de clave, `runAIAnalysis` actualizado para usar cabeceras BYOK cuando corresponda.
- 2/2: fallback a `localStorage` en las 3 funciones de correcciones aprendidas + nota en su
  tarjeta + actualizar PROJECT.md (marcar ambas como resueltas, ya no "pendiente de decisión").

**Dónde tocar:** `wakfu_calculadora_1.html` — buscar `btnAnalyzeAI`/`runAIAnalysis` y
`loadResolutions`/`saveResolution`/`deleteResolution`.

**Último commit relevante:** (ver `git log`, el de "Docs: principio de diseño standalone...")

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
