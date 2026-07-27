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
- [ ] Selector de idioma de la interfaz (ES/EN) — pendiente si se necesita
- [ ] Revisar eventos marcados como bug en el modo debug cuando el usuario los exporte

## Créditos

Idea original: [MethodWakfu Companion](https://companion.methodwakfu.com/) (jpark / Zephyrs
(Nox)). WAKFU es una marca de Ankama Studio; esta herramienta no tiene relación oficial con
Ankama.
