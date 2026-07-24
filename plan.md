# Plan — Nutcracker: ejecución masiva, alineación OWASP MAS y dashboard

## Context

`nutcracker` es una herramienta de seguridad móvil (Android APK) en Python, **solo terminal**, con
un CLI monolítico (`nutcracker.py`, 2.260 líneas) sobre un paquete `nutcracker_core/`
(analyzer, pipeline runtime, detectores, scanners, MASVS) y un **sistema de plugins limpio**
(`nutcracker_core/plugins/__init__.py`): auto-descubre carpetas con
`register(cli)` y expone post-hooks `after_analysis` / `after_batch`. Toda la IA vive como plugin
(`aipwn` = dos agentes ReAct Frida/exploit vía `any-llm-sdk`; `aireview` = filtro de falsos positivos).

Tres accionables, en fases:

1. **Ejecución masiva de APKs** con cola paralela/secuencial y **scheduler configurable** que garantice
   revisiones periódicas (por defecto ≥1/mes por APK).
2. **Alinear el proyecto a OWASP MAS** (MASVS + MASWE + MASTG + CWE) ejecutando **tests `.py`
   deterministas** estáticos y dinámicos para mejores hallazgos.
3. **Dashboard web** (hoy todo es terminal): vídeo del celular físico vía scrcpy, logs junto al
   razonamiento del agente, panel del prompt del sistema/turno, y un chat que dispare acciones del agente.

Además: refactoring para mantenibilidad, respetando la frontera **core (todo) vs plugin (IA)**.

### Estado actual — huecos confirmados
- **Batch** (`nutcracker.py:1964`): bucle **secuencial**, sin persistencia, sin cola,
  sin paralelismo, sin estado entre corridas, **sin scheduling**.
- **Persistencia**: solo archivos planos `reports/<pkg>/*.json` + PDF (`reporter.py:184`).
  **No hay BD**.
- **OWASP**: mapeo a **MASVS v2** (subconjunto, `masvs.py`) + reglas semgrep MASTG
  (`MSTG-*`). **MASWE ausente por completo**; **CWE** solo se consume de metadata de semgrep para escalar
  severidad, no forma parte de la taxonomía emitida. Checks en 3 registros hechos a mano
  (`RULES` en `vuln_scanner.py`, `_NATIVE_RULES` en `native_scanner.py`,
  `ALL_DETECTORS` en `analyzer.py:87`).
- **Tests**: casi inexistentes (solo `tests/test_i18n.py`).
- **IA**: agentes ya imprimen razonamiento (`aipwn.show_thinking`) y streamean salida Frida/logcat en
  tiempo real (`frida_capture.py`) — buena base para el dashboard.
  No hay streaming a nivel de token ni chat interactivo humano.

### Restricción arquitectónica clave (define la cola)
El análisis **estático** (decompilación, semgrep, regex, native, OSINT) **paraleliza bien**.
El análisis **dinámico** (Frida/ADB/aipwn) usa el **teléfono físico** y **debe serializarse por
dispositivo** — dos análisis dinámicos no pueden compartir el mismo celular. La cola será:
**pool paralelo para estático + lock por dispositivo para dinámico**.

### ⚠️ Nota de seguridad
`config.yaml` local contiene secretos reales (`google_play.aas_token`, `llm.api_key` de z.ai/GLM).
Está en `.gitignore` (no se commitea) — bien, pero **rotar la `api_key` LLM** si estuvo expuesta y
migrar secretos a variables de entorno (ver Fase 0).

### Decisiones tomadas
- Orden: **Fase 0 Fundamentos → Fase 1 Cola/Scheduler → Fase 2 OWASP MAS → Fase 3 Dashboard**.
- Persistencia: **SQLite** (un archivo, sin servidor extra).
- Ejecución: **daemon con scheduler interno** (`nutcracker serve` + APScheduler + pool + lock por device).
- Dashboard: **web local** (FastAPI + WebSocket + SPA), scrcpy embebido, como **plugin nuevo**.

---

## Estado de avance (actualizado 2026-07-24)

Revisión del commit `90fad34 "advance"` (posterior al commit del plan) contra este documento.

| Fase | Estado |
|---|---|
| **Fase 0** — Fundamentos + persistencia | ✅ **Completa** (con 2 huecos menores, ver abajo) |
| **Fase 1** — Cola + scheduler | ✅ **Completa** (POC verificada con APK real, ver abajo) |
| **Fase 2** — OWASP MAS | ✅ **Completa** (investigación real + bugs de mapeo corregidos, ver abajo) |
| **Fase 3** — Dashboard web | ✅ **Completa** (alcance ajustado en 2 puntos, ver abajo) |

### Fase 0 — detalle de lo entregado
- ✅ **0.1 Persistencia SQLite**: `nutcracker_core/store/{db,repository,hooks}.py` + `schema.sql`.
  Esquema idéntico al planeado (`apps, runs, findings, artifacts, schedule`); `findings` ya trae
  columnas `masvs/maswe/cwe` listas para la Fase 2. Doble escritura no destructiva confirmada:
  `store/hooks.py::install()` registra el post-hook `after_analysis` (llamado desde
  `cli/__init__.py`); los JSON/PDF en disco siguen intactos.
- ✅ **0.2 Refactor CLI**: `nutcracker.py` quedó en **18 líneas** (shim → `from nutcracker_core.cli
  import cli`). Paquete `cli/` con `scan.py, analyze.py, launch.py, batch.py, setup_token.py,
  regen_pdf.py`. `orchestrator.py` (1.496 líneas) concentra la orquestación.
- ✅ **0.3 Split de `vuln_scanner.py`**: nacieron `scan_types.py` y `leak_scanner.py`;
  `vuln_scanner.py` bajó de ~1.400 a 456 líneas.
