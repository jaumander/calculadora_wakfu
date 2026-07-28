# HANDOFF — estado en curso (no confundir con PROJECT.md)

> Si este archivo está vacío/pone "Nada en curso", el proyecto está en un punto limpio:
> parte de PROJECT.md con normalidad. Si NO está vacío, hay trabajo a medias — lee esto
> ANTES de tocar código, para no reimplementar o pisar algo ya empezado.

## Estado: Nada en curso

Última feature completada: filtro por etiquetas en Analisis con IA y Vista de tendencia
(filterHistoryByTags reutilizada en ambas), y selector de sesion multiple en la carga de log
(detectSessions). Documentado en PROJECT.md, secciones "Filtro por etiquetas en Analisis con
IA y Vista de tendencia" y "Selector de sesion cuando el log tiene varias con las mismas
palabras clave". 3 milestones commiteados y pusheados. No toco parseCombat ni resolveSource
(no hacia falta log real, ver alcance del commit inicial). Cada milestone se probo con Node:
detectSessions con 5 escenarios (1 sesion, 3 seguidas, end huerfano, starts duplicados, sin
keywords), filterHistoryByTags/buildAIPayload con filtro/fallback/sin-etiquetas, y
uniqueTagCombos/drawTrendChart con datos simulados. Nada roto.

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
