# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** Dos features acordadas con el usuario tras la sesión de brainstorm:
1. Que el Análisis con IA y la Vista de tendencia filtren la comparación por etiquetas
   (jefe/mazmorra/dificultad) en vez de mezclar todo el historial.
2. Selector de sesión cuando el log tiene varias con las mismas palabras clave, en vez de
   quedarse siempre con "la más reciente" en silencio.

**Alcance — qué toca / qué NO toca:**
- Toca: `buildAIPayload` (nuevo parámetro `tagFilter` + `filterHistoryByTags` reutilizable),
  `runAIAnalysis` (lee los campos de etiqueta ya existentes en el formulario de guardar, aunque
  no se haya guardado todavía), `buildAIPrompt` (explicar el nuevo campo `comparacion` a la IA);
  `renderTrendUI`/`drawTrendChart` (nuevo selector de etiqueta, reutiliza `filterHistoryByTags`);
  el flujo de `btnSubmit` (detectar TODAS las sesiones inicio/fin, no solo coger la última, y
  mostrar un selector si hay más de una antes de calcular).
- NO toca: `parseCombat`, `resolveSource` ni ningún regex de líneas de combate — el selector de
  sesión solo cambia qué rango de líneas se le pasa a `parseCombat`, no cómo se interpretan.
  Tampoco toca el formato guardado de `entry.tags`/`entry.roles`/historial existente — compatible
  con combates guardados antes de esta feature (sin etiquetas = sin filtro, comportamiento igual
  que hoy).
- No hace falta log real para probar ninguna de las dos — no se toca el formato de línea de
  combate, solo el filtrado de historial guardado (datos ya estructurados) y la detección de
  keywords (ya implementada, solo se expone al usuario en vez de auto-elegir).

**Hecho hasta ahora:** nada commiteado todavía.

**Falta por hacer:**
- 1/3: Filtro de etiquetas en Análisis con IA (`filterHistoryByTags`, payload con campo
  `comparacion`, prompt actualizado, línea de contexto en el resultado mostrando si se filtró o
  no, cacheKey incluye las etiquetas para no servir caché obsoleta si cambian).
- 2/3: Filtro de etiquetas en Vista de tendencia (select nuevo con combinaciones de etiquetas
  detectadas en el historial, reutiliza `filterHistoryByTags`).
- 3/3: Selector de sesión en la carga de log — detectar todos los pares inicio/fin posibles,
  si hay más de uno mostrar una lista para elegir cuál calcular en vez de auto-elegir el último.

**Dónde tocar:** `wakfu_calculadora_1.html` — `buildAIPayload`/`runAIAnalysis` (~línea 525-650),
`renderTrendUI`/`drawTrendChart` (~línea 850-1000), `btnSubmit.addEventListener` (~línea 1147).

**Último commit relevante:** 7e18be9 (cierre de la feature QoL, HANDOFF ya en "Nada en curso")

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto.

**Log de prueba usado:** ninguno necesario, no se toca el parser (ver "Alcance" arriba).

Última feature completada: mejoras de calidad de vida (QoL) en carga de log, tablas y
desplegables. Documentado en PROJECT.md, sección "Mejoras de calidad de vida (QoL)". Los 6
milestones commiteados y pusheados:
1/6 recordar palabras clave inicio/fin, 2/6 botón copiar en celdas con línea(s) de log,
3/6 buscador de jugador, 4/6 recordar desplegables abiertos (solo en memoria de sesión),
5/6 aviso si el archivo no parece un log de Wakfu, 6/6 copiar tabla principal como texto.
Nada roto, nada a medio probar. No tocó `parseCombat` ni `resolveSource` (el usuario autorizó
explícitamente saltarse la validación con log real para esta feature por no tocar el parser);
cada milestone sí se comprobó con `node --check` sobre el `<script>` y pruebas de lógica aisladas
con Node para las funciones puras nuevas (`looksLikeWakfuLog`, `buildMainTableText`, `copyBtn`,
guardado/carga de palabras clave).

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