- ✅ **0.4 Config con env vars**: `config.py` soporta `${ENV_VAR}`, cubierto por
  `tests/test_config_env.py`.
- ✅ **0.5 Tests base**: `pytest.ini`, `requirements-dev.txt`, `conftest.py` +
  `test_store_repository.py`, `test_store_hooks.py`, `test_config_env.py`,
  `test_vuln_scanner_split.py`.

**Huecos abiertos dentro de Fase 0 (bloquean el arranque limpio de Fase 1):**
1. ⚠️ `orchestrator.py` es un *lift-and-shift* de los helpers privados de `nutcracker.py`
   (`_run_analysis`, `_post_analysis_flow`, etc.) — **no expone todavía una API pública
   `run_static(apk)` / `run_dynamic(apk, serial)`** reutilizable por la cola. Se resuelve como
   primer paso de la Fase 1 (ver 1.0 abajo) en vez de re-tocar Fase 0.
2. ❌ `pyproject.toml` no existe todavía (empaquetado/entry-points) — se deja para el cierre de
   mantenibilidad transversal, no bloquea Fase 1.
3. ❔ Suite `pytest` no verificada en este entorno por falta de dependencias instaladas
   (`loguru` ausente); es un problema de entorno local, no del código — pendiente de confirmar
   en un entorno con `pip install -r requirements.txt -r requirements-dev.txt`.

### Fase 1 — qué falta (antes de empezar, confirmado por inspección directa)
Nada de esto existe aún: `nutcracker_core/queue/` (engine con pool estático paralelo + lock por
device), `nutcracker_core/scheduler.py` (APScheduler), comando `nutcracker serve`, comandos
`queue`/`schedule`, `batch --due`. `apscheduler` no está en `requirements.txt`.

---

## Principios rectores
- **No romper la frontera core/plugin.** Persistencia, cola y scheduler son **core** (utilidad general).
  El dashboard y la IA son **plugins**. El dashboard **lee** el store SQLite y **consume** el streaming de
  aipwn; no mete lógica de negocio en el plugin.
- **Reusar lo que existe:** el registro de post-hooks (`register_post_hook`/`fire_post_hooks`), el loader de
  plugins, `load_config`/`cfg_get`, los dataclasses `AnalysisResult`/`ScanResult`/`MASVSReport`, y el
  streaming ya presente en `frida_capture.py` y `print_agent_thinking`.
- **Incremental y verificable:** cada fase deja el CLI actual funcionando y añade una capa.

---

## Fase 0 — Refactoring + fundamentos de persistencia (SQLite)

Objetivo: crear la base de estado compartido y ordenar el código antes de construir encima.

### 0.1 Capa de persistencia SQLite (core, nuevo)
Nuevo módulo `nutcracker_core/store/` (paquete):
- `schema.sql` / `db.py` — conexión (stdlib `sqlite3`, WAL mode), migraciones simples versionadas.
- Tablas:
  - `apps(package PK, source, first_seen, last_run_at, next_due_at, schedule_json)`
  - `runs(id PK, package FK, kind[static|dynamic|full], status[queued|running|done|error], started_at, finished_at, verdict, masvs_score, grade, error)`
  - `findings(id PK, run_id FK, rule_id, title, severity, category, masvs, maswe, cwe, file, line, confirmed[bool])`
  - `artifacts(run_id FK, type[pdf|json|screenshot|frida_log], path)`
  - `schedule(package FK, interval_days default 30, cron, enabled, last_scheduled_at)`
- `repository.py` — funciones CRUD tipadas (`insert_run`, `update_run_status`, `record_findings`,
  `apps_due(before)`, `history(package)`).
- **Doble escritura no destructiva:** un post-hook core `after_analysis` persiste el resultado en SQLite
  reusando el flujo actual (los JSON/PDF en disco siguen igual). Convertir hallazgos con los dataclasses
  existentes (`VulnFinding`, `MASVSReport.to_dict()`).

### 0.2 Refactor del CLI monolítico
`nutcracker.py` (2.260 líneas) → paquete `nutcracker_core/cli/`:
- `cli/__init__.py` (grupo Click + `load_plugins`), `cli/scan.py`, `cli/analyze.py`, `cli/launch.py`,
  `cli/batch.py`, `cli/setup_token.py`, `cli/regen_pdf.py`.
- Extraer los helpers de orquestación (`_run_analysis`, `_post_analysis_flow`, `_generate_pdf`,
  `save_analysis_json`) a `nutcracker_core/orchestrator.py` para que **CLI, daemon y dashboard** los
  reusen sin duplicar. Este es el cambio de mantenibilidad más importante.
- Mantener `nutcracker.py` como shim de entrada (`from nutcracker_core.cli import cli`).

### 0.3 Split de `vuln_scanner.py` (ya en ROADMAP)
Ejecutar el ítem de arquitectura del `ROADMAP.md`: dividir en `scan_types.py` (dataclasses),
`vuln_scanner.py` (regex+semgrep), `leak_scanner.py` (apkleaks+gitleaks). Prerrequisito para escalar checks
en Fase 2.

### 0.4 Higiene de config/secretos
- Soportar `${ENV_VAR}` en `load_config` (`config.py`) para leer
  `llm.api_key`, `google_play.aas_token`, tokens OSINT desde entorno.
- Documentar en `config.yaml.example`. Rotar la key expuesta.

### 0.5 Base de tests
Introducir `pytest` + `pytest.ini`, fixtures con un APK mínimo de prueba, y tests unitarios para
`store/repository.py`, el mapeo MASVS y el split del scanner. CI opcional (GitHub Actions) en esta fase.

---

## Fase 1 — Ejecución masiva: cola paralela/secuencial + scheduler

Objetivo: `nutcracker serve` = daemon que encola, ejecuta y agenda revisiones periódicas.

