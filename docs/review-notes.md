# Notas de revisión — documentos y planes de nutcracker

> Revisión de los documentos del proyecto (`plan.md`, `ROADMAP.md`, `README.md`,
> `docs/*.md`) contrastados contra el código real.
> Última revisión: 2026-07-27.

## Alcance

Se revisaron los siguientes documentos:

- `plan.md` — 1.403 líneas
- `ROADMAP.md` — 34 líneas
- `README.md` — 984 líneas
- `docs/aipwn-flow.md` — 478 líneas
- `docs/static-analysis-flow.md` — 319 líneas
- `docs/owasp-mas-coverage.md` — 116 líneas

Y se verificaron inconsistencias contra el código (`orchestrator.py`,
`cli/queue_cmd.py`, `cli/batch.py`, `store/schema.sql`, `nutcracker_core/__init__.py`,
21 archivos de tests en `tests/`).

Los hallazgos se agrupan en:

- **(A) Inconsistencias plan↔código** — objetivas y verificadas.
- **(B) Mejoras de estructura y calidad documental** — subjetivas y accionables.

---

## A) Inconsistencias plan↔código (objetivas)

### A1. `queue rm` no existe, pero está documentado

- `plan.md` (Fase 1.3) y `README.md` describen `nutcracker queue add|ls|rm`.
- Verificado en `cli/queue_cmd.py`: solo existen `add` y `ls` (líneas 43 y 119).
  No hay `rm`/`remove`.
- **Acción:** implementar `queue rm` (borrado de job encolado) o corregir la
  documentación para eliminar la mención.

### A2. `batch --due` no existe, pero está descrito en el plan

- `plan.md` (Fase 1.3) lo especifica como "modo one-shot para cron/systemd
  (encola solo lo vencido y sale)".
- Verificado en `cli/batch.py`: no hay flag `--due` ni lógica de vencidos.
- **Acción:** implementarlo o marcarlo explícitamente como "no implementado /
  diferido" en el plan.

### A3. `schema.sql` incompleto respecto al schema real en runtime

- `plan.md` (Fase 0.1) presenta `schema.sql`/`db.py` como fuente del esquema.
  Pero la tabla `queue_jobs` (añadida en Fase 1) **no está en `schema.sql`** —
  solo existe vía migración programática en `db.py`. Un `schema.sql` que alguien
  lea pensando que es la verdad del esquema queda desactualizado.
- **Acción:** regenerar `schema.sql` desde el código de migración actual, o
  documentar claramente que `schema.sql` es solo el base v1 y las migraciones
  viven en `db.py`.

### A4. `ROADMAP.md` contradice a `plan.md` sobre la verificación de WebUSB

- `ROADMAP.md` (item WebUSB): *"Not verified: an actual end-to-end USB
  connection against a physical device"*.
- `plan.md` Fase 4: *"✅ Confirmado funcionando por el usuario, con video real"*
  y *"Fase 4 queda validada de punta a punta contra hardware físico real"*.
- **Acción:** sincronizar ambos documentos (probablemente actualizar el ROADMAP
  con el resultado de la validación en vivo del 2026-07-27).

### A5. Métrica "cobertura OWASP" usada con dos significados distintos

- `docs/owasp-mas-coverage.md` y `README.md`: **18/24** controles con al menos
  un check (correcto y honesto).
- `plan.md` (Fase 2 — estado): *"coverage pasó de 15/24 a 24/24 controles
  reportados"*. Ese "24/24" se refiere a "controles que aparecen en el reporte
  MASVS" (incluyendo los que salen como `—`/gap), no a "controles con check".
  Usar el mismo número (24) con significados opuestos genera confusión.
- **Acción:** en `plan.md` renombrar a *"24/24 controles reportados (18 con
  check, 6 como gap)"* para desambiguar.

### A6. Conteo de tests: 12 números distintos sin fuente única

- `plan.md` reporta secuencialmente 54 → 73 → 95 → 110 → 119 → 125 → 130 → 167
  → 169 → 181 → 197 → **202** (último). Ningún documento dice "estado actual:
  202 tests, 21 archivos".
- **Acción:** añadir una línea canónica en `ROADMAP.md` o `README.md`
  (*"Test suite: 202 tests, 21 files, 100% passing"*) y, si es posible,
  actualizarla vía CI.

---

## B) Mejoras de estructura y calidad documental

### B1. `plan.md` mezcla 4 roles distintos en 1.403 líneas

Hoy contiene simultáneamente:

1. El **plan original** (texto prospectivo de Fases 0-3, en presente/futuro).
2. El **estado post-implementación** de cada fase.
3. Una **bitácora de sesiones** con bugs encontrados y diálogos con el usuario.
4. **Notas operacionales** (IPs de dispositivos, jobs reales vistos en DB).

El texto del plan original no se editó tras implementar, así que convive
"vamos a hacer X" con "ya hicimos X" en el mismo documento.

- **Acción:** separar en `plan.md` (plan + decisiones, congelado) +
  `DECISIONS.md` o `CHANGELOG.md` (bitácora cronológica de lo implementado,
  bugs, desviaciones). Esto reduciría `plan.md` a ~400-500 líneas legibles.

### B2. No existe `CHANGELOG.md`

Verificado: no hay `CHANGELOG*` en el repo. Toda la evolución vive enterrada en
`plan.md`. Para un proyecto con versiones (`0.2.0`) y entry points publicados,
un changelog formato Keep-a-Changelog es estándar y barato.

- **Acción:** crear `CHANGELOG.md` con entradas derivadas de las secciones
  fechadas (2026-07-24/25/26/27) de `plan.md`.

### B3. Duplicación/dispersión de flujos entre `docs/*.md` y `plan.md`

