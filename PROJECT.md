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

## Selector de sesión cuando el log tiene varias con las mismas palabras clave

Antes, si un log tenía varias sesiones inicio/fin con las mismas palabras clave (típico de una
noche con varios intentos al mismo jefe usando el mismo par de palabras cada vez), la app
siempre calculaba la más reciente y solo avisaba en un texto de que había más — sin forma de
verlas sin cambiar de palabras clave a mano.

Ahora `detectSessions` empareja cada "fin" con el "inicio" más cercano anterior a él que no se
haya usado ya en una sesión previa, devolviendo TODAS las sesiones del archivo en orden
cronológico (no solo la última). El comportamiento por defecto no cambia — al pulsar "Calcular"
se sigue mostrando automáticamente la sesión más reciente — pero si se detecta más de una,
aparece debajo un selector con cada sesión (número de líneas + primera línea de log) y un botón
"Calcular esta" para recalcular con cualquiera de las anteriores sin tocar las palabras clave.

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

## Principio general: datos indirectos con lanzador desplazado (Pulgas, El Gatallón, futuros casos)

**El problema, en general:** el motor asume por defecto que un evento de daño/curación directo
(`pdvRe`) pertenece a quien tiene el turno activo ("lanzador activo") en ese instante — heurística
documentada como fiable porque "es el resultado directo de una acción, no un efecto retardado de
zona". Pulgas y El Gatallón demuestran que esa premisa **no siempre es cierta**: son estados/auras
que un jugador aplica en un momento dado, y que luego generan daño/curación varios turnos después
—cuando el turno activo ya es el de otro jugador, o incluso el del propio objetivo—, sin que haya
ningún "lanza el hechizo" inmediatamente antes que delate quién es el responsable real.

**Cómo se reconoce este patrón en el log (señales, no reglas fijas):**
- El evento (`+N PdV` o `-N PdV`) no va precedido de un `lanza el hechizo` del jugador correcto.
- Puede llevar el nombre del efecto explícito entre paréntesis (caso fácil, como El Gatallón: se
  resuelve consultando `stateOwners`, sin hacer falta más), o puede no llevarlo (caso difícil,
  como Pulgas: hace falta una señal indirecta, en ese caso la pasiva "Racha Sanadora", para
  confirmar el origen sin adivinar).
- El "objetivo" del evento suele ser quien lleva el estado/debuff que el verdadero responsable
  aplicó antes, no el responsable en sí.

**Cómo se investiga un caso nuevo (nunca al revés — nunca implementar primero y verificar
después):**
1. Pedir o usar un log real; nunca asumir el patrón por analogía con Pulgas/El Gatallón.
2. Contar CUÁNTOS eventos entran en el patrón propuesto y A QUIÉN pertenecen actualmente (grep al
   log, o Node contra `parseCombat` real). Si la regla propuesta "roba" datos de jugadores que no
   tienen nada que ver, hay que estrechar la condición (fue exactamente lo que pasó con la
   petición original de Pulgas: "toda cura (luz)/(fuego)" robaba a 7 jugadores; la regla real
   quedó mucho más estrecha).