### 1.1 Cola y workers (core, nuevo `nutcracker_core/queue/`)
- `job.py` — `Job(package, kind, priority, created_at)` persistido en tabla `runs` (estado `queued`).
- `engine.py` — orquestador basado en `concurrent.futures`:
  - **Pool estático** configurable (`queue.static_workers`, p.ej. 4) para la fase estática de cada APK.
  - **Lock por dispositivo** (`threading.Lock` keyed por `serial`) que serializa la fase
    dinámica; si hay varios devices en `adb devices`, un lock por serial permite paralelismo entre
    teléfonos pero nunca en el mismo.
  - Reusa `orchestrator.run_static(apk)` y `orchestrator.run_dynamic(apk, serial)` de Fase 0.2.
- Modo **secuencial** = pool de tamaño 1 (config `queue.mode: sequential|parallel`).

### 1.2 Scheduler (core, APScheduler)
- `scheduler.py` — `BackgroundScheduler` de APScheduler:
  - Job maestro periódico (cada hora) que llama `repository.apps_due(now)` y encola las apps cuyo
    `next_due_at <= now`.
  - Al terminar un run, recalcula `next_due_at = finished_at + interval_days` (default **30**).
  - Config por app (tabla `schedule`) y global en `config.yaml` (nuevo bloque `scheduler:` con
    `default_interval_days: 30`, `cron` opcional, `enabled`).
- Añadir `requirements.txt`: `apscheduler>=3.10`.

### 1.3 Comandos CLI nuevos
- `nutcracker serve` — arranca daemon (scheduler + cola + API para el dashboard de Fase 3).
- `nutcracker queue add <lista|pkg>` / `queue ls` / `queue rm`.
- `nutcracker schedule set <pkg> --every 30d` / `schedule ls`.
- `nutcracker batch --due` — modo one-shot para cron/systemd (encola solo lo vencido y sale), por si se
  quiere disparo externo además del daemon.
- Migrar el `batch` actual para que use `queue.engine` (compatibilidad hacia atrás con `list_file`).

**Entregable:** dado un `list_file`, el daemon revisa todas las apps, respeta paralelo/secuencial y
device-lock, y re-agenda cada una a ≥1/mes automáticamente.

### Fase 1 — estado: ✅ completa (2026-07-24)

**Decisiones de diseño tomadas durante la implementación** (ajustan el texto original de
esta fase a la realidad del código):

- **Aislamiento por subproceso, no por hilo in-process.** `orchestrator.py` guarda estado
  mutuo a nivel de módulo (`_CFG`, `_MANIFEST_ANALYSIS`, `_OSINT_RESULT`, ...) que no es
  thread-safe entre análisis concurrentes. En vez de reescribir 1.500 líneas ya probadas,
  cada job de la cola corre como **subproceso aislado** que invoca el CLI existente
  (`nutcracker analyze`/`scan`), construido por el nuevo helper
  `orchestrator.build_job_cmd()`. Esto da paralelismo real (sin GIL) y reutiliza el
  pipeline al 100% sin duplicar lógica. `apply_static_only_override()` + el flag
  `--static-only` (nuevo en `analyze`/`scan`) fuerzan decompilación jadx y evitan tocar el
  dispositivo en jobs estáticos.
- **Tabla `queue_jobs` separada de `runs`** (migración SQLite versión 2, `store/db.py`).
  `runs` sigue siendo escrita únicamente por el post-hook `after_analysis`
  (`store/hooks.py`) al terminar un análisis real; `queue_jobs` registra el ciclo de vida
  del encolado (`queued→running→done/error`) y se enlaza al `run_id`/`package` reales vía
  la variable de entorno `NUTCRACKER_QUEUE_JOB_ID`, que el subproceso hijo propaga hasta el
  hook (`repository.link_job_run`). Evita adivinar el package antes de que el análisis
  termine y evita duplicar filas en `runs`.
- **Bug real encontrado y corregido en la propia POC:** cada invocación de
  `nutcracker queue add` es un proceso nuevo, así que la lista en memoria `_pending` de
  `QueueEngine` no persiste entre invocaciones. Sin corrección, un job encolado por un
  proceso y drenado por otro (`queue add --run`, o el scheduler en un tick posterior) se
  quedaba huérfano en estado `queued` para siempre. Fix: `QueueEngine.drain()` ahora llama
  primero a `_load_queued_from_db()`, que recupera de SQLite cualquier job `queued` no
  presente ya en memoria. `NutcrackerScheduler._tick()` también se simplificó para llamar
  siempre a `drain()` (barato si no hay nada pendiente) en vez de solo cuando hay apps
  vencidas — así recoge jobs encolados manualmente entre ticks. Regresión cubierta por
  `tests/test_queue_engine.py::test_drain_picks_up_jobs_queued_by_a_different_engine_instance`.
- **`batch` NO se migró internamente al motor de cola** (a diferencia de lo previsto en el
  texto original de 1.3). Se mantiene tal cual (secuencial, in-process, ya probado) por
  relación riesgo/beneficio: reescribirlo para reconstruir su resumen consolidado
  (severidades, top findings, categorías) desde SQLite en vez de desde objetos en memoria
  era un cambio de mayor superficie sin beneficio claro sobre simplemente ofrecer el nuevo
  camino recomendado (`queue add <list_file>` + `serve`/`--run`), que cubre el mismo
  entregable (revisión masiva con paralelo/secuencial + reintentable + persistida). Pendiente
  como mejora futura opcional, no bloqueante.

