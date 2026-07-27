# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: En curso

**Feature/tarea:** Modo debug que aprende (correcciones de atribución persistentes)

**Objetivo:** Cuando el usuario confirma en el modo debug quién debería llevarse el mérito de un
evento con nombre de estado/aura reconocido, esa corrección se guarda y se aplica sola en
próximos combates — sin tener que pasarme el JSON exportado para que yo ajuste el código a mano.
Alcance deliberadamente limitado: la corrección es solo un mapa `{ nombreEstado: nombreJugador }`
persistente, usado únicamente dentro de `resolveSource`. No puede tocar regex, categorías,
cantidades ni ningún otro comportamiento de la app — así el usuario no puede romperla por mucho
que "aprenda" cosas raras, como mucho atribuye mal un evento concreto.

**Hecho hasta ahora:**
- `parseCombat(lines, resolutions)` acepta un segundo parámetro con las correcciones aprendidas;
  `resolveSource` las consulta ANTES de mirar `stateOwners`, así ganan siempre que existan.
  Validado con Node: sin corrección el evento con 2 dueños posibles sigue quedando ambiguo (como
  antes); con corrección guardada se atribuye solo, sin regresión en el caso de dueño único.
- `logEvent`/`resolveSource` ahora exponen `effName` en cada `debugEvent` cuando el evento viene
  de un nombre entre paréntesis, para saber en qué filas tiene sentido ofrecer "aprender".
- Storage: `loadResolutions()`, `saveResolution(effName, playerName)`, `deleteResolution(effName)`
  sobre la clave `stateResolutions` (mismo patrón que el historial: `window.storage` si está
  disponible, no-op si no).
- Modo debug: nueva columna "Aprender" — solo aparece si `ev.effName` existe, con un `<select>`
  de jugadores del combate actual + botón "Confirmar" que llama a `saveResolution` y deshabilita
  la fila tras guardar.
- Nota explicativa añadida arriba de la tabla de debug aclarando el alcance de "Aprender".
- `btnSubmit` ahora es async y carga `loadResolutions()` antes de llamar a `parseCombat`.
- Todo commiteado y pusheado a `main` (ver "Último commit relevante").

**Falta por hacer:**
- Añadir la tarjeta "Correcciones de atribución aprendidas" (siempre visible, como la de
  Historial) que liste las guardadas con botón de borrar, y complete el stub actual de
  `renderResolutionsList()` (ahora mismo hace un no-op defensivo si no encuentra
  `#resolutionsList` en el DOM — el contenedor aún no existe).
- Llamar a `renderResolutionsList()` también al cargar la página (igual que `renderHistoryList()`).
- Probar el flujo completo en el navegador (no solo con Node): marcar "Confirmar" en una fila con
  effName, comprobar que aparece en la tarjeta nueva, recargar y comprobar que persiste.
- Actualizar `PROJECT.md` con la sección de esta feature y su alcance limitado, y marcar el ítem
  correspondiente en "Estado actual".
- Cuando esté todo, vaciar este HANDOFF a "Nada en curso".

**Dónde tocar:** `wakfu_calculadora_1.html` — funciones `renderResolutionsList` (stub, buscar
"Stub defensivo"), `loadResolutions`/`saveResolution`/`deleteResolution` (junto a las de
historial), y la tarjeta `historyCard` en el HTML (la nueva tarjeta de resoluciones debería ir
justo después, mismo patrón visual).

**Último commit relevante:** "WIP: modo debug que aprende (3/4) - columna Aprender en el modo
debug + wiring de saveResolution" (ver el commit anterior "(1/4)" para la capa de datos)

**Cualquier cosa rota/a medio probar ahora mismo:** nada roto — `renderResolutionsList()` es un
stub seguro (no-op) hasta que se añada su contenedor en el DOM en el siguiente milestone. El botón
"Confirmar" ya guarda de verdad en `window.storage`, solo falta la vista para consultarlo/borrarlo.

**Log de prueba usado:** validado con logs sintéticos en Node (dos "dueños" del mismo nombre de
estado + evento ambiguo), no con un log real de combate — no hay ninguno adjunto en esta sesión.
No afecta a `parseCombat` en sí (que sigue validado contra combates reales de sesiones previas),
solo a la capa nueva de resolución.

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