3. Buscar si el estado en cuestión ya se registra correctamente con el mecanismo estándar de
   "Niv. 1" (`stateLevelRe`) — si el formato real del log no encaja (como pasaba con "Pulgas
   (+31 niv.)"), ese es motivo para arreglar el registro en vez de inventar una señal externa.
4. Validar con Node: conservación exacta (lo que pierde un jugador es exactamente lo que gana
   otro, nunca se debe crear ni destruir daño/curación), y comprobar explícitamente que jugadores
   con datos parecidos por coincidencia NO se ven afectados (falsos positivos).
5. Si el log real no tiene ningún ejemplo donde la regla dispare de verdad (como pasó con El
   Gatallón, que solo tenía 1 ejemplo y ya estaba bien atribuido por casualidad), decirlo tal
   cual — no dar la regla por probada solo porque no rompió nada. Un caso sintético construido a
   mano puede demostrar que la lógica funciona, pero no sustituye la validación contra datos
   reales cuando existan.

**Casos conocidos hasta ahora** (detalle completo en sus propias secciones más abajo):
- **Pulgas → curas a Laik:** curación etiquetada EXACTAMENTE `(fuego)` o `(luz) (Fuego)`, sin
  nombre de efecto, verificada con la señal de "Racha Sanadora".
- **El Gatallón → cura a su dueño:** curación que nombra el efecto entre paréntesis; se resuelve
  consultando el dueño ya registrado en `stateOwners` (regla general, no hardcodeada al nombre).

**Para próximas sesiones:** si al usuario se le ocurre un caso nuevo de este tipo, seguir el
proceso de arriba antes de tocar `parseCombat`. Puede tratarse de curación, daño (ej. un reflejo
o contraataque que se dispara en el turno de otro), o cualquier otro campo — el principio es el
mismo independientemente del tipo de dato.

## Excepción de atribución: curas de Pulgas (luz)/(fuego) a Laik

**Petición original:** "toda curación etiquetada (luz)/(fuego) se atribuye a Laik si está en el
combate". Investigación contra un log real descartó esa regla tal cual: de las curaciones con
esas etiquetas, la mayoría eran de Hidori, Ledgem, Androido, Hikku, Ofrizz — curaciones legítimas
de sus propios hechizos (Cunejo, Llama Purificadora). Implementarla literalmente les habría
robado curaciones a esos jugadores.

**Aclaración del usuario que definió la regla real:** el "robo" es correcto solo para un
subconjunto — una curación indirecta que ocurre al terminar el turno de un jugador que "lleva
pulgas" (debuff que Laik aplica con hechizos como "Ruleta de Dados"/"Laceraciones"), esa
curación puntual sí debería atribuirse a Laik aunque el objetivo (y el "lanzador activo" del
motor) sea otro. El usuario también aclaró que "Racha Sanadora" (una pasiva de Laik que sube %
con cualquier curación suya) NO es la regla en sí, solo una señal de que una curación es suya —
la regla real es la etiqueta (luz)/(fuego) de las pulgas.

**Regla implementada (verificada línea por línea contra `wakfu_chat.log` real):** una curación
etiquetada EXACTAMENTE con `(fuego)` o `(luz) (Fuego)` —sin ningún nombre de efecto detrás— se
atribuye a Laik en vez de al lanzador activo. La señal de "Racha Sanadora" de Laik se usa
internamente solo como comprobación (no como la condición conceptual) para no robarle a Hidori
su "Llama Purificadora" ni a su invocación Cunejo su "Murmullos Curativos", que por coincidencia
producen curaciones con esa misma etiqueta de elemento sin nombre de efecto detrás, pero no
tienen nada que ver con Pulgas.

**Validado contra el log real con Node** (`parseCombat` aislado, antes/después):
- 11 eventos de curación se reatribuyen de Hidori/Hikku/Ledgem/Ofrizz/Androido/Súper Cunejo a
  Laik.
- Conservación exacta: la suma de lo que pierden esos 6 jugadores (5.881 PdV) es exactamente lo
  que gana Laik (+5.881 PdV) — nada se crea ni se pierde, solo se reasigna.
- Cero falsos positivos: se comprobó explícitamente que las curaciones de Hidori (Llama
  Purificadora) y Cunejo (Murmullos Curativos) con esa misma etiqueta de elemento siguen
  atribuidas a ellos, no a Laik.
- Las curaciones con nombre de efecto explícito (ej. "Marca del Reintegro", de Hikku, no
  relacionado con Pulgas) no se tocan — siguen con la heurística de lanzador activo de siempre.
- Un caso relacionado pero fuera de esta regla, dejado tal cual a propósito: el aura "El
  Gatallón" (que Laik también aplica) cura con etiqueta `(agua)`, no `(fuego)`/`(luz)(Fuego)` —
  no entra en esta regla porque el usuario la definió específicamente para las pulgas, no para
  esa aura. Si hiciera falta corregirla también, sería una regla aparte.

**Dónde vive en el código:** `parseCombat`, función `healBelongsToLaikPulgas(rest, idx)` (justo
antes del bucle principal) + su uso en la rama `pdvRe` de curación (`sign === '+'`). No toca
`resolveSource` ni el resto de estados/debuffs — es una excepción puntual, autocontenida, sobre
la heurística de lanzador activo existente. El desglose "por hechizo" del jugador que aparecía
como lanzador activo NO cuenta estas curaciones reatribuidas (para no duplicar el número en dos
sitios a la vez).

## Extensión: curas de auras con nombre de efecto explícito (ej. "El Gatallón") a su dueño real

**Petición del usuario:** las curas de "El Gatallón" (otra aura que Laik aplica) también son
indirectas y deben atribuirse a Laik, igual que Pulgas.

**Por qué es un caso distinto y más simple que Pulgas:** a diferencia de Pulgas (formato de
nivel "+N niv." que no encaja con el registro estándar de estados, y sin nombre de efecto en la
propia línea de curación), "El Gatallón" SÍ se registra correctamente con el mecanismo estándar
de "Niv. 1" ya existente, y la curación SÍ lleva el nombre del efecto explícito en su propia
línea: `Laik: +354 PdV (fuego) (El Gatallón)`. No hace falta ninguna señal externa tipo "Racha
Sanadora" — basta con mirar si alguno de los nombres entre paréntesis de la curación coincide con
un estado ya registrado en `stateOwners` con un único dueño (mismo principio que `resolveSource`
ya usa para buffs/debuffs, pero mirando TODOS los paréntesis de la línea, no solo el primero, que
en una curación suele ser el elemento y no el nombre del efecto).

**Regla implementada, general (no hardcodeada a "El Gatallón" ni a "Laik"):** una curación cuya
línea nombra entre paréntesis un estado ya registrado con un único dueño se atribuye a ese dueño
en vez de al lanzador activo. Esto cubre "El Gatallón" sin necesidad de codificar su nombre a
mano, y cubriría automáticamente cualquier otra aura similar que aparezca en el futuro, siempre
que ya esté registrada por el mecanismo estándar de "Niv. 1" y tenga un único dueño.

**Validación contra `wakfu_chat.log` real — resultado honesto:** en este log solo hay 1 ejemplo
de curación de "El Gatallón", y en ese caso concreto el lanzador activo ya era Laik por
coincidencia (se la aplicó a sí mismo y se curó en su propio turno), así que la regla no tiene
ningún caso real en este log donde demuestre estar corrigiendo algo — pero tampoco rompe nada
(comprobación de regresión: los 11 eventos de Pulgas y el total de curación de Laik no cambian).
Para probar que la lógica sí dispara cuando hace falta, se validó con un caso SINTÉTICO (no del
log real, construido a mano): Laik lanza "El Gatallón", luego otro jugador pasa a ser lanzador
activo, y llega una curación indirecta con "(El Gatallón)" en el texto sobre un tercero — se
atribuye correctamente a Laik, no al lanzador activo del momento.

**Dónde vive en el código:** `parseCombat`, función `healNamedEffectOwner(rest)`, junto a
`healBelongsToLaikPulgas`. Mismo flag `healSourceOverride` que Pulgas para excluir estas
curaciones del desglose "por hechizo" del lanzador activo desplazado.

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

## Filtro por etiquetas en Análisis con IA y Vista de tendencia

Las etiquetas (jefe/mazmorra/dificultad) existían desde que se guarda un combate, pero el
Análisis con IA y la Vista de tendencia seguían comparando contra TODO el historial sin
filtrar — comparar un intento a un jefe difícil contra la media de mazmorras fáciles de relleno
daba consejos y gráficos menos útiles de lo que las etiquetas prometían. `filterHistoryByTags`
(pura, reutilizada en ambas features) resuelve esto:

- **Análisis con IA**: lee los campos Jefe/Mazmorra/Dificultad del propio formulario de
  "Guardar en historial" tal cual estén rellenos en ese momento (no hace falta haber guardado
  el combate todavía) y filtra la media histórica por ellos. El payload que recibe la IA incluye
  `comparacion.modo`: `"filtrado"` si hubo coincidencias (la IA lo menciona explícitamente,
  tipo "comparado con tus otros intentos a este jefe"), `"fallback_sin_coincidencias"` si se
  pidió el filtro pero no había combates guardados con esas etiquetas (se avisa y se usa el
  historial general en su lugar, nunca un resultado vacío), o `"sin_etiquetas"` si no se rellenó
  ningún campo (comportamiento de siempre). Una línea de contexto sobre los consejos deja claro
  cuál de los tres casos se aplicó. La caché en memoria de respuestas incluye las etiquetas en su
  clave, para no servir un consejo obsoleto si se cambian las etiquetas y se vuelve a analizar.
- **Vista de tendencia**: nuevo tercer selector con las combinaciones de etiquetas detectadas en
  el historial guardado (`uniqueTagCombos`) + "Todos los combates" por defecto. Combates
  guardados sin etiquetas simplemente no aparecen en ninguna combinación del selector y quedan
  fuera si se filtra por una etiqueta concreta — no rompen nada, solo no cuentan para ese filtro.

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

### Click-to-filter: de un dato concreto de cualquier tabla a sus líneas exactas en modo debug

Con combates largos, buscar a mano en el modo debug qué líneas forman un número concreto (ej.
"daño recibido por Ofrizz") era tedioso incluso con el filtro por categoría. Ahora casi todas las
celdas de datos de las demás tablas (tabla principal, avanzada, mods, desglose por hechizo) son
clicables: al pulsarlas, se abre el modo debug (si estaba cerrado) y se filtra **solo** a los
eventos que forman ese dato exacto, con un chip "Filtrando: …" y botón para quitar el filtro.

- No toca `parseCombat`: reutiliza los campos que ya guarda cada evento (categoría/objetivo/
  atribuido/línea) y, para hechizos concretos, las líneas exactas que ya guarda cada
  `p.spells[nombre].lines` — por eso "Daño" de un hechizo con robo de vida excluye correctamente
  las líneas de curación de ese mismo hechizo (y viceversa).
- El filtro fijado (`pinnedFilter`) tiene prioridad sobre el desplegable de categoría y la
  búsqueda de texto, que se deshabilitan mientras esté activo; se combina con el filtro de "solo
  marcados 🐞" si también está encendido.
- Se probó con `jsdom` simulando clics reales (no solo revisando el código): daño/curación hecho
  y recibido, armadura dada, un hechizo concreto con robo de vida, una fila de mods, y que un
  recálculo con un combate distinto no deja "colgado" el filtro del combate anterior.
- **Daño hecho / Curación hecha** llevan además una casilla "incluir lanzamientos de
  &lt;jugador&gt;" en el propio chip del filtro: al marcarla, se añaden también las líneas de
  categoría `lanzamiento` de ese jugador (aunque no formen parte del predicado exacto del dato),
  para poder cotejar de un vistazo si el hechizo lanzado justo antes de cada golpe encaja con a
  quién se le atribuyó ese daño/curación — pensado para el caso "¿asignó bien el lanzador de
  este daño?". Probado con `jsdom`: marcar la casilla añade las filas de lanzamiento sin tocar
  las demás, desmarcarla las vuelve a ocultar, y la casilla no aparece en absoluto en métricas
  sin este sentido (daño/curación *recibida*, derribos, armadura, etc.).

## Casos conocidos (regresión del motor de atribución) — ALCANCE

Objetivo: que los eventos que ya se marcan con 🐞 en el modo debug dejen de perderse al cerrar la
pestaña, y sirvan de red de seguridad cuando se toque `resolveSource`/`parseCombat` en el futuro —
sin añadir superficie nueva de cara al día a día de jugar (esto es fontanería interna, no una
función que tú vayáis a usar jugando).

**Qué toca:**
- El flujo de exportar 🐞 (además de descargar el JSON como ya hace, sin quitarlo).
- Guardado persistente vía `window.storage` (mismo patrón que el historial: `hasStorage()`,
  cachear, fallback sin storage).
- Un botón nuevo, discreto, dentro del propio modo debug: "Validar casos conocidos".

**Qué NO toca (para no crecer el proyecto de más):**
- No añade ninguna tarjeta/sección nueva en la pantalla principal.
- No añade UI para editar o borrar casos uno a uno a mano (si hace falta más adelante, se añade
  entonces).
- No modifica `parseCombat` ni `resolveSource` — solo lee `debugEvents` ya calculados.
- No inventa un campo de "respuesta correcta" estructurado: la nota que ya se escribe al marcar
  con 🐞 sigue siendo texto libre; "validar" compara la atribución de entonces contra la de ahora
  para la misma línea exacta de log, y deja que la persona juzgue si el cambio es una mejora.

**Milestones:**
1. Guardado persistente de los casos marcados (sin UI nueva visible aparte del propio flujo de
   exportar, que ahora también guarda).
2. Botón "Validar casos conocidos": para el combate cargado ahora mismo, busca qué casos
   guardados coinciden por línea exacta y avisa si la atribución cambió desde que se marcaron.
3. Exportar/importar casos conocidos (igual que ya existe para el historial, para uso standalone
   sin `window.storage`) y cerrar la documentación.

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
- [x] Click-to-filter: clic en un dato concreto de cualquier tabla abre y filtra el modo debug
      a solo los eventos que forman ese número exacto (ver sección dedicada más arriba)
- [x] Historial persistente de combates (guardado con nombre, `window.storage` en el panel de
      chat, export/import `wakfu_historial.json` para uso como archivo suelto)
- [x] Comparativa entre 2 combates guardados: totales del grupo y desglose por jugador con
      delta y color según si la métrica mejora o empeora
- [x] Modo debug que aprende: correcciones de atribución (`{estado: jugador}`) confirmables desde
      la propia tabla de debug, guardadas en `window.storage`, aplicadas solas en próximos
      combates — alcance limitado a atribución, ver sección dedicada más arriba
- [x] Idioma de la interfaz: 100% español, por decisión de alcance (no un descuido). Se descartó
      un selector ES/EN — no se quiere mantener ni testear varios idiomas. Si algún día hace falta
      compartir la herramienta fuera de un grupo hispanohablante, se replantea entonces desde cero
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
- [x] Mejoras QoL (carga de log, tablas, desplegables) — ver sección dedicada más arriba: recordar
      palabras clave, copiar líneas de log, buscador de jugador, recordar desplegables abiertos en
      la sesión, aviso si el archivo no parece un log de Wakfu, copiar tabla principal como texto
- [x] Casos conocidos (regresión del motor de atribución) — persistencia de eventos marcados con
      🐞, botón "Validar casos conocidos" en el modo debug, export/import standalone. Ver sección
      dedicada más arriba. Fontanería interna: sin tarjeta nueva en pantalla principal a propósito
- [x] Copias de seguridad y memoria de roles — botón "Descargar copia de este log", líneas de log
      originales guardadas en cada entrada nueva del historial (`entry.rawLines`), y clasificación
      aliado/enemigo recordada por nombre entre sesiones (preselección en el modal, nunca
      automática). Ver sección dedicada más arriba. Idioma: descartado el selector ES/EN, 100%
      español por decisión de alcance
- [x] Clasificación aliado/enemigo por combate (modal + `entry.roles`), Total del grupo dividido
      en Aliados/Enemigos en la comparativa, en vez de amalgamar ambos bandos
- [x] Filtro por etiquetas (jefe/mazmorra/dificultad) en Análisis con IA y Vista de tendencia —
      compara solo contra combates similares en vez de todo el historial mezclado
- [x] Selector de sesión cuando el log tiene varias con las mismas palabras clave, en vez de
      calcular siempre la más reciente en silencio

## Mejoras de calidad de vida (QoL) — carga de log, tablas, desplegables

**Alcance — qué toca:**
- Zona de carga de archivo: recordar las palabras clave de inicio/fin entre sesiones, y avisar
  de forma específica si el archivo cargado no tiene pinta de log de Wakfu (en vez del mensaje
  genérico de "no se encontraron las palabras clave").
- Tablas de resultados (principal, avanzada, hechizos, mods): botón de copiar en las celdas que
  ya muestran línea(s) de log en el tooltip; buscador de jugador que filtra filas/bloques en esas
  mismas tablas; botón "Copiar tabla como texto" en la tabla principal.
- Desplegables del panel de resultados (avanzado/mods/hechizos/debug): recordar cuáles estaban
  abiertos al recalcular otro combate, solo durante la sesión actual de la pestaña (variable en
  memoria, no persistente entre recargas — así el riesgo de esta parte queda igual de bajo que
  antes).

**Qué NO toca (explícito para no poder romperlo por accidente):**
- `parseCombat`, `resolveSource` ni ningún regex de parseo de eventos ya existente. Ninguna de
  estas mejoras cambia cómo se interpretan las líneas del log, solo cómo se usan/muestran los
  datos ya calculados.
- Historial, comparativa, vista de tendencia, Análisis con IA, clasificación aliado/enemigo,
  modo debug que aprende (`stateResolutions`) — nada de esa lógica se modifica.

**Milestones:**
1. Recordar palabras clave de inicio/fin (storage igual que `stateResolutions`: `window.storage`
   dentro del panel, `localStorage` fuera).
2. Botón copiar en celdas con tooltip de línea(s) de log (derribos, resurrecciones,
   buffs/debuffs, hechizos).
3. Buscador rápido de jugador (filtra tabla principal, avanzada, mods y bloques de hechizos).
4. Recordar qué desplegables estaban abiertos entre cálculos, solo en memoria de la sesión.
5. Aviso específico cuando el archivo cargado no tiene formato de log de Wakfu reconocible.
6. Botón "Copiar tabla como texto" en la tabla principal.

## Copias de seguridad y memoria de roles — ALCANCE

**Motivo:** `wakfu_chat.log` no tiene fecha (solo hora `HH:MM:SS,mmm`) y no hay documentación
oficial de Ankama sobre cuándo el cliente lo limpia o rota — se comprobó en un log real subido
por el usuario que dentro de una sesión continua no se corta (solo hay un salto de medianoche,
`23:59` → `00:00`, sin pérdida de líneas), pero no se puede descartar que se sobrescriba al
reiniciar el cliente. Como no se puede verificar la regla exacta, la mitigación no depende de
adivinarla: la propia app guarda su propia copia en el momento en que ya tiene los datos en la
mano, en vez de vigilar el archivo real (que además ni siquiera puede — el navegador solo lee el
archivo una vez, al soltarlo, no mantiene acceso continuo a él).

**Qué toca:**
- Zona de carga de archivo: botón para descargar una copia local del log tal cual se cargó
  (backup manual con clic explícito, nunca descarga automática sin que el usuario la pida).
- Guardado en historial: cada combate guardado incluye ahora las líneas de log originales del
  rango usado (`entry.rawLines`), para poder reabrir el modo debug / validar casos conocidos de
  un combate guardado sin depender de que `wakfu_chat.log` siga teniendo esas líneas. Entradas
  antiguas del historial sin este campo se siguen mostrando igual, solo sin esa opción extra.
- Modal de clasificación aliado/enemigo (`openRoleModal`): recuerda por nombre la última
  clasificación usada (`window.storage`/`localStorage`, mismo patrón que el resto) y la
  preselecciona la próxima vez que aparezca ese nombre — pero SIEMPRE pidiendo confirmación,
  nunca se salta el modal ni se infiere el rol a partir de las estadísticas del combate (un jefe
  puede pegar y curar igual que un aliado, así que no hay forma fiable de inferirlo solo).

**Qué NO toca:**
- `parseCombat` / `resolveSource`.
- No intenta detectar ni gestionar la rotación real de `wakfu_chat.log` — es estructuralmente
  imposible de vigilar desde el navegador (ver "Motivo" arriba); la copia de seguridad es
  responsabilidad de la persona, la app solo se lo pone fácil con un botón.
- No infiere el rol aliado/enemigo por daño/curación ni por ningún otro dato del combate — solo
  recuerda clasificaciones manuales anteriores por nombre exacto.

**Milestones:**
1. Botón "Descargar copia de este log" en la zona de carga de archivo.
2. Guardar las líneas de log originales del rango en cada entrada nueva del historial.
3. Recordar la clasificación aliado/enemigo por nombre y preseleccionarla en el modal (siempre
   pidiendo confirmación).
4. Nota de alcance "solo español" en este documento, descartando el selector de idioma ES/EN que
   estaba en la lista de pendientes.

## Sesiones de mazmorra — ALCANCE

**Motivo:** ahora mismo cada combate se guarda y analiza suelto. En la práctica, una mazmorra
encadena varios combates seguidos (o varios intentos al mismo jefe), y no hay forma de ver el
conjunto — solo combate a combate o comparando de dos en dos. El historial, las etiquetas, la
comparativa y los roles aliado/enemigo ya existen; esta feature es una capa de agregación encima,
sin tocar nada de cómo se calcula un combate individual.

**Qué toca:**
- Historial: las casillas de selección dejan de limitarse a 2. Con exactamente 2 marcadas se
  sigue mostrando la comparativa de siempre (sin cambios ahí). Con 2 o más aparece además una
  barra para agruparlas en una sesión nueva (campo de nombre + botón "Crear sesión").
- Persistencia nueva `loadSessions()`/`saveSessionsList()`, mismo patrón que el historial:
  `window.storage` en el panel, export/import JSON fuera de él. Una sesión es solo
  `{ id, name, date, entryIds: [...] }` — una lista de IDs que apuntan a combates ya existentes
  del historial, nunca una copia de sus datos.
- Tarjeta nueva "Sesiones de mazmorra": por cada sesión guardada, total agregado (aliados y
  enemigos por separado, reutilizando `combatTotals`/`splitByRole`/`subsetPlayers` ya existentes),
  qué combate de la sesión fue el mejor/peor (criterio simple y explícito: menos derribos totales
  de aliados, empate por daño hecho de aliados — nada de puntuaciones compuestas inventadas),
  lista de combates que la componen, y borrar sesión (nunca borra los combates que contiene).

**Qué NO toca:**
- `parseCombat` / `resolveSource` / ninguna lógica de cálculo de un combate individual.
- No copia datos de los combates dentro de la sesión — solo guarda sus IDs. Si se borra un
  combate del historial, desaparece con gracia de cualquier sesión que lo tuviera (sin reventar,
  sin dejar entradas fantasma en la UI).
- La vista de tendencia actual (por jugador, entre combates sueltos) no se toca ni se le añade
  agrupación por sesión en esta ronda — sigue funcionando exactamente igual que ahora. Se puede
  plantear como ronda futura, pero queda fuera de este alcance.
- El criterio de "mejor/peor combate" es deliberadamente simple (ver arriba) — no se inventa una
  métrica ponderada ni un sistema de puntuación.

**Milestones:**
1. `loadSessions()`/`saveSessionsList()` + quitar el límite de 2 en las casillas del historial +
   botón "Agrupar seleccionados en una sesión" (con nombre) cuando hay 2 o más marcadas.
2. Tarjeta "Sesiones de mazmorra": listar sesiones, total agregado por sesión (aliados/enemigos),
   combates que la componen, borrar sesión.
3. Detectar el mejor/peor combate de la sesión con el criterio simple, y manejar con gracia los
   IDs de combates que ya no existen en el historial.
4. Export/import de sesiones (mismo patrón que historial/casos conocidos) para uso standalone.

## Trabajo multi-sesión / multi-cuenta

Este proyecto se trabaja desde varias sesiones de Claude (distintas cuentas) en paralelo,
así que **antes de tocar código, mira `HANDOFF.md`** en la raíz del repo — dice si hay
trabajo a medias de una sesión anterior que se quedó sin tokens, y evita reimplementar o
pisar algo ya empezado. Al dejar algo a medias (por ejemplo por falta de tokens), se rellena
esa plantilla y se hace commit/push antes de cerrar, en vez de esperar a tener la feature
100% terminada. Detalle completo del protocolo dentro del propio `HANDOFF.md`.

## Higiene de git (evitar que se cuelen archivos que no tocan)

El repo tuvo un incidente donde `node_modules/` (carpeta de dependencias de Node, generada
automáticamente al usar herramientas como jsdom para validar) se subió por error a GitHub,
inflando el repo a 27MB/1836 archivos de más y ralentizando cualquier sesión que lo clonara.

Para que no se repita:
- Existe un `.gitignore` con `node_modules/` — no debe borrarse.
- Antes de cualquier `git add`, revisar `git status` y añadir archivos de forma explícita
  (ej. `git add wakfu_calculadora_1.html PROJECT.md`) en vez de `git add .` a ciegas.
- Si una sesión necesita instalar dependencias de Node para validar algo, recordar que esa
  carpeta es temporal y no debe commitearse.

## Roadmap — mejoras inspiradas en herramientas similares (WarcraftLogs/FFLogs, MethodWakfu)

Decidido con el usuario el 2026-07-31. Uso interno (no hay cuentas ni comparación contra otros
jugadores — esa idea se descartó explícitamente por no encajar con el alcance del proyecto).

**Corrección sobre la versión original de este roadmap:** se escribió sin haber leído primero
todo lo ya implementado por sesiones anteriores (ver "Estado actual" más abajo), así que
duplicaba trabajo ya hecho. Corregido:

- ~~% de uptime real de buffs/debuffs por turno~~ y ~~gráfico de daño/curación por turno~~:
  **ya investigado y descartado como inviable** — ver "Desglose por turno: descartado, no solo
  pendiente" en "Limitaciones conocidas". El log no marca de forma fiable el fin de cada turno
  (solo cuando se pasa antes de tiempo), así que no hay forma honesta de trocear el combate por
  turno. La alternativa ya implementada son los **ratios de eficiencia** (daño medio por hechizo
  con efecto, daño hecho ÷ recibido) en "Datos avanzados".
- ~~Histórico de combates en el navegador~~: **ya implementado**, y de forma más completa de lo
  planteado originalmente — historial persistente con nombre/etiquetas, comparativa entre 2
  combates con deltas, gráfico de tendencia con 3+ combates, y hasta análisis con IA de esos
  deltas. Ver "Estado actual".

Queda pendiente de esta lista original solo:

1. **Desglose de daño/curación hecha por objetivo** (a qué mob/jugador enemigo golpeó cada
   jugador, y cuánto). Usa datos que el parser ya extrae (el `target` de cada línea de PdV),
   solo cambia cómo se agrupan. **En curso** — ver HANDOFF.md.

**Descartado explícitamente:** ranking o comparación contra otros jugadores (percentiles tipo
WarcraftLogs) — requeriría servidor/cuentas y el usuario quiere la app de uso interno, sin subir
datos a ningún sitio.

## Créditos

Idea original: [MethodWakfu Companion](https://companion.methodwakfu.com/) (jpark / Zephyrs
(Nox)). WAKFU es una marca de Ankama Studio; esta herramienta no tiene relación oficial con
Ankama.