**Archivos nuevos:** `nutcracker_core/queue/{__init__,job,engine}.py`,
`nutcracker_core/scheduler.py`, `nutcracker_core/cli/{serve,queue_cmd,schedule_cmd}.py`,
`tests/test_queue_engine.py`, `tests/test_scheduler.py`.
**Archivos modificados:** `store/schema` (vía migración en `db.py`, no se tocó
`schema.sql`), `store/repository.py` (CRUD de `queue_jobs`), `store/hooks.py` (enlace
`NUTCRACKER_QUEUE_JOB_ID`→`run_id`), `orchestrator.py` (`apply_static_only_override`,
`build_job_cmd`), `cli/analyze.py`+`cli/scan.py` (`--static-only`), `cli/__init__.py`
(registro de comandos), `requirements.txt` (+`apscheduler`), `config.yaml(.example)`
(bloques `queue:`/`scheduler:`), `.gitignore` (+`nutcracker.db*`).

**Comandos nuevos:** `nutcracker serve`, `nutcracker queue add|ls`, `nutcracker schedule set|ls`.

**Tests:** 15 nuevos (`test_queue_engine.py` ×11, `test_scheduler.py` ×5, uno reemplazado) —
paralelismo estático (timing), modo secuencial (pool=1), serialización por device-lock
(mismo serial nunca solapa; seriales distintos sí paralelizan), rechazo de job dinámico sin
`.apk` local, transición de estados + error capturado, re-agendado automático, filtro de
`enqueue_due_apps`, y el fix de recuperación cross-proceso. Suite completa: **54/54 passing**
(`.venv` con `requirements.txt`+`requirements-dev.txt`).

**POC real ejecutada** (no simulada): usando un APK legítimo de Mobile Hacking Lab
(`com.mobilehackinglab.iotconnect.apk`, entorno de entrenamiento en seguridad móvil ya
presente en la máquina del usuario) copiado 3 veces, encolado vía **3 invocaciones
separadas** de `nutcracker queue add` (2 sin `--run`, la 3ª con `--run`):
- Los 3 jobs — encolados por procesos CLI distintos — se recuperaron y ejecutaron desde la
  3ª invocación gracias al fix de `_load_queued_from_db()`.
- `queue_jobs.started_at` idéntico para los 3 (`17:54:29`) → paralelismo real confirmado,
  no solo solapamiento parcial. Tiempo total: **~84s** vs. ~174s que habría tomado en serie
  (58s por análisis real × 3), con `static_workers=3`.
- Cada job quedó enlazado a su `run_id`/`package` real (`com.mobilehackinglab.iotconnect`,
  verdict `protected`, score 80, grade B, 1 finding `AUTH001`→`MASVS-AUTH-2`) — persistido en
  `runs`/`findings` por el hook de Fase 0, sin tocarlo.
- **Re-agendado automático confirmado**: sin que nadie llamara `schedule set`, al terminar
  el primer job se creó una fila en `schedule` (`interval_days=30, enabled=1`) y
  `apps.next_due_at` quedó en `last_run_at + 30 días` — el requisito "≥1 revisión/mes por
  defecto" queda satisfecho automáticamente para cualquier app que pase por la cola.
- `nutcracker serve` arranca (banner + "daemon activo"), corre el scheduler en background, y
  se apaga con SIGTERM imprimiendo "nutcracker serve — detenido" (sin colgarse).
- `nutcracker queue ls` y `nutcracker schedule ls` muestran el estado correctamente.

---

## Fase 2 — Alineación OWASP MAS (MASVS + MASWE + MASTG + CWE) con tests `.py` deterministas

Objetivo: convertir los 3 registros hechos a mano en un **framework de checks unificado** con taxonomía
OWASP completa y separación estático/dinámico explícita.

### 2.1 Taxonomía enriquecida
- Extender el modelo de finding (`VulnFinding` / `scan_types.py`) con campos **`maswe`** (p.ej. `MASWE-0001`)
  y **`cwe`** de primera clase (hoy CWE solo vive en metadata semgrep, `vuln_scanner.py:997`).
- En `masvs.py`: añadir `RULE_TO_MASWE` y `RULE_TO_CWE` junto a los `RULE_TO_MASVS`
  existentes; añadir `MASTG_TEST_IDS` por control. Completar `MASVS_CONTROLS` a la matriz v2 real
  (incluir PRIVACY y los controles hoy ausentes) para que el "24 controles" del README sea fiel.
- Reporte (`reporter.py` + `pdf_reporter.py`) muestra por hallazgo: MASVS control, MASWE, CWE, MASTG test-id.

### 2.2 Framework de checks deterministas `.py` (core, nuevo `nutcracker_core/checks/`)
Unificar el patrón de extensión (hoy fragmentado en 3 listas) en un registro con auto-descubrimiento,
al estilo del loader de plugins:
- `checks/base.py` — `class Check` con metadata `id, masvs, maswe, cwe, mastg, severity, kind[static|dynamic]`
  y método `run(ctx) -> list[Finding]`. `ctx` da acceso a: dir decompilado, manifest, strings/classes,
  y (para dinámicos) `serial`/handle Frida.
- `checks/registry.py` — descubre todos los `Check` en `checks/static/` y `checks/dynamic/`.
- **Estáticos** (`checks/static/*.py`): portar las `VulnRule`/`_NativeRule`/detectores actuales como checks
  deterministas mapeados a MASWE/MASTG. Cada uno es un `.py` testeable de forma aislada.
- **Dinámicos** (`checks/dynamic/*.py`): checks deterministas que corren en el device **sin** LLM
  (p.ej. verificar `android:debuggable`, cleartext real, componentes exportados invocables, prefs
  world-readable) — extrae la lógica ya presente en las 12 tools de
  `exploit_agent.py` hacia checks core reutilizables; el
  agente LLM queda para lo que requiere razonamiento/adaptación (bypass RASP).
- El scanner actual (`scan_directory`, `scan_native_libs`, detectores) se convierte en runner que itera el
  registry. Migración gradual: registry envuelve los registros existentes primero, luego se portan.

