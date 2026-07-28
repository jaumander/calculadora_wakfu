# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: Nada en curso

Última feature completada: copias de seguridad y memoria de roles. Documentado en PROJECT.md,
sección "Copias de seguridad y memoria de roles — ALCANCE". Los 4 milestones commiteados y
pusheados: 1/4 botón "Descargar copia de este log" (descarga manual, con clic explícito, del
texto tal cual se cargó, con nombre con timestamp), 2/4 cada entrada nueva del historial guarda
las líneas de log originales del rango (`entry.rawLines`; entradas antiguas simplemente no lo
tienen, sin migración necesaria), 3/4 el modal de clasificación aliado/enemigo preselecciona por
nombre según la última clasificación recordada de cualquier combate anterior (siempre pidiendo
confirmación, nunca automático ni basado en estadísticas del combate), 4/4 nota de alcance
"100% español" en PROJECT.md, descartando el selector ES/EN que estaba pendiente.
No tocó `parseCombat` ni `resolveSource`. No se intentó detectar/gestionar la rotación real de
`wakfu_chat.log` (estructuralmente imposible desde el navegador, ver el "Motivo" de esa sección).
Validado con Node: lógica de preselección + actualización de memoria de roles con 3 nombres
(2 recordados, 1 nuevo clasificado a mano), además de los `node --check` de sintaxis en cada
milestone.

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
