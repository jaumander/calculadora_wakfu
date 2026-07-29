# Calculadora de Combate Wakfu (ES) — validada con logs reales

Herramienta inspirada en [MethodWakfu Companion](https://companion.methodwakfu.com/), reconstruida
para procesar logs de chat de Wakfu **en español**, ya que la original solo soporta inglés/francés.

## Objetivo

A partir de `wakfu_chat.log` y dos palabras clave de inicio/fin (escritas en el chat del juego
antes y después del combate), extraer esa sesión y calcular por jugador: daño hecho/recibido,
curación hecha/recibida, % de crítico real, derribos, resurrecciones, armadura, golpes
anticipados, debuffs de característica, buffs/debuffs porcentuales por categoría (con turnos), y
desglose de daño/curación por hechizo con su elemento dominante.

Todo se procesa en el propio navegador; no se sube nada a ningún servidor.

## Principio de diseño: debe funcionar también fuera del panel de chat (standalone)

La herramienta se distribuye como un único `.html` sin dependencias — parte de su razón de ser es
que cualquiera pueda descargarlo y abrirlo directamente en su navegador, sin depender de estar
dentro del panel de chat de Claude. Por eso, **ninguna feature debe depender EXCLUSIVAMENTE de una
capacidad que solo existe dentro del artifact** (`window.storage`, la llamada a la API de Claude
sin clave) sin ofrecer una alternativa razonable fuera de él, aunque sea más manual. El patrón ya
usado para el historial (guardado automático dentro del panel / exportar-importar JSON fuera) es
el modelo a seguir: detectar el contexto con `hasStorage()` y degradar de forma explícita y
avisada, nunca en silencio.

**Estado real a día de hoy — las tres features ya cumplen:**
- ✅ **Historial persistente**: botones "Exportar historial"/"Importar historial" como
  alternativa fuera del panel, con nota explicando la limitación.
- ✅ **Correcciones de atribución aprendidas** (`stateResolutions`): fuera del panel
  (`!hasStorage()`), `loadResolutions`/`saveResolution`/`deleteResolution` leen y escriben en
  `localStorage` (clave `wakfu_calc_resolutions`) en vez de quedarse solo en memoria. La tarjeta
  muestra una nota avisando de que, fuera del panel, se guardan solo en ese navegador.
- ✅ **Análisis con IA**: fuera del panel funciona con "trae tu propia clave" (BYOK). Se eligió
  la opción cómoda: la clave se guarda en `localStorage` (clave `wakfu_calc_api_key`), nunca
  viaja dentro del `.html` en sí (vive en el navegador de quien la pegó, no en el archivo — quien
  reciba el `.html` compartido tendría que pegar la suya propia). La tarjeta de IA
  (`renderAICard`, en un contenedor `#aiCardBody` que se re-renderiza solo, sin repintar toda la
  tabla de resultados) tiene 3 modos: dentro del panel (botón directo, sin clave, como siempre);
  fuera del panel sin clave guardada (campo para pegarla + "Guardar clave"); fuera del panel con
  clave guardada (botón directo + enlace "Olvidar clave"). Las cabeceras del `fetch` cambian según
  el caso (`buildAIHeaders`, función pura): sin clave, solo `Content-Type`; con clave, se añaden
  `x-api-key`, `anthropic-version: 2023-06-01` y `anthropic-dangerous-direct-browser-access: true`
  — el mecanismo oficial de Anthropic para llamar a la API directamente desde el navegador.
  **Matiz de seguridad conocido y aceptado:** el origen de `localStorage` para páginas `file://`
  es inconsistente entre navegadores (en algunos, todas las páginas locales abiertas así pueden
  compartir almacenamiento); para uso personal en un único navegador el riesgo se consideró bajo
  frente a la comodidad de no tener que pegar la clave en cada análisis.

## Formato del log — confirmado con datos reales, no supuesto

A diferencia de una reconstrucción basada en documentación de terceros, cada patrón de este
parser se ha contrastado línea por línea contra **múltiples combates reales** subidos por el
usuario. Formato base de línea:

```
HH:MM:SS,mmm - [Canal] Autor : mensaje
```

Patrones confirmados dentro de `[Información (combate)]`:

| Elemento | Texto confirmado |
|---|---|
| Unidad de vida | `PdV` (no `PV`) |
| Lanzamiento de hechizo | `X lanza el hechizo Y.` (+ `(Crítico)` si crítico) |
| Elemento del golpe | primer paréntesis tras PdV: `fuego, agua, tierra, aire, neutral, luz` |
| Derribo (jugador) | `¡X está K.O.!` |
| Derribo (invocación/mob, variante 1) | `¡X está fuera de combate!` |
| Derribo (invocación/mob, variante 2) | `X: está fuera de combate` (sin exclamación) |
| Resurrección | `X: ha sido reanimado (Hechizo)` |
| Armadura | `X: N Armadura (Origen)` |
| Nivel de estado/aura | `X: NombreEstado (Niv. N)` — usado para inferir el "dueño" del aura |
| Debuff de resistencia | `X: -N resistencia elemental (Origen)` |
| Debuff PA/PM/PW | `X: -N (PA\|PM\|PW)` — **idéntico al gasto propio de recursos**; solo se cuenta si el objetivo ≠ lanzador actual |
| Buffs/debuffs % | `X: [-]N% de (daños infligidos\|daños causados\|golpe crítico\|curas realizadas\|curas recibidas\|armadura recibida\|anticipación)`, con `(N turnos)` opcional |

## Motor de atribución (quién se lleva el mérito de cada evento)

Regla base: se atribuye al **lanzador activo** (quien lanzó el último hechizo antes del evento).

**Problema detectado y resuelto — efectos indirectos/de zona (ej. árbol de Sadida "Guarda
Plantas"):** un aura sigue activa turnos después de lanzada, así que cuando OTRO jugador la pisa,
"el lanzador activo" es quien esté jugando en ese instante, no quien creó el aura. Solución
implementada:

1. Cuando un estado aparece por primera vez (`Niv. 1`), se registra quién tenía el turno como
   "dueño" de ese nombre de estado.
2. Cada vez que ese mismo nombre de estado vuelve a activarse (en cualquier objetivo), se busca
   su dueño registrado y se le atribuye a él, no al lanzador activo del momento.
3. **Si hay 2+ dueños posibles para el mismo nombre de estado** (ej. dos Sadidas con el mismo
   aura), no se adivina: el evento se marca como **ambiguo**, no se atribuye a nadie, y la app
   avisa arriba de la tabla con el detalle de los candidatos.

Este mismo sistema (`resolveSource`) se usa para armadura, resistencia, PA/PM/PW y los
buffs/debuffs %. El daño/curación directos NO pasan por este sistema — usan solo la heurística
de lanzador activo, que es fiable porque son el resultado directo de una acción, no un efecto
retardado de zona.

## Historial persistente y comparativa entre combates

Cada vez que se calcula un combate, aparece un bloque "Guardar este combate en el historial"
donde el usuario le pone un nombre (ej. "Ereshkigal intento 3") y lo guarda. El historial vive
en una tarjeta aparte, siempre visible, con la lista de combates guardados (nombre, fecha, MVP
de daño, derribos totales) y un checkbox por combate.

- **Guardado automático**: si la herramienta corre dentro del panel de chat de Claude (hay
  `window.storage` disponible), cada combate guardado persiste entre sesiones sin que el usuario
  haga nada más.
- **Uso como archivo suelto**: si el usuario abre el `.html` descargado directamente en su
  navegador, `window.storage` no existe — el historial de esa sesión se pierde al recargar. Para
  ese caso están los botones "Exportar historial" (descarga `wakfu_historial.json` con todos los
  combates) e "Importar historial" (fusiona un `wakfu_historial.json` con lo que ya hubiera,
  sin duplicar por id). La app avisa de esta limitación con una nota junto al historial cuando
  detecta que no hay almacenamiento persistente disponible.
- **Comparativa**: al marcar exactamente 2 combates del historial aparece una tabla de
  diferencias — primero el total del grupo (daño hecho/recibido, curación, derribos) y luego,
  por cada jugador presente en **ambos** combates, daño hecho/recibido, curación hecha, % de
  crítico real y derribos, con flecha (▲/▼), delta absoluto y % de cambio, coloreado en verde si
  es una mejora y en rojo si es un empeoramiento (la dirección "buena" depende de la métrica: p.
  ej. subir daño hecho es bueno, subir daño recibido es malo). Los jugadores que solo aparecen en
  uno de los dos combates se listan aparte, sin comparar.

**Etiquetas opcionales**: al guardar un combate se pueden rellenar tres campos opcionales —
Jefe, Mazmorra, Dificultad (texto libre, sin lista cerrada). Se muestran bajo el nombre del
combate en el historial y en la cabecera de la comparativa. Los combates guardados antes de
esta feature simplemente no tienen etiquetas y se muestran igual, sin romperse.

- **Vista de tendencia**: bajo un desplegable "Ver vista de tendencia", en cuanto hay 3+
  combates guardados aparece un gráfico de líneas (SVG, sin librerías externas) con dos
  selectores — jugador (o "Total del grupo") y métrica (daño hecho/recibido, curación
  hecha/recibida, derribos, % de crítico real) — que dibuja la evolución de esa métrica a lo
  largo de los combates guardados, ordenados por fecha. Cada punto se colorea en verde/rojo
  según si mejora o empeora respecto al punto anterior (misma lógica de "dirección buena"
  que en la comparativa) y al pasar el ratón muestra el nombre y fecha del combate. Si el
  jugador elegido no participó en algún combate guardado, ese combate simplemente no aporta
  punto a la línea (no se inventa un cero ni rompe el gráfico) — por eso hacen falta al menos
  2 combates con datos de ese jugador/métrica para poder dibujar algo. Con menos de 3
  combates en el historial se muestra un aviso en vez del gráfico.

- **Análisis con IA**: en el panel de resultados de un combate recién calculado, junto a
  "Guardar en historial", hay un botón "Analizar con IA". Al pulsarlo, la propia herramienta
  llama directamente a la API de Claude (sin clave, sin coste aparte — consume el uso normal
  de la cuenta con la que se está usando la herramienta, igual que cualquier "AI-powered
  artifact") pasándole, por cada jugador del combate actual, sus números junto con la media de
  ese mismo jugador en el historial guardado (si existe) y el cambio porcentual frente a esa
  media. Con eso, la IA devuelve hasta 3 consejos concretos en texto, priorizando los cambios
  más grandes y el peor rendimiento relativo — no consejos genéricos tipo "sigue practicando".
  Si un jugador no tiene combates previos guardados, se le indica explícitamente a la IA que no
  invente una comparación para él. La respuesta se cachea en memoria durante la sesión (no en
  `window.storage`) para no repetir la llamada por accidente si se pulsa el botón dos veces;
  hay un enlace "Volver a generar" para forzar una llamada nueva a propósito. Dentro del panel de
  chat de Claude funciona directamente, sin clave. Fuera del panel (`.html` suelto), usa BYOK con
  una clave propia guardada en `localStorage` — ver "Principio de diseño: debe funcionar también
  fuera del panel de chat" al principio de este documento para el detalle.

## Clasificación aliado/enemigo

**Problema:** el chat del juego no distingue quién es aliado y quién es enemigo — el log solo
dice nombres. Por eso los totales de grupo (`combatTotals`, usados en la línea resumen del
historial y en "Total del grupo" de la comparativa) sumaban a todo el mundo junto: el daño hecho
por tu equipo y el daño hecho por los enemigos que os pegan entre ellos, etc. El síntoma más
visible era que, en la comparativa, "Daño hecho" del grupo y "Daño recibido" del grupo salían
siempre con el mismo número exacto, porque todo daño hecho por alguien es recibido por alguien
más dentro del mismo conjunto de nombres.

**Solución:** como no se puede inferir del texto del log (ver "Formato del log" — no hay ninguna
marca de bando), la clasificación es manual: un botón "Clasificar" en cada combate del historial
(y también un enlace directo desde el aviso de la comparativa) abre un modal que lista los
nombres implicados y deja marcar cada uno como Aliado o Enemigo. Se guarda en el propio combate
(`entry.roles = { nombreJugador: 'aliado' | 'enemigo' }`), así que se clasifica una vez y queda
guardado — los combates guardados antes de esta feature simplemente no tienen `roles` y se tratan
como si todo estuviera sin clasificar, sin romperse. Nunca se adivina: un nombre sin marcar queda
"sin clasificar" y se excluye de los totales de Aliados/Enemigos (mismo espíritu que los eventos
"ambiguos" de `resolveSource` — no atribuir con una suposición).

- **En el historial**: la línea resumen de cada combate avisa de cuántos jugadores están sin
  clasificar (si hay alguno).
- **En la comparativa**: "Total del grupo" ya no es una única tabla mezclada — son dos tablas,
  "Total del grupo — Aliados" y "Total del grupo — Enemigos" (calculadas filtrando por
  `entry.roles` con el helper `subsetPlayers`). Si hay jugadores sin clasificar entre los dos
  combates comparados, aparece un aviso con un enlace que abre el modal ya cargado con la unión
  de nombres de ambos combates. En la sección "por jugador" (comparación individual, sin cambios
  en su cálculo) se añade una etiqueta de bando junto al nombre; si el mismo nombre tiene un
  bando distinto en cada uno de los dos combates comparados, se avisa en vez de mostrar uno
  arbitrario.
- El modal (`openRoleModal`) es reutilizable: acepta 1 combate (desde el historial) o 2 (desde
  la comparativa) y, al guardar, aplica el bando elegido a ese nombre en todos los combates que
  se le pasaron y que lo contienen — se asume que un mismo personaje es del mismo bando en los
  combates que se están clasificando juntos.

**Limitación conocida, fuera de alcance a propósito:** la vista de tendencia (opción "Total del
grupo") todavía amalgama aliados y enemigos igual que antes de esta feature — no se tocó para no
ampliar el alcance. Si hace falta separarla también, sería una feature aparte reutilizando
`splitByRole`/`subsetPlayers`.

## Modo debug que aprende (alcance deliberadamente limitado)

Antes, cada bug marcado en el modo debug se exportaba a JSON y había que pasárselo a Claude para
que ajustara el código a mano. Ahora, para un tipo concreto de error — la atribución de un
evento a un jugador equivocado (o a nadie, por ambigüedad) — el usuario puede confirmarlo él
mismo desde la propia tabla del modo debug, y esa corrección se aplica sola en los próximos
combates.

**Por qué el alcance está limitado a esto y no a "cualquier bug":** dejar que el usuario corrija
libremente cualquier aspecto (regex de parseo, categorías, cantidades…) desde la interfaz abriría
la puerta a que una corrección mal hecha rompa el parser para todo el mundo. En vez de eso:

- Lo único que se guarda es un mapa `{ nombreEstado: nombreJugador }` (clave `stateResolutions`
  en el mismo almacenamiento persistente que el historial).
- Solo se puede confirmar en filas del modo debug donde el evento venga de un nombre de
  estado/aura ya reconocido entre paréntesis (columna "Aprender": un `<select>` con los jugadores
  del combate actual + botón "Confirmar" — nunca texto libre).
- `resolveSource` consulta ese mapa antes que la heurística de "dueño del estado"; si hay una
  corrección guardada para ese nombre, gana siempre, sin volver a preguntar.
- No puede alterar cantidades de daño/curación, categorías, ni ningún regex de parseo — como
  mucho, en el peor caso, sigue atribuyendo mal un evento concreto, igual que antes.
- Cualquier otro tipo de bug (parseo incorrecto, categoría mal detectada, etc.) sigue el flujo
  manual de siempre: marcar 🐞, anotar por qué, exportar y pasárselo a Claude.

Hay una tarjeta siempre visible ("Correcciones de atribución aprendidas") que lista todo lo
aprendido hasta ahora y permite "Olvidar" cualquier corrección si resultó ser un error.

## Modo debug

Sección plegable que expone, evento por evento: la línea exacta del log, la categoría, el
objetivo, a quién se atribuyó y la razón textual generada por `resolveSource`/la heurística de
turno. Cada evento se puede marcar con un checkbox 🐞 como "posible bug", lo que despliega una
caja de texto para anotar el motivo. Botón "Exportar eventos marcados" descarga un JSON con
línea + categoría + atribución + razón del sistema + nota del usuario, pensado para pegárselo a
Claude y depurar casos concretos juntos.

## Clic en un dato → modo debug filtrado (en curso — ver ALCANCE)

**Objetivo:** hoy el hover ya enseña las líneas de log detrás de un dato (derribos,
resurrecciones, mods, hechizos), pero no las 4 columnas agregadas de la tabla principal (Daño
hecho/recibido, Curación hecha/recibida) — y en ningún caso hay forma de saltar de un dato al
Modo debug ya filtrado a esas líneas exactas. Se pide: clic en un dato → abre el Modo debug
filtrado exactamente a los eventos que lo componen. Además, para el caso de daño/curación hechos,
poder comparar esos eventos con los de lanzamiento (cast) atribuidos a ese mismo jugador, para
verificar visualmente si la atribución de lanzador fue correcta.

**Solución:**
- Se añaden tooltips a las 4 columnas agregadas que hoy no lo tienen (Daño hecho/recibido,
  Curación hecha/recibida), calculados filtrando `debugEvents` al vuelo (no hace falta tocar
  `parseCombat` — `debugEvents` ya registra cada línea con su `category`/`source`/`target`).
- Esas 4 columnas, más Derribos y Resurrecciones (que ya tenían tooltip), pasan a ser
  clicables: el clic abre el Modo debug con los filtros ya puestos a los eventos exactos detrás
  de ese número, y hace scroll hasta ahí.
- El Modo debug gana dos selects nuevos junto al de categoría — "Objetivo" y "Atribuido a" — y
  una casilla "+ incluir lanzamientos de 'Atribuido a'" que, marcada, añade también las líneas
  de categoría `lanzamiento` de ese jugador aunque el filtro de categoría esté en otra cosa. Al
  entrar desde Daño hecho/Curación hecha se marca automáticamente, para poder comparar de un
  vistazo el momento de cada golpe con el hechizo que lo causó.
- Mapeo dato → filtro: Daño hecho = categoría `daño` + Atribuido a = jugador. Daño recibido =
  categoría `daño` + Objetivo = jugador. Curación hecha = categoría `curación` + Atribuido a =
  jugador. Curación recibida = categoría `curación` + Objetivo = jugador. Derribos = categoría
  `derribo` + Objetivo = jugador. Resurrecciones = categoría `resurrección` + Atribuido a =
  jugador.

### ALCANCE

**Toca:** las 6 columnas agregadas de la tabla principal citadas arriba; el Modo debug (nuevos
selects, checkbox, refactor de la lógica de filtrado a una función pura `debugRowMatches`
testable con Node, sin cambiar el comportamiento de los filtros que ya existían — categoría,
búsqueda de texto, solo marcados).

**NO toca:** `parseCombat`/`resolveSource` (no se cambia nada de cómo se calculan ni atribuyen
los eventos, solo cómo se consultan después); el desglose por hechizo ni el de armadura/mods
(ya tienen tooltip por hover, pero el clic-a-debug para esas tablas queda fuera de esta feature
— posible ampliación futura, no se pierde nada al dejarlo así); historial/comparativa/modo
debug que aprende (sin cambios).

**Milestones:**
1. Lógica pura y testable: `eventsForMetric(debugEvents, category, {source, target})` y
   `debugRowMatches(ev, filtros)`; atributos `data-source`/`data-target` en las filas de debug;
   nuevos selects "Objetivo"/"Atribuido a" + checkbox en la toolbar, usando la función pura.
   Validar con Node que el comportamiento de los filtros ya existentes (categoría, búsqueda,
   solo marcados) no cambia.
2. Tooltips que faltaban en las 4 columnas agregadas + clic-a-debug en las 6 columnas (función
   `openDebugFiltered`, que abre la sección, pone los filtros y hace scroll) + estilo visual que
   distinga qué celdas son clicables.
3. Pulido: casilla "incluir lanzamientos" automática al entrar desde Daño/Curación hecha, cerrar
   esta sección de PROJECT.md.

## Limitaciones conocidas (algunas deliberadas, no por desconocimiento)

- **PA/PM/PW, resistencia y esquiva/placaje dados (en positivo) NO se trackean.** El texto del
  log es idéntico tanto si el aumento viene de un compañero como si es la propia pasiva del
  personaje activándose cada turno — no hay forma fiable de distinguirlo. Se investigó el
  proyecto open-source `Nexus-Hub/Wakfu-Companion` (MIT) como posible referencia — confirmado
  que tampoco resuelve esta ambigüedad ni el texto de crítico/esquiva en español; no había nada
  que reutilizar de ahí.
- **Robo de vida** ya está incluido dentro de "Curación hecha" — no se cuenta aparte.
- Si el log contiene varias sesiones con las mismas palabras clave, se usa siempre **la más
  reciente** (el último "fin" emparejado con el "inicio" más cercano antes de él); la app avisa
  si detecta más de una sesión.
- **Efectos de zona/aura con 2+ fuentes simultáneas del mismo nombre**: atribución
  estructuralmente imposible de resolver solo con el texto del chat (ver "Motor de atribución"
  arriba) — se deja sin atribuir y marcado como ambiguo en vez de adivinar.
- **Desglose por turno: descartado, no solo pendiente.** El log solo marca el fin de turno
  cuando el jugador pasa antes de tiempo y se le devuelven segundos ("X segundos devueltos
  para el siguiente turno"); cuando el turno acaba por el timer, no se registra nada. No hay
  forma fiable de saber dónde empieza/acaba cada turno con este log, así que no se puede
  trocear el combate turno a turno sin adivinar. En su lugar, en "Datos avanzados" hay un
  **ratio de eficiencia** por jugador que no depende de turnos: daño medio por hechizo con
  efecto, y daño hecho ÷ daño recibido.

## Estructura de archivos

```
wakfu-calc/
├── PROJECT.md                  este documento
└── wakfu_calculadora.html      la herramienta (un único archivo, sin dependencias externas)
```

## Estado actual

- [x] Parser validado contra múltiples combates reales distintos
- [x] Daño/curación hecho/recibido, % crítico (excluyendo hechizos sin efecto), derribos,
      resurrecciones
- [x] Armadura dada/recibida, golpes anticipados
- [x] Buffs/debuffs % por categoría con turnos de duración, toggle acumulado/pico alcanzado
- [x] Desglose de daño/curación por hechizo, con punto de color por elemento dominante
- [x] Detección y aviso de múltiples sesiones en el mismo archivo de log
- [x] Tablas ordenables (clic en cabecera) y filtrables por categoría/tipo
- [x] Tooltips con la línea de log original en cada fila (derribos, resurrecciones, mods, hechizos)
- [x] Motor de atribución de efectos de zona/aura (dueño de estado) con detección de ambigüedad
- [x] Modo debug: línea + razón de atribución por evento, flag de bug + nota, exportación a JSON
- [x] Historial persistente de combates (guardado con nombre, `window.storage` en el panel de
      chat, export/import `wakfu_historial.json` para uso como archivo suelto)
- [x] Comparativa entre 2 combates guardados: totales del grupo y desglose por jugador con
      delta y color según si la métrica mejora o empeora
- [x] Modo debug que aprende: correcciones de atribución (`{estado: jugador}`) confirmables desde
      la propia tabla de debug, guardadas en `window.storage`, aplicadas solas en próximos
      combates — alcance limitado a atribución, ver sección dedicada más arriba
- [ ] Selector de idioma de la interfaz (ES/EN) — pendiente si se necesita
- [ ] Revisar eventos marcados como bug en el modo debug cuando el usuario los exporte
- [x] Vista de tendencia (gráfico SVG, jugador/métrica seleccionables) cuando haya 3+
      combates guardados
- [x] IA que analiza el historial y da consejos concretos a partir de los deltas (botón
      "Analizar con IA", llamada directa a la API de Claude sin clave, caché en memoria)
- [x] Desglose por turno — descartado (el log no marca fin de turno de forma fiable, ver "Limitaciones conocidas")
- [x] Etiquetar combates (jefe/mazmorra/dificultad) al guardarlos, mostradas en historial y comparativa
- [x] Ratios de eficiencia: daño medio por hechizo con efecto, y daño hecho ÷ recibido (en "Datos avanzados")
- [x] Standalone fuera del panel de chat: BYOK (localStorage) para Análisis con IA, localStorage
      para correcciones aprendidas — ver "Principio de diseño" al principio del documento
- [x] Clasificación aliado/enemigo por combate (modal + `entry.roles`), Total del grupo dividido
      en Aliados/Enemigos en la comparativa, en vez de amalgamar ambos bandos

## Trabajo multi-sesión / multi-cuenta

Este proyecto se trabaja desde varias sesiones de Claude (distintas cuentas) en paralelo,
así que **antes de tocar código, mira `HANDOFF.md`** en la raíz del repo — dice si hay
trabajo a medias de una sesión anterior que se quedó sin tokens, y evita reimplementar o
pisar algo ya empezado. Al dejar algo a medias (por ejemplo por falta de tokens), se rellena
esa plantilla y se hace commit/push antes de cerrar, en vez de esperar a tener la feature
100% terminada. Detalle completo del protocolo dentro del propio `HANDOFF.md`.

## Créditos

Idea original: [MethodWakfu Companion](https://companion.methodwakfu.com/) (jpark / Zephyrs
(Nox)). WAKFU es una marca de Ankama Studio; esta herramienta no tiene relación oficial con
Ankama.