### 2.3 Suite de tests deterministas (MASTG-style)
- `tests/checks/` con casos por check (APK/snippet mínimo → hallazgo esperado), garantizando
  determinismo y no-regresión. Mapear cada test a su MASTG test-id.
- Doc `docs/owasp-mas-coverage.md` (matriz MASVS×MASWE×MASTG cubierto / pendiente), generable desde el
  registry.

### 2.4 Cerrar ítem de ROADMAP relacionado
Implementar la separación **verdicto estático vs bypass dinámico** ya especificada en
`ROADMAP.md` (`aipwn_bypass_confirmed` en `AnalysisResult`, ajuste en `build_masvs_report`),
que encaja naturalmente con el modelo `runs.kind` de Fase 0.

### Fase 2 — estado: ✅ completa (2026-07-24)

**Investigación previa a implementar** (obligatoria dado el riesgo de inventar datos de un
estándar): se extrajeron los **24 controles MASVS v2.1 reales** (8 categorías, incluyendo
PRIVACY) y el **catálogo completo de 119 debilidades MASWE** directamente de
`mas.owasp.org/MASVS/`, `github.com/OWASP/masvs` (`OWASP_MASVS.yaml`, fuente estructurada de
verdad) y `github.com/OWASP/maswe` — no se inventó ningún ID ni texto de control.

**Hallazgo central: el repo tenía 3 bugs reales de mapeo MASVS**, encontrados al contrastar
el código contra el texto oficial de cada control (no eran solo "faltaban campos", eran
mapeos **incorrectos**):
1. **RESILIENCE-2 ↔ RESILIENCE-4 invertidos.** El texto oficial de R2 es "anti-tampering"
   (firma del paquete, integridad DEX/nativo/recursos) y R4 es "anti-dynamic-analysis"
   (debugger, Frida, hooking) — el código tenía exactamente al revés. Afectaba
   `DETECTOR_TO_MASVS` ("APK signature verification" → R4 en vez de R2; detección de Frida →
   R2 en vez de R4) y `RULE_TO_MASVS` (NAT003, anti-debug nativo → R2 en vez de R4).