- El flujo del análisis estático vive en `docs/static-analysis-flow.md` (319
  líneas) **y** parcialmente repetido en `plan.md`.
- El flujo de aipwn vive en `docs/aipwn-flow.md` (478 líneas) **y** en `plan.md`
  (Fase 3/4).
- Las decisiones de la cola viven en `plan.md` (Fase 1), en `README.md`
  (sección Mass Execution), y en `docs/static-analysis-flow.md` (sección
  integración con dashboard).
- **Acción:** `plan.md` debería *enlazar* a `docs/*.md` en vez de duplicar;
  los `docs/*.md` son el source of truth del "cómo funciona hoy".

### B4. `ROADMAP.md` es demasiado escaso (34 líneas) frente al `plan.md` (1.403)

El roadmap público no refleja el estado real del proyecto. Items completados
(cola, scheduler, dashboard, WebUSB, batch desde dashboard) están marcados como
`[x]` solo porque se añadió a posteriori, pero faltan categorías enteras que sí
aparecen en `plan.md` (p. ej. "limitaciones conocidas", "deuda técnica
deliberada").

- **Acción:** añadir a `ROADMAP.md` una sección *"Known limitations / deliberate
  debt"* que consolide los gaps honestos hoy dispersos: misconfigs no
  persistidos en `findings`, solo 2 checks dinámicos, MASWE ausente en manifest
  findings, techo de fps en scrcpy file-based, bug del multimodal con z.ai.

### B5. Falta un diagrama de arquitectura general

`docs/aipwn-flow.md` y `docs/static-analysis-flow.md` tienen buenos diagramas
ASCII de *flujo de un análisis*, pero no hay una vista de *arquitectura del
sistema* (core `nutcracker_core/{store,queue,scheduler,checks,orchestrator}`
vs plugins `plugins/{aipwn,aireview,dashboard}` vs externos
`adb`/`frida`/`scrcpy`/`semgrep`). La frontera core/plugin se menciona mucho
pero nunca se *dibuja*.

- **Acción:** un diagrama `docs/architecture.md` con los 3 niveles (core /
  plugins / herramientas externas) y las flechas de dependencia.

### B6. `README.md` (984 líneas) mezcla audiencias

Contiene instalación (macOS/Linux/Windows), features, arquitectura, ejemplos de
CLI, dashboard, OWASP, todo junto. Es la cara del proyecto pero pesa de leer.

- **Acción:** mover detalle a `docs/` y dejar en `README.md` un índice +
  quickstart + links (target <400 líneas).

### B7. Inconsistencia menor de URLs de badges

- `README.md` badges apuntan a `github.com/drneox/nutcracker`.
- El workspace del repo es `github.com/Hiteek/nutcracker` (remote `origin`).
- **Acción:** decidir cuál es el canonical y unificar (o documentar el
  fork/mirror).

### B8. `docs/aipwn-flow.md` lista "27 Herramientas del agente" sin validación

El detalle dice "27" en el título de la sección 5; sería útil validar ese número
contra `TOOL_SCHEMAS` real (puede haber cambiado tras Fase 3/4) y, si es
automatable, generarlo desde el código como ya se hace con
`owasp-mas-coverage.md`.

- **Acción:** script `tools/gen_agent_tools.py` (o similar) que regenere esa
  lista desde `frida_agent_tools.py`, evitando drift.

---

## Resumen priorizado

| # | Tipo | Severidad | Esfuerzo |
|---|---|---|---|
| A1 (`queue rm` inexistente) | inconsistencia | media | bajo |
| A2 (`batch --due` inexistente) | inconsistencia | media | medio |
| A4 (ROADMAP contradice plan sobre WebUSB) | inconsistencia | alta | bajo |
| A3 (`schema.sql` incompleto) | inconsistencia | media | bajo |
| A5 (métrica "cobertura" ambigua) | inconsistencia | baja | bajo |
| A6 (conteo de tests sin fuente única) | inconsistencia | baja | bajo |
| B1 (`plan.md` mezcla 4 roles) | mantenibilidad | alta | medio |
| B2 (sin `CHANGELOG`) | mantenibilidad | media | bajo |
| B3 (duplicación de flujos) | mantenibilidad | media | medio |
| B4 (ROADMAP escaso) | calidad externa | media | bajo |
| B5 (sin diagrama de arquitectura) | calidad | media | medio |
| B6 (`README.md` mezcla audiencias) | calidad | baja | medio |
| B7 (URLs de badges) | calidad | baja | bajo |
| B8 (conteo de tools del agente) | calidad | baja | bajo |

## Orden sugerido de ejecución

Las de **severidad alta** (A4 y B1) son las que más impactan la confianza de
quien lee: A4 porque dos documentos del mismo repo se contradicen sobre si algo
está verificado, y B1 porque `plan.md` se volvió ilegible en su estado actual
(1.403 líneas que mezclan plan, bitácora y notas operacionales).

Orden propuesto:

1. **A4** — sincronizar ROADMAP ↔ plan sobre WebUSB (cambio textual, bajo
   esfuerzo).
2. **B2** — crear `CHANGELOG.md` (base para el refactor de B1).
3. **A1 / A2** — decidir implementar o eliminar `queue rm` / `batch --due`.
4. **B1** — refactor de `plan.md` (plan + decisiones vs bitácora).

## Referencias cruzadas

- `plan.md` — historial detallado de las Fases 0-4.
- `ROADMAP.md` — ítems pendientes/completados (a sincronizar).
- `docs/static-analysis-flow.md` — flujo del análisis estático.
- `docs/aipwn-flow.md` — flujo del agente aipwn.
- `docs/owasp-mas-coverage.md` — matriz de cobertura MASVS (generada desde el
  registry).