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
- [ ] Vista de tendencia (gráfico) cuando haya 3+ combates guardados — pendiente
- [ ] IA que analice el historial y dé consejos concretos a partir de los deltas — pendiente
- [ ] Desglose por turno — pendiente
- [x] Etiquetar combates (jefe/mazmorra/dificultad) al guardarlos, mostradas en historial y comparativa
- [ ] Ratios de eficiencia (daño por turno, daño hecho vs. recibido) — pendiente

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