2. **AUTH001/DBG002/DBG003/INFO001 en controles cuyo texto oficial no aplicaba.** "Token en
   logs" estaba en `MASVS-AUTH-2` (cuyo texto real es autenticación local biométrica/PIN, no
   logs) y `android:debuggable=true` estaba en `MASVS-CODE-2` (cuyo texto real es "mecanismo
   de actualización forzada", no debuggable). El catálogo MASWE oficial resolvió la duda:
   `MASWE-0001` ("Insertion of Sensitive Data into Logs") está catalogada bajo **STORAGE**, y
   `MASWE-0067` ("Debuggable Flag Not Disabled") está catalogada bajo **RESILIENCE** → movidas
   a `MASVS-STORAGE-2` y `MASVS-RESILIENCE-4` respectivamente (ambas además calzan con el
   texto oficial de esos controles, que mencionan "logs" y "anti-dynamic-analysis" de forma
   explícita).
3. **`COMP008` (ContentProvider exportado) no tenía ningún mapeo MASVS/MASWE/CWE** — se
   generaba en runtime (`vuln_scanner.scan_manifest_components`) pero nunca se agregó a
   `masvs.py` cuando se creó la regla.

También se corrigió un **bug de implementación** encontrado por los tests del framework
nuevo (no de mapeo, de código): los 8 checks adaptados desde `analyzer.ALL_DETECTORS`
resolvían su metadata MASVS consultando `RULE_TO_MASVS` (por rule_id) en vez de
`DETECTOR_TO_MASVS` (por nombre de detector) — un dict distinto —, dejando **toda** la
metadata de los detectores de resiliencia vacía. Detectado por
`test_static_adapter_wraps_all_existing_rules` al generar `docs/owasp-mas-coverage.md` y ver
`MASVS-RESILIENCE-1` casi sin checks listados pese a tener 5 detectores activos.

**Entregado:**
- `masvs.py` reescrito: 24 controles reales + `MASWE_CATALOG` (119 entradas) + `RULE_TO_MASWE`
  + `RULE_TO_CWE` (mapeo prudente: solo reglas con correspondencia semántica clara y
  verificable contra el catálogo oficial — no se forzó un MASWE/CWE para reglas ambiguas como
  HC007/HC008/ST006/OBF001/NAT001/NAT006/NAT007, que se dejaron sin esa dimensión antes que
  inventar una).
- `store/hooks.py`: cada `finding` persistido ahora trae `masvs` + `maswe` + `cwe` (los
  campos ya existían en el schema desde Fase 0 pero nunca se poblaban). Verificado con APK
  real: `AUTH001|high|MASVS-STORAGE-2|MASWE-0001|CWE-532`.
- `nutcracker_core/checks/`: framework unificado (`base.py`, `registry.py`) +
  `checks/static/adapter.py` (envuelve los ~54 checks existentes de
  `vuln_scanner.RULES`/`EXPORTED_COMPONENT_RULES`/`native_scanner._NATIVE_RULES`/
  `analyzer.ALL_DETECTORS` sin reescribirlos — riesgo/beneficio de portar ~50 reglas maduras a
  archivos individuales no lo justificaba) + `checks/dynamic/` con **2 checks dinámicos reales
  y testeados** (no simulados): `DebuggableDynamicCheck` (confirma `debuggable` en runtime vía
  `run-as`/JDWP) y `CleartextTrafficDynamicCheck` (detecta HTTP en claro en logcat durante la
  ejecución) — ambos con un `adb_run` inyectable, así que corren deterministas en CI sin
  dispositivo. Se hizo un pequeño refactor seguro en `vuln_scanner.py`: `component_tags` (dict
  local dentro de `scan_manifest_components`) se elevó a constante de módulo
  `EXPORTED_COMPONENT_RULES` para que el adaptador la reuse sin duplicarla.
- `docs/owasp-mas-coverage.md`, generado por `tools/gen_owasp_coverage.py` desde el registry
  (no escrito a mano — regenerar tras cualquier cambio en `masvs.py`/checks). Resultado
  honesto: **14/24 controles MASVS con al menos un check**; los 10 sin cobertura
  (AUTH-1/2/3, CRYPTO-2, CODE-1/2/3, PRIVACY-2/3/4) quedan listados explícitamente como gap,
  no ocultos.
- ROADMAP "Differentiate runtime bypass vs DEX extraction": `AnalysisResult.aipwn_bypass_confirmed`
  (nuevo campo, con roundtrip en `to_dict`/`from_dict`); `build_masvs_report()` ahora usa
  `analysis.protection_broken or analysis.aipwn_bypass_confirmed`; banner de terminal nuevo
  ("BYPASS CONFIRMED" / i18n EN+ES) en `orchestrator._print_verdict` cuando el bypass viene de
  aipwn sin extracción de DEX. **Nota importante:** `nutcracker_core/plugins/aipwn/` es un
  **repositorio git separado** (remoto propio `github.com/nutcracker-sh/aipwn`, invisible al
  `git status` de este repo) — la parte de wiring en `plugins/aipwn/__init__.py` (marca
  `aipwn_bypass_confirmed=True` y re-guarda el JSON tras un `report_success`) quedó modificada
  ahí y **no se comiteó** (fuera del alcance de este repo; coordinar con el mantenedor de ese
  repo aparte si se quiere subir).
- Actualización de la sección "PDF report con sección de dynamic analysis" (punto 3 del item
  de ROADMAP) — **diferida**: el banner de terminal y el campo/score ya distinguen ambos
  verdictos; el rediseño de la sección PDF es un cambio de mayor superficie en
  `pdf_reporter.py` que se deja como follow-up explícito, no reclamado como hecho.

**Deliberadamente fuera de alcance de esta fase** (para no sobre-extender el ya considerable
trabajo de investigación + implementación):
- Migrar cada regla existente a un archivo `.py` individual dentro de `checks/static/` (el
  adaptador cubre la meta de "registry unificado" sin ese riesgo/costo).
- MASWE a nivel de `MISCONFIG_TO_MASVS` (hallazgos del manifest): se corrigieron los IDs
  MASVS incorrectos ahí, pero no se añadió una dimensión MASWE porque esos hallazgos no pasan
  hoy por `store/hooks.py::_to_finding_record` (no se persisten en la tabla `findings`) —
  extenderlo requeriría antes persistir misconfigs, que es un cambio de alcance mayor.
- Checks dinámicos adicionales más allá de los 2 implementados (debuggable, cleartext) —
  quedan como candidatos naturales para portar más lógica de
  `plugins/aipwn/exploit_agent.py` cuando se retome esta fase.

**Tests:** 19 nuevos (`tests/checks/test_registry.py` ×8, `tests/checks/test_dynamic_checks.py`
×7, `tests/test_masvs_bypass.py` ×4). Suite completa: **73/73 passing**. Verificado también
con el mismo APK real de Mobile Hacking Lab usado en la POC de Fase 1: coverage pasó de
15/24 a **24/24** controles reportados, `MASVS-STORAGE-2` ahora captura correctamente el
finding `AUTH001` (antes atado a un `AUTH-2` cuyo texto oficial no aplicaba), y
`MASVS-PRIVACY-1`/`MASVS-RESILIENCE-2` pasan a tener veredicto real (antes vacíos).

---

## Fase 3 — Dashboard web local (plugin nuevo)

Objetivo: plugin `nutcracker_core/plugins/dashboard/` que sirva una web local sobre el daemon de Fase 1.
Respeta la frontera: **lee SQLite y consume el streaming de aipwn**; no reimplementa análisis.

### 3.1 Backend (FastAPI, dentro del daemon `serve`)
- `dashboard/server.py` — FastAPI montado por `nutcracker serve`:
  - REST: `/api/apps`, `/api/runs`, `/api/runs/{id}/findings`, `/api/schedule`, control de cola
    (add/pause/trigger), historial y tendencias MASVS (desde SQLite).
  - **WebSocket `/ws/run/{id}`** — streamea en vivo: razonamiento del agente (`print_agent_thinking` →
    evento), "Nutcracker says", cada `tool_call`, y salida Frida/logcat (ya emitida por
    `frida_capture.py`). Se logra publicando esos eventos a
    un bus (callback) que hoy va a `rich.Console`: añadir un sink opcional en aipwn que empuje al WS.
  - **WebSocket `/ws/scrcpy`** — vídeo del teléfono físico. Empaquetar **ws-scrcpy** (scrcpy sobre
    WebSocket/WebRTC) y proxyearlo; el frontend renderiza el stream H.264.
  - **Chat `/ws/chat`** — endpoint que inyecta mensajes del operador en la conversación del agente
    (los agentes ya son loops multi-turno con `self.messages`); permite pedir acciones ad-hoc
    ("toma screenshot", "prueba este hook") traducidas a tool calls.

### 3.2 Frontend (SPA ligera servida como estático por FastAPI)
Layout:
- **Panel vídeo scrcpy** (celular en vivo) a la izquierda.
- **Panel razonamiento del agente** + **logs Frida/logcat** al costado (dos pestañas o split).
- **Panel "Prompt"**: system prompt vigente + contexto del turno + últimas tool calls (transparencia del
  razonamiento).
- **Chat** con el agente (acciones).
- **Vista cola/scheduler**: runs en curso, encolados, próximos vencimientos, historial y score MASVS por app.
- Stack: SPA vanilla o Preact + WebSocket; todo servido local. Sin CDNs (self-contained).

### 3.3 Comando
- `nutcracker serve --dashboard` (o siempre-on) levanta API+WS+estáticos en `127.0.0.1:<port>`.
- Config `dashboard:` en `config.yaml` (`port`, `bind`, `enable_chat`, `scrcpy: true`).

### Fase 3 — estado: ✅ completa (2026-07-24), con 2 ajustes de alcance honestos

**Ajuste de arquitectura respecto al texto original:** en vez de un flag `--dashboard` sobre el
comando core `serve` (que obligaría a `nutcracker_core/cli/serve.py`, core, a conocer detalles de
un plugin — rompe la frontera), el dashboard es su **propio comando** registrado por el plugin:
`nutcracker dashboard`. Arranca su propia `QueueEngine` + `NutcrackerScheduler` (reusa las clases
de Fase 1 tal cual) además de la API — mismo resultado ("un comando levanta todo"), pero el core
sigue sin saber que el plugin existe. Mismo patrón que ya usa `aipwn` (comando propio, no un flag
de un comando core).

**Dos alcances deliberadamente reducidos frente al texto original** (documentados como tal, no
reclamados como hechos):
1. **Video del dispositivo:** el plan original pedía scrcpy embebido vía ws-scrcpy (WebSocket/WebRTC,
   H.264 en vivo). Este entorno **no tiene un dispositivo Android conectado** (`adb devices` vacío
   durante toda la sesión) y ws-scrcpy es un proyecto Node/TypeScript de varios miles de líneas —
   integrarlo sin poder probarlo contra hardware real habría sido reclamar un feature no verificado.
   Se implementó en su lugar `GET /api/device/screenshot` (captura real vía `adb exec-out screencap`)
   que el frontend refresca por polling cada 2.5s — no es video de baja latencia, pero es una vista
   *real y funcional* del dispositivo, con degradación honesta ("sin dispositivo") si no hay uno
   conectado (verificado: la POC real correctamente devolvió 503 al no haber ningún device).
2. **Razonamiento del agente en vivo / chat consumido por el agente:** el plan pedía streamear
   `print_agent_thinking`, cada `tool_call`, y que el chat inyectara mensajes en el loop del agente.
   El agente de bypass (`aipwn`) solo corre con un dispositivo físico conectado — inalcanzable aquí
   — y además **es un repositorio git separado** (`github.com/nutcracker-sh/aipwn`, ver nota de
   Fase 2), así que wire-up adicional ahí tampoco sería verificable en este entorno. Lo que sí se
   construyó y quedó **real y probado**: `GET /api/agent/prompt` (lee el `_SYSTEM_PROMPT` real de
   `aipwn.frida_agent` si el plugin está instalado — transparencia sin necesitar una corrida en vivo)
   y `/ws/chat/{package}` (mensaje real operador→WebSocket→bus→todos los suscriptores, con historial).
   El consumo automático de esos mensajes por un agente en ejecución queda como follow-up explícito.

   En cambio, el streaming de **logs en vivo de los jobs de la cola SÍ es real y de punta a punta**
   (no un sustituto reducido): se agregó un modo streaming opcional a `QueueEngine`
   (`engine.on_line`, usa `subprocess.Popen` en vez de `subprocess.run` solo cuando está asignado —
   el camino sin callback, usado por Fase 1/CLI/scheduler, queda exactamente igual) y el dashboard lo
   conecta a un `EventBus` en memoria que cada `/ws/jobs/{id}` consume. La POC real lo confirmó con
   **282 líneas de log reales** recibidas por WebSocket durante un análisis real del mismo APK de
   Mobile Hacking Lab usado en Fases 1-2.

**Entregado:**
- `nutcracker_core/queue/engine.py`: `on_line` callback opcional + `_run_streaming()` (Popen +
  lectura línea a línea). Default `None` preserva el camino de Fase 1 sin cambios (tests de Fase 1
  siguen monkeypatcheando `subprocess.run` sin tocar nada).
- `nutcracker_core/plugins/dashboard/`: `events.py` (EventBus pub-sub en memoria, con historial por
  canal), `store_reader.py` (queries de lectura con forma de dashboard sobre el store de Fase 0 —
  `list_apps`, `list_runs`, `run_detail`, `masvs_trend`, `summary_counts`), `device.py` (screenshot
  + listado de seriales vía adb), `api.py` (REST: `/api/summary`, `/api/apps`, `/api/apps/{pkg}/trend`,
  `/api/runs`, `/api/runs/{id}`, `/api/schedule` GET/POST, `/api/queue` GET/POST, `/api/queue/drain`,
  `/api/device`, `/api/device/screenshot`, `/api/agent/prompt`), `ws.py` (`/ws/jobs/{id}`,
  `/ws/chat/{package}`), `server.py` (`create_app()`), `__init__.py` (comando `nutcracker dashboard`),
  `static/index.html` (SPA self-contained: apps, cola, logs en vivo, dispositivo, prompt del agente,
  chat — sin CDN, dark/light aware).
- El encolado vía `/api/queue` (`run_now=True`) dispara `engine.drain()` en un hilo de fondo
  (no bloquea la respuesta HTTP — un job real puede tardar minutos) — el progreso se sigue por WS.

**Bug real encontrado y corregido (crítico, no de este plugin):** `.gitignore` tenía una regla
`plugins/` sin anclar que también atrapaba `nutcracker_core/plugins/` — el directorio del **sistema
de plugins del core**, no solo plugins externos. Cualquier plugin de primera parte nuevo (como este
dashboard) quedaba invisible para git por completo (`git add -A` lo saltea en silencio, sin error).
Confirmado que el usuario ya había comiteado Fases 1 y 2 en commits separados (`053b4e4 "fase 1"`,
`23052d0 "Fase 2"`) — de seguir el mismo patrón para Fase 3, todo este plugin se habría perdido
silenciosamente. Corregido reestructurando la regla a `nutcracker_core/plugins/*` +
allow-list explícito de los plugins de primera parte (`aireview`, `dashboard`), preservando el
comportamiento original para plugins externos (siguen ignorados por defecto; `aipwn` además queda
protegido por tener su propio `.git` anidado, con o sin esta regla). Un efecto secundario de la
negación (reincluía también `__pycache__/` dentro de esos plugins) se corrigió reforzando esa regla
al final del archivo. Verificado con `git check-ignore` y `git add -A --dry-run`: los 12 archivos
nuevos del plugin dashboard ahora sí quedan listos para `git add`.

**Tests:** 24 nuevos (`tests/dashboard/test_events.py` ×6, `test_api.py` ×10 vía FastAPI TestClient
sin red real, `test_ws.py` ×4 vía `websocket_connect`, más 2 en `tests/test_queue_engine.py` para el
streaming). Se encontró y corrigió un problema de aislamiento entre tests propio: un test que dispara
`run_now=True` dejaba un hilo de fondo corriendo tras terminar la función de test, contaminando el
monkeypatch de `subprocess.run` de un test posterior (flakiness dependiente de timing/orden) —
arreglado esperando explícitamente a que el hilo de fondo termine antes de que el test retorne. Suite
completa estable en 3 corridas consecutivas: **95/95 passing**.

**POC real ejecutada** (no simulada): `nutcracker dashboard --config ... --port 8765 --no-scheduler`
levantado como proceso real; un cliente Python (`httpx` + `websockets`) encoló el mismo APK real de
Mobile Hacking Lab vía `POST /api/queue`, se conectó a `/ws/jobs/{id}` *antes* de que terminara el
análisis, y recibió **282 líneas de log reales en vivo** (banner, tabla de vulnerabilidades, tabla
MASVS completa) seguidas del evento de estado final. Tras terminar: `GET /api/apps` mostró el
resultado real (`protected`, score 80, grade B, `next_due_at` auto-agendado a 30 días — Fase 1);
`GET /api/runs/{id}` mostró el finding `AUTH001` con **MASVS-STORAGE-2 + MASWE-0001 + CWE-532**
completos (Fase 2, fluyendo íntegro por la API del dashboard); `GET /api/device` y
`/api/device/screenshot` se degradaron correctamente a "sin dispositivo" (503); `GET /api/agent/prompt`
reportó honestamente `available: false` (el plugin aipwn no tiene sus dependencias — `any-llm-sdk` —
instaladas en este entorno, nunca se llegó a invocar `nutcracker aipwn <pkg>` de verdad, solo
`--help`, que no dispara el auto-install del plugin loader).

---

## Ideas transversales de mantenibilidad (a lo largo de las fases)
- **`orchestrator.py`** como única fuente de la verdad del pipeline (usado por CLI/daemon/dashboard) — evita
  la lógica duplicada que hoy vive en `nutcracker.py`.
- **Registry pattern** para checks y downloaders (auto-descubrimiento) igual que el loader de plugins ya
  probado — reduce el acoplamiento de las 3 listas manuales.
- **Modelo de dominio compartido** (`scan_types.py`) reusado por store, reporter y dashboard.
- **Tests + CI** desde Fase 0 para no regresionar durante los refactors.
- **Empaquetado**: añadir `pyproject.toml` (hoy corre como script suelto) para instalar `nutcracker` como
  paquete con entry-points (`nutcracker`, `nutcracker-serve`).

---

## Verificación por fase
- **Fase 0:** `pytest` verde; `nutcracker analyze <apk>` produce el mismo JSON/PDF que antes **y** una fila
  en `runs`/`findings` de SQLite (`sqlite3 nutcracker.db "select * from runs"`). CLI refactorizado pasa un
  smoke test de cada subcomando.
- **Fase 1:** con un `list_file` de 3 apps y `static_workers=2`, verificar en logs/DB que la fase estática
  corre en paralelo y la dinámica se serializa por device; `schedule set` fija `next_due_at`; adelantar reloj
  (o `--every 0d`) demuestra el re-encolado automático. `nutcracker serve` arranca y agenda.
- **Fase 2:** cada check nuevo tiene test determinista en `tests/checks/`; un APK de prueba con secreto
  conocido reporta `rule_id + MASVS + MASWE + CWE + MASTG`; `docs/owasp-mas-coverage.md` se genera desde el
  registry. Verdicto estático vs dinámico separado en el reporte.
- **Fase 3:** con el daemon activo, abrir `http://127.0.0.1:<port>`: se ve el celular vía scrcpy, el
  razonamiento y logs del agente en vivo durante un `aipwn`, el panel de prompt, el historial MASVS desde
  SQLite, y el chat inyecta una acción que el agente ejecuta en el device.

## Archivos/áreas críticas a tocar
- Nuevos (core): `nutcracker_core/store/`, `nutcracker_core/orchestrator.py`, `nutcracker_core/queue/`,
  `nutcracker_core/scheduler.py`, `nutcracker_core/checks/`, `nutcracker_core/cli/`.
- Modificar: `nutcracker.py` (→ shim/`cli/`), `vuln_scanner.py`
  (split), `masvs.py` (MASWE/CWE/MASTG), `config.py`
  (env vars), `reporter.py` + `pdf_reporter.py`
  (taxonomía + verdicto dual), `analyzer.py` (`aipwn_bypass_confirmed`).
- Nuevos (plugin): `nutcracker_core/plugins/dashboard/` (FastAPI + WS + SPA + ws-scrcpy). Sink de streaming
  opcional en `plugins/aipwn/frida_agent.py` / `frida_capture.py`.
