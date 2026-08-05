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
- **`batch` migrado al motor de cola** (2026-07-25, ver detalle más abajo — cerraba el único
  ítem pendiente de Fase 1). Ya no es secuencial/in-process: corre en paralelo según
  `queue.static_workers`, con resumen consolidado reconstruido desde SQLite.

### `batch` migrado al motor de cola (2026-07-25)

Cerrado el único pendiente de Fase 1. `cli/batch.py` ya no duplica su propio bucle de
descarga+análisis in-process (`APKAnalyzer` + `_post_analysis_flow` directos, con su propio
`results_summary` armado en memoria) — ahora cada target pasa por el mismo
`QueueEngine.submit()`/`drain()` que usan `queue add` y `serve`: subproceso aislado
(`analyze`/`scan`), paralelo real según `queue.static_workers` (default 4) o estrictamente
secuencial con `static_workers: 1`.

- **Resumen reconstruido desde SQLite**, no desde objetos en memoria: nueva función
  `_summary_for_outcome()` en `cli/batch.py` — como cada job corrió en su propio subproceso, no
  hay `ScanResult` en memoria para armar severidades/top-findings/categorías; se leen de
  `repository.findings_for_run()` usando el `run_id` que `JobOutcome` ya trae enlazado (Fase 1
  original). El PDF consolidado (`generate_batch_report`) no cambió — sigue recibiendo la misma
  forma de `dict` de siempre.
- **`--stop-on-error` solo se garantiza estrictamente en modo secuencial**
  (`static_workers: 1`): ahí se encola y drena un target a la vez, cortando antes del
  siguiente. En paralelo no hay forma de cancelar jobs que el pool ya empezó (QueueEngine no
  soporta cancelación a mitad de camino) — documentado en el `--help` del comando.
  `--keep-apk` se eliminó del CLI de `batch`: `build_job_cmd` ya fuerza `--keep-apk` en todo
  job de tipo `scan` (decisión ya tomada en Fase 1 para depuración), así que el flag propio de
  `batch` había quedado inalcanzable de todas formas.
- **Efecto colateral bienvenido**: cada app que pasa por `batch` ahora también queda
  auto-agendada para revisión periódica (Fase 1.2) — confirmado en la prueba real
  (`next_due_at` se fijó a 30 días).
- **Bug preexistente encontrado y arreglado al validar con hardware real (no relacionado con
  esta migración)**: `generate_batch_report()` crasheaba con `FPDFUnicodeEncodingException` al
  renderizar el título "Comparative Table — Protection Status" — el em-dash (`—`) no es
  soportable por la fuente Helvetica no-Unicode que usa fpdf2 para `section_title()`. Bug
  presente desde antes (mismo string, mismo problema con el `batch` viejo), simplemente nunca
  se había ejercitado con >1 resultado exitoso en un test real. Fix: reemplazado por un guion
  simple en `i18n.py` (EN + ES) — único string con em-dash de los 43 que existen en el archivo
  que efectivamente se usa en `pdf_reporter.py`. Test de regresión:
  `tests/test_pdf_reporter_batch.py`.
- **Validado con APKs reales contra el dispositivo físico** (no solo con la QueueEngine
  mockeada de los tests): 3 targets (2 válidos + 1 corrupto a propósito) en paralelo
  (`static_workers=3`) → resumen correcto (2 OK, 1 error con el mensaje real "End of central
  directory record (EOCD) signature not found"), PDF consolidado generado; y en secuencial
  (`static_workers=1`) con `--stop-on-error` → se detuvo correctamente antes del tercer target
  tras el error en el segundo.

**Tests:** 9 nuevos (`tests/test_batch.py` — reconstrucción de resumen desde SQLite con
severidades/leaks/categorías/mapeo de verdicto, más el flujo del comando vía `CliRunner` con
una `QueueEngine` falsa, incluida la detención en modo secuencial) + 1
(`tests/test_pdf_reporter_batch.py`, regresión del em-dash). Suite completa: **119/119
passing**.

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

### Cierre de cobertura OWASP MAS: 14/24 → 18/24 (2026-07-25)

De los 10 controles sin ningún check, se cerraron 4 con checks reales y de alta confianza — el
resto se dejó **honestamente sin cobertura** en vez de forzar checks de baja confianza solo para
subir el número (mismo criterio de prudencia que el resto de Fase 2).

- **`MASVS-CODE-1` (versión de plataforma actualizada) — "victoria gratis"**: el check ya
  existía (`manifest_analyzer.py` detecta `targetSdkVersion` bajo y ya estaba mapeado a CODE-1
  en `MISCONFIG_TO_MASVS` desde la propia Fase 2), pero no aparecía en el registry/coverage doc
  porque el adaptador solo envolvía `vuln_scanner`/`native_scanner`/detectores. Nueva función
  `_load_manifest_analyzer_checks()` en `checks/static/adapter.py` lo representa con un id
  sintético `MANIFEST-LOW-TARGET-SDK` (no es un `VulnRule` real — no aparece en hallazgos
  persistidos, esos `Misconfiguration` no pasan por `store/hooks.py` hoy, mismo gap ya
  documentado arriba).
- **`MASVS-CRYPTO-2` (gestión de claves)** — `CRYPTO007`: detecta `SecretKeySpec` construido
  directamente desde los bytes de un password/PIN (`getBytes()`), sin una función de derivación
  de claves. Patrón real y específico (exige que el nombre de variable contenga
  password/passwd/pin/secret), bajo riesgo de falso positivo. `MASWE-0010` (Improper
  Cryptographic Key Derivation), `CWE-916`.
- **`MASVS-AUTH-2` (autenticación local)** — `AUTH002`: detecta uso de
  `android.hardware.fingerprint.FingerprintManager`, deprecada desde Android 9/API 28 en favor
  de `androidx.biometric.BiometricPrompt`. Nombre de clase completo, sin ambigüedad. `MASWE-0032`
  (Platform-provided Authentication APIs Not Used), `CWE-477`.
- **`MASVS-PRIVACY-2` (prevenir identificación del usuario)** — `PRIVACY001`: detecta lectura de
  identificadores de hardware persistentes (`getImei`/`getDeviceId`/`getMeid`/
  `getSimSerialNumber`/`getSubscriberId`) usados típicamente para tracking entre instalaciones.
  `MASWE-0110` (Use of Unique Identifiers for User Tracking), `CWE-359`.

**Quedan sin cobertura, a propósito** (no son verificables de forma determinista analizando solo
el APK — necesitarían tráfico en vivo, backend, o entendimiento de lógica de negocio/UX):
`MASVS-AUTH-1` (seguridad de protocolos remotos de auth), `MASVS-AUTH-3` (autenticación adicional
para operaciones sensibles — requiere saber qué es "sensible" en la lógica de la app),
`MASVS-CODE-2` (mecanismo de actualización forzada — comportamiento servidor/backend, no
verificable desde el APK), `MASVS-CODE-3` (SCA — componentes sin vulnerabilidades conocidas;
requeriría una base de datos de CVEs real, no una lista de patrones regex; queda como follow-up
explícito, no forzado con datos inventados), `MASVS-PRIVACY-3`/`MASVS-PRIVACY-4` (transparencia y
control del usuario sobre sus datos — políticas/UX, no analizables estáticamente con confianza).

Se descartó deliberadamente un candidato de baja confianza: detectar `KeyGenerator`/
`KeyPairGenerator` sin proveedor `AndroidKeyStore` explícito — riesgo real de falso positivo alto
(muchas apps generan claves simétricas efímeras de un solo uso legítimamente sin necesitar
respaldo de hardware), se prefirió no incluirlo antes que meter ruido.

**Tests:** 8 nuevos (`tests/test_vuln_scanner_split.py` ×6: positivo/negativo por cada regla
nueva; `tests/checks/test_registry.py` actualizado para el nuevo id sintético). Suite completa:
**125/125 passing**. `docs/owasp-mas-coverage.md` regenerado: **18/24 controles**, 68 checks
totales (66 estáticos + 2 dinámicos), 33/119 debilidades MASWE referenciadas.

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

**Nota de `.gitignore` (encontrada, no corregida a pedido explícito del usuario):** `.gitignore`
tiene una regla `plugins/` sin anclar que también atrapa `nutcracker_core/plugins/` — el directorio
del **sistema de plugins del core**, no solo plugins externos. Cualquier plugin de primera parte
nuevo (como este dashboard) queda invisible para git por completo (`git add -A` lo saltea en
silencio, sin error) — confirmado con `git check-ignore -v`. Se probó y verificó un fix (reestructurar
la regla a `nutcracker_core/plugins/*` + allow-list explícito de `aireview`/`dashboard`), pero el
usuario pidió explícitamente no tocar `.gitignore` y dejar el plugin donde está — **decisión
consciente, no un problema pendiente**: el dashboard queda intencionalmente fuera del tracking de
git de este repo, igual que ya trataba a `aipwn`. Esto no afecta en nada la funcionalidad (gitignore
no tiene efecto en tiempo de ejecución) — reconfirmado corriendo la suite completa después de revertir
el intento de fix: **95/95 passing**. Si en el futuro se quiere trackear el dashboard, el fix ya
probado es: `nutcracker_core/plugins/*` + `!nutcracker_core/plugins/dashboard/` +
`!nutcracker_core/plugins/dashboard/**` (cuidado: esa negación reincluye también `__pycache__/`
dentro del plugin — hay que reforzar `**/__pycache__/` después en el archivo).

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

---

## Sesión de pruebas con dispositivo físico real (2026-07-24)

Prueba real contra un Moto G23 (Android 14) rooteado con Magisk, con módulos de anti-detección
avanzados instalados (`hma_oss_zygisk`, `playintegrityfix`, `tricky_store`, `specter`,
`zygisksu`) — un banco de pruebas realista pensado específicamente para evadir detectores de
root/RASP. Apps de prueba usadas: `com.example.tapjacking` (sin protección) y `owasp.sat.agoat`
(AndroGoat, la app oficial de OWASP para práctica de seguridad móvil — protegida). Se evitó
deliberadamente tocar las apps bancarias reales instaladas en el device (BCP, Yape, Tenpo).

### Hallazgos y fixes aplicados

**🔴 Bug crítico de core (preexistente, no introducido en este plan) — pérdida total de
resultados con `--launch`.** `orchestrator._launch_frida_bypass()` usaba `os.execvp()`, que
**reemplaza el proceso Python actual** por el de `frida` (documentado en el propio código:
"reemplaza el proceso"). Como consecuencia, cualquier `analyze --launch` / `scan --launch`
contra una app protegida perdía el análisis estático completo — sin JSON, sin PDF, sin fila en
`runs`/`findings` de SQLite — **sin importar si Frida lograba conectar o no**, porque todo el
código de persistencia (`save_analysis_json`, PDF, reporte MASVS, post-hooks) vive *después* de
esa llamada en `_run_analysis()`, y `execvp` nunca retorna. Reproducido en vivo con AndroGoat:
el análisis estático corrió completo (banner PROTECTED, tabla de hallazgos con certificate
pinning real) pero `reports/owasp.sat.agoat/` no llegó a existir.
- **Fix:** `os.execvp(frida_cmd[0], frida_cmd)` → `subprocess.run(frida_cmd)`
  (`orchestrator.py`). Preserva el uso interactivo original (hereda stdin/stdout/stderr, un
  humano en terminal sigue viendo el REPL de frida en vivo) pero retorna control al llamador
  cuando frida termina, permitiendo que el resto del pipeline guarde sus resultados.
- **Validado en vivo tras el fix:** mismo AndroGoat, mismo dispositivo → `status: done`,
  `package: "owasp.sat.agoat"`, `run_id: 3`, con `reports/owasp.sat.agoat/` conteniendo JSON +
  PDF + osint.json + vuln.json, y hallazgos reales persistidos con MASVS+MASWE+CWE completos
  (p.ej. `HC005 Hardcoded AWS credentials` → `MASVS-STORAGE-2` / `MASWE-0005` / `CWE-798`,
  encontrado de verdad en `CloudServicesActivity.java` de AndroGoat).

**🔴 Bug de diseño en Fase 1 — jobs dinámicos de la cola dependían de un REPL interactivo.**
`QueueEngine`/`build_job_cmd` usaban `--launch` para los jobs `kind="dynamic"` — un flag pensado
para ceder el control a un humano en una sesión interactiva de Frida, incompatible con
automatización headless (un job de cola sin humano delante quedaría esperando un REPL que nunca
recibe input).
- **Fix:** nuevo flag `--dynamic-checks` en `analyze` (core, `cli/analyze.py`) que corre los
  checks dinámicos de Fase 2 (`checks/dynamic/` — ADB puro, sin Frida, sin REPL) contra
  `--serial` tras el análisis estático, vía la función nueva
  `orchestrator._run_dynamic_checks_for()`. `build_job_cmd` renombró su parámetro `launch` →
  `dynamic_checks`; los jobs dinámicos de la cola ya no invocan `--launch` en absoluto.
- **Validado en vivo:** el mismo re-run de AndroGoat mostró en el log en vivo del dashboard:
  `DYN-DEBUGGABLE` **detectado** (`run-as` funcionó — AndroGoat resultó ser debuggable en su
  instalación real en el device, un hallazgo genuino) y `DYN-CLEARTEXT-TRAFFIC` no detectado
  (app no lanzada activamente durante la ventana del check) — ambos corrieron sin colgarse.

**🟡 Menor — `default_adb_runner` descartaba `stderr`.** Comandos como `run-as` escriben su
mensaje real de error a stderr (p.ej. `"run-as: package not debuggable: <pkg>"`); capturar solo
`stdout` lo perdía en silencio, dejando el campo `detail` de los checks dinámicos sin
diagnóstico útil. **Fix:** `stderr=subprocess.STDOUT` en `checks/dynamic/context.py`. Validado:
tras el fix, `DYN-DEBUGGABLE` mostró el detalle real (`"run-as funcionó; JDWP no confirma"`) en
vez de un string vacío.

**🟡 Menor — `QueueEngine` truncaba a ciegas el campo `error`.** `error = output[-2000:]` perdía
la causa raíz real cuando el proceso seguía imprimiendo (tablas de hallazgos, banners) después
del error — se vio en vivo con `"Failed to spawn: ... No route to host"` tapado por una tabla de
vulnerabilidades posterior, dejando en `error` solo un fragmento ilegible de esa tabla. **Fix:**
`_extract_error_summary()` nueva en `queue/engine.py` — prioriza líneas con marcadores de error
conocidos (`error`, `failed`, `no route to host`, `timeout`, `refused`, ...) sobre el tail ciego.

### Confirmado funcionando correctamente contra hardware real (sin cambios necesarios)
- Detección de protecciones (AndroGoat → `PROTECTED`, MASVS score 70/grade C).
- Cola con device-lock real (Fase 1) — job dinámico sobre `tapjacking.apk`, sin protección,
  corrió limpio de punta a punta antes de tocar ningún código.
- Dashboard completo (Fase 3): API REST, WebSocket de logs en vivo (282+ líneas reales
  streameadas durante los distintos runs), y **`/api/device/screenshot` capturando la pantalla
  real del device** (confirmado con una imagen PNG real de 720×1600, no simulada).
- Fallback automático existente en el código: cuando USB no está disponible, nutcracker ya
  intenta conectar a frida-server vía la IP WiFi del propio device (`frida -H <ip>:27042`) — un
  mecanismo inteligente preexistente que solo no funcionó por la limitación de red del entorno
  de prueba (ver abajo), no por un bug de nutcracker.

### Corrección de diagnóstico: no era un bloqueo de red WSL↔Windows
El primer diagnóstico de esta sesión (Frida no conectaba: "adb.exe de Windows vs WSL, dos stacks
de red separados, necesita `wsl --shutdown` + mirrored mode o `usbipd-win`") **resultó
incorrecto** — quedaba documentado así más arriba antes de esta corrección. La causa real, mucho
más simple, era una combinación de dos cosas:
1. **IP equivocada**: `config.yaml` ya tenía `frida_host: '192.168.1.4:27042'` de una sesión
   anterior — pero la IP real del teléfono (confirmada con la propia captura de pantalla del
   dashboard, sección "Detalles de la red") era `192.168.1.42`. `.4` simplemente no era un host
   vivo en esa LAN → "No route to host".
2. **frida-server sin `-l 0.0.0.0`**: el binario, iniciado sin ese flag, solo escuchaba en
   `127.0.0.1` — inalcanzable por TCP desde cualquier otro host, WSL incluido.

Con la IP corregida en `config.yaml` y frida-server reiniciado con `-l 0.0.0.0` (vía el fix de
`_launch_frida_bypass()`/`setup_frida_server` de más abajo), la conexión funcionó **directamente
desde WSL, sin `wsl --shutdown` ni cambios de `networkingMode`** — confirmado con
`frida-ps -H 192.168.1.42:27042` listando el proceso real del device. La guía de mirrored
mode/usbipd-win de más abajo queda como referencia general (útil si algún día SÍ hay un problema
de ruteo real), pero no era necesaria en este caso — buena lección: verificar la IP y el bind
antes de asumir un problema de red más profundo.

**🔴 Bug adicional encontrado al preparar la conexión por red — `--launch` nunca escuchaba en
red.** Al revisar `config.yaml` con `strategies.frida_host`/`frida_server_version` recién
configurados, se encontró que `_launch_frida_bypass()` reiniciaba frida-server con un simple
`nohup .../frida-server &` — **sin el flag `-l 0.0.0.0`**, así que el server solo escuchaba en
`127.0.0.1` sin importar qué tan bien rutee la red. Además ignoraba por completo
`frida_server_version` (riesgo de mismatch con la librería Python) y no hacía el
`unset LD_PRELOAD` que sí hace `setup_frida_server()` — necesario específicamente en devices con
módulos Magisk Zygisk/LSPosed (`hma_oss_zygisk`, `zygisksu`, exactamente los que tiene el
dispositivo de prueba), que rompen el attach/spawn de Frida si no se limpia esa variable.
- **Fix:** `_launch_frida_bypass()` ahora reusa `setup_frida_server()` (el mismo mecanismo que ya
  usa el flujo FART en `pipeline.py`) cuando se pasa `--serial`: descarga/sincroniza la versión de
  `frida_server_version`, detecta arquitectura del device, y pasa `listen_all=bool(frida_host)`.
  Sin `--serial` (un único device/emulador, sin forma fiable de resolver arquitectura) se conserva
  el restart simple de antes como fallback, para no dejar de reiniciar frida-server en absoluto.
  Un fallo al reiniciar (red caída, etc.) se loggea como warning y no aborta el intento de launch.

### Tests añadidos (34 nuevos, 110/110 en la suite completa)
`tests/test_orchestrator.py` (9 tests: `execvp`→`subprocess.run`, uso de `setup_frida_server` con
`listen_all`/`frida_server_version` cuando hay `--serial`, fallback al restart simple sin
`--serial`, resiliencia si `setup_frida_server` lanza una excepción, `build_job_cmd` con
`dynamic_checks`, `_run_dynamic_checks_for` incluyendo manejo de checks que fallan),
`tests/test_queue_engine.py` (+4: `_extract_error_summary`), `tests/checks/test_dynamic_checks.py`
(+2: `default_adb_runner` combina stderr / retorna vacío si el binario no existe). Todos
deterministas — no requieren un dispositivo conectado para correr en CI.

## Mejora del dashboard: vista de detalle por app (2026-07-26)

El dashboard de Fase 3 mostraba únicamente una tabla plana de apps (package, verdict, score,
próxima revisión) — toda la taxonomía OWASP MAS construida en Fase 2 (MASVS/MASWE/CWE por
hallazgo) y el historial de runs en SQLite existían en el backend pero nunca se exponían en la
UI. No era un bug: la Fase 3 se había enfocado en logs en vivo + cola, y la Fase 2 llegó después
sin volver a tocar el frontend. Se cerró ese hueco.

- **Modal de detalle por app** (`nutcracker_core/plugins/dashboard/static/index.html`, sin
  cambios de backend — `/api/runs/{id}`, `/api/apps/{pkg}/trend` y `/api/schedule` ya existían):
  clic en cualquier fila de la tabla de apps abre un modal con:
  - **KPIs** del último run: veredicto, score MASVS + grade, próxima revisión.
  - **Gráfico de tendencia MASVS** (SVG propio, sin librerías externas): score por run a lo
    largo del tiempo, un punto por run coloreado según el grade (A/B verde, C ámbar, D naranja,
    F rojo), gridlines en 0/25/50/75/100, tooltip al hacer hover sobre cada punto
    (`fecha — Score N (grade)`), labels de fecha en los extremos.
  - **Editor de revisión periódica** inline (días entre revisiones + botón Guardar → `POST
    /api/schedule/{package}`), en vez de requerir la CLI para reagendar una app puntual.
  - **Controles MASVS afectados**: hallazgos agrupados por control MASVS, con la severidad más
    alta de cada grupo y el conteo.
  - **Tabla de hallazgos** del último run: regla, severidad (badge con ícono + color, nunca solo
    color), MASVS, MASWE, CWE, ubicación (archivo:línea).
- **Paleta de estado**: colores fijos validados vía el skill `dataviz` (good `#0ca30c`, warning
  `#fab219`, serious `#ec835a`, critical `#d03b3b`) — nunca reciclados como colores categóricos,
  siempre acompañados de texto/ícono, nunca el único portador del significado.
- **Verificación visual real**: se sembraron datos de ejemplo (`com.example.bankapp`, 5 runs con
  progresión de score 45→80, 4 hallazgos con metadata MASVS/MASWE/CWE completa) en una copia
  local de `nutcracker.db` (gitignorado, jamás commiteado), se levantó el dashboard
  (`nutcracker.py dashboard --port 8765`) y se capturaron screenshots reales con Playwright
  (`chromium-headless-shell`) de: la tabla principal, el modal abierto, y el tooltip del gráfico
  en hover. Cero errores de JS/consola; layout, colores y geometría del gráfico verificados
  visualmente sin colisiones ni overflow. La base de datos de prueba se descartó después
  (`rm nutcracker.db*`) para no dejar datos sintéticos en el entorno del usuario.
- **Sin cambios de backend**: toda la funcionalidad se apoyó en endpoints ya existentes de
  Fase 3 — confirma que el diseño API-first de esa fase era suficiente para una UI más rica sin
  tocar `server.py`.
- Suite completa: **125/125 tests pasan** (sin tests nuevos — cambio puramente de frontend, sin
  lógica Python nueva que testear; la verificación fue visual vía Playwright, no unitaria).

## Fix: panel "Logs en vivo" mostraba basura coloreada + layout no responsivo (2026-07-26)

El usuario reportó "la web no es responsive" adjuntando dos screenshots: una del panel
"Dispositivo" (bien) y una del panel "Logs en vivo" mostrando el banner ASCII-art de nutcracker
como un bloque ilegible de píxeles de colores en vez de texto.

**🔴 Causa raíz del bloque de colores — códigos ANSI truecolor crudos volcados al navegador.**
`QueueEngine._run_job()` construye el entorno del subproceso con `env = dict(os.environ)`, sin
tocar nada relacionado a color. Si el proceso que arrancó el dashboard heredó `FORCE_COLOR`
(algunas terminales/integraciones, p.ej. la terminal integrada de VS Code, lo exportan), rich
detecta `is_terminal=True` **incluso cuando stdout es un pipe no interactivo** (confirmado
directamente: `Console().is_terminal` da `True` con solo `FORCE_COLOR=1` en el entorno, sin
importar `isatty()`) y emite ANSI truecolor real. Ese texto crudo (códigos de escape +
caracteres de medio-bloque `▄`/`▀` del logo pixel-art) se streamea línea por línea al WebSocket
y se inserta tal cual en `<pre id="log">` — de ahí el bloque de "píxeles" de colores.
- **Fix:** `_run_job()` ahora fuerza `env["NO_COLOR"] = "1"` y elimina `FORCE_COLOR` del entorno
  del subproceso — confirmado que `NO_COLOR` manda por encima de `FORCE_COLOR` en rich (con
  ambos presentes, 0 códigos de color, solo quedan 2 secuencias OSC8 de hipervínculo inofensivas
  e invisibles en el navegador). Defensa adicional: nueva `_strip_ansi()` en `queue/engine.py`
  aplicada a cada línea antes de publicarla via `on_line` (streaming) y al resumen de error
  (`_extract_error_summary`), por si alguna herramienta de terceros (semgrep, etc.) ignora
  `NO_COLOR` y fuerza color con sus propios flags.
- **Validado en vivo:** se reprodujo el bug exacto (servidor arrancado con
  `FORCE_COLOR=1 COLORTERM=truecolor` forzados) y se confirmó que, tras el fix, el mismo banner
  se ve en texto monocromo limpio y legible en el panel de logs — captura real vía Playwright.

**🟡 Menor — layout no protegido contra overflow horizontal en viewports angostos.**
`index.html` (dashboard) no envolvía las tablas de "Apps" y "Cola de análisis" en un contenedor
con scroll horizontal propio, así que en pantallas angostas (móvil) el ancho de la tabla podía
forzar scroll horizontal de toda la página. El header (`.stats`) tampoco hacía wrap. La imagen
`#shot` (captura del device) combinaba `width:100%` (CSS) con `height="360"` (atributo HTML fijo),
lo que podía distorsionar el aspect ratio de una captura de teléfono real (720×1600) en vez de
escalarla proporcionalmente.
- **Fix:** nueva clase `.table-scroll{overflow-x:auto}` envolviendo ambas tablas; `header`/`.stats`
  con `flex-wrap:wrap`; nuevo breakpoint `@media (max-width:600px)` con paddings/gaps más
  compactos; `#shot` cambiado a `max-height:70vh;object-fit:contain` sin altura fija, para que la
  captura del device escale manteniendo su proporción real.
- **Validado:** Playwright a 375×812 (viewport móvil real) — `document.documentElement.scrollWidth
  - clientWidth === 0` (cero overflow horizontal), sin errores de JS, captura real revisada
  visualmente.

**⚠️ Nota operacional durante la verificación:** para reproducir el bug con `FORCE_COLOR` forzado
se levantó un dashboard de prueba en el puerto 8766 apuntando al `nutcracker.db` real del
repositorio (mismo `cwd`) — se descubrió ahí un job real preexistente (`#1,
pe.indigital.tunki.user, running`, el mismo que aparece en la captura que envió el usuario), no
generado por esta sesión. Se detuvo el servidor de prueba de inmediato y se borraron **solo** las
filas insertadas por la prueba (`apps`/`schedule`/`queue_jobs` para `com.example.bankapp`),
dejando el job real #1 completamente intacto sin tocarlo.

Tests añadidos (5, en `tests/test_queue_engine.py`): `_strip_ansi` (colores, hipervínculos OSC8,
no-op en texto plano), `_run_job` fuerza `NO_COLOR`/limpia `FORCE_COLOR` en el entorno del
subproceso, `_run_streaming` limpia ANSI de cada línea antes de publicarla. **130/130 tests
pasan.**

## Documentación + empaquetado (2026-07-26)

Ítem de mantenibilidad transversal, pendiente desde Fase 0 (ver huecos abiertos al inicio de este
documento): `README.md` no mencionaba en absoluto la cola, el scheduler, el dashboard, ni la
taxonomía MASWE/CWE — todo el trabajo de las Fases 1-3 era invisible para quien solo lee el
README. Tampoco existía `pyproject.toml`.

- **`README.md`**: nuevas secciones "Mass Execution: Queue & Scheduler" (`queue add`/`queue
  ls`/`schedule set`/`schedule ls`/`serve`, con ejemplos reales verificados contra el `--help`
  actual de cada comando), "OWASP MAS Alignment (MASVS + MASWE + CWE)" (cómo se genera
  `docs/owasp-mas-coverage.md`, cobertura real 18/24 controles honestamente indicada, no
  aspiracional), y "Web Dashboard" (`nutcracker dashboard`, qué muestra cada panel, límites
  conocidos ya documentados en Fase 3). Tabla de plugins actualizada con `dashboard`. Árbol de
  "Project Structure" reescrito para reflejar el estado real (`cli/`, `store/`, `queue/`,
  `checks/`, `orchestrator.py`, `scheduler.py`, `plugins/dashboard/` — antes solo listaba los
  módulos de Fase -1). Bloque de ejemplo de `config.yaml` ampliado con `store:`/`queue:`/
  `scheduler:`/`dashboard:`.
- **`config.yaml.example`**: nuevo bloque `dashboard: {bind, port}` — existía en el código
  (`cfg_get(config, "dashboard", "bind"/"port")`) pero no estaba documentado en ningún lado.
- **`ROADMAP.md`**: marcados como hechos (con fecha y referencia a la sección de `plan.md`
  correspondiente) dos ítems que ya estaban completos — "Split `vuln_scanner.py`" (Fase 0.3) y,
  parcialmente (`[~]`), "Differentiate runtime bypass vs DEX extraction" (los pasos 1-2 de 4 están
  hechos desde Fase 2.4; los pasos 3-4, banner en `reporter.py` y sección nueva en el PDF, siguen
  pendientes — documentado así, no reclamado como completo). Añadidos como ítems nuevos: scrcpy
  real y wiring del chat/razonamiento de `aipwn` en el dashboard (los 2 recortes de alcance
  honestos ya documentados en Fase 3, ahora también visibles en el roadmap de cara afuera).
- **`pyproject.toml`** (nuevo): empaqueta `nutcracker_core` (auto-descubrimiento de subpaquetes)
  con entry point `nutcracker = nutcracker_core.cli:cli` y extra opcional `[dashboard]`
  (fastapi/uvicorn). `python nutcracker.py <cmd>` en modo script sigue funcionando exactamente
  igual — el paquete es un modo de instalación adicional, no un reemplazo. Validado con
  `pip install -e .` real: `nutcracker --help`/`--version` funcionan como comando instalado
  (`.venv/bin/nutcracker`), y `pip install -e ".[dashboard]"` instala fastapi/uvicorn
  correctamente. `nutcracker.egg-info/` (artefacto de build, no cubierto por `.gitignore`) se
  borró manualmente tras cada verificación en vez de tocar `.gitignore`.
- **Fix menor encontrado de paso**: `@click.version_option("0.1.0", ...)` en `cli/__init__.py`
  tenía la versión hardcodeada y desincronizada de `nutcracker_core.__version__` (`0.2.0`, la que
  sí se usa correctamente en el banner). Cambiado a `@click.version_option(_VERSION, ...)`.
  `nutcracker --version` ahora reporta `0.2.0` en vez de `0.1.0`.

Sin cambios de lógica de negocio — puramente documentación + empaquetado. **130/130 tests
pasan** (sin tests nuevos: nada de esto tiene comportamiento en tiempo de ejecución que testear
más allá de lo que ya cubre `test_queue_engine.py`/CLI existente, verificado manualmente con
`pip install -e .` real en vez de con un test unitario).

## Cierre de los 2 recortes de alcance de Fase 3: scrcpy real + wiring de aipwn (2026-07-27)

El usuario pidió implementar los dos últimos ítems documentados como "alcance reducido" en Fase
3. Ambos se completaron y **se validaron en vivo contra el Moto G23 real** (dispositivo rooteado
con Magisk usado en la sesión de pruebas del 2026-07-24), no solo con mocks.

### scrcpy real (video en vivo del dispositivo)

**Investigación previa (documentada por transparencia, no descartada sin más):** se decompiló con
`jadx` el `scrcpy-server.jar` oficial de Genymobile en 3 versiones (v1.25 vía `apt-get download`
sin root; v2.4 y v3.1 vía descarga directa de GitHub Releases) para entender el wire protocol
exacto (socket abstracto `scrcpy`/`scrcpy_<scid>`, header de 68 bytes de device-meta, header de 12
bytes de frame-meta, modo `raw_video_stream`/flags individuales sin ningún header). Con el
protocolo ya verificado por lectura de fuente (no adivinado), una reimplementación propia en
Python (`socket` + `PyAV` para decodificar H.264) **se probó exhaustivamente contra el device real
y nunca logró completar el handshake** — el servidor se queda colgado indefinidamente dentro de
`Device()` (antes de aceptar la conexión), sin excepción ni AVC denial visible en `logcat` extenso,
reproducido igual con 3 versiones distintas del server. Se descartó como enfoque.

**Lo que sí funcionó, y es la implementación final:** el usuario indicó usar su propio scrcpy 3.1
de Windows (`scrcpy-win64-v3.1`), accedido desde WSL vía interop (`/mnt/c/...scrcpy.exe`,
ejecutable directamente como ".exe" gracias al interop de WSL2). El binario **real** sí completa el
handshake contra el mismo device sin problema — confirma que el cuelgue era específico de mi
reimplementación del protocolo (probablemente algún detalle fino del handshake del cliente real que
no se pudo reproducir en el tiempo disponible), no una limitación irresoluble del hardware/Magisk.

Como `scrcpy` no ofrece un modo de salida por stdout/pipe (solo GUI o `--record=<file>`, y `--record`
requiere un archivo real, no `-` /stdout), la solución final es:
- `nutcracker_core/plugins/dashboard/scrcpy_video.py` (nuevo): lanza
  `scrcpy --no-window --record=<mkv temporal>` sin ventana; un hilo de fondo relee ese archivo
  periódicamente con **PyAV** (nueva dependencia del plugin dashboard, `av>=11.0`) y se queda con el
  último frame decodificado como JPEG — **confirmado empíricamente que el .mkv crece de forma
  incremental en disco mientras scrcpy sigue grabando** (con actividad real de pantalla; con
  pantalla estática/bloqueada no se generan frames nuevos, por el optimize de "repeat-previous-frame"
  de scrcpy — comportamiento esperado, no un bug). La grabación se reinicia cada 20s a un archivo
  nuevo para acotar el costo de releer un archivo cada vez más grande.
- **Bug real encontrado y arreglado durante la validación en vivo**: matar el proceso `adb
  shell`/`scrcpy` LOCAL (`SIGKILL`, o incluso `SIGTERM` a través de la interop WSL↔Windows) **no
  siempre mata el `app_process` REMOTO en el device** — queda escuchando el socket "scrcpy" para
  siempre, y cada intento posterior falla con "Address already in use" (server) / "Server
  connection failed" (cliente). Fix: `ScrcpyVideoSession._kill_orphaned_remote_server()` corre
  `adb shell pkill -f com.genymobile.scrcpy.Server` (best-effort) antes del primer ciclo y tras
  cada ciclo fallido.
- `GET /api/device/video` (nuevo, `api.py`): `StreamingResponse` multipart/x-mixed-replace — el
  navegador lo reproduce nativo en un `<img>`, sin JS ni librerías adicionales. `GET
  /api/device/video/status` para que el frontend decida sin arrancar nada. 503 + degradación
  honesta si no hay binario configurado.
- `config.yaml` (`dashboard.scrcpy_path`): ruta explícita del usuario a su propia instalación de
  scrcpy — nutcracker **no vendorea el binario** (¿por qué reimplementar/empaquetar un proyecto de
  terceros cuando el usuario ya lo tiene?). Vacío = busca `scrcpy`/`scrcpy.exe` en PATH.
- Frontend (`index.html`): pestaña "Dispositivo" ahora intenta video en vivo primero
  (`toggleDeviceView()`), con fallback automático y transparente al polling de screenshots
  existente si no hay binario configurado o si el stream falla a mitad de sesión (`onerror` del
  `<img>`).
- **Validado en vivo con Playwright**: captura real mostrando la pantalla actual del device
  (pantalla "Detalles de la red" con la IP real `192.168.1.42`) dentro del panel del dashboard, nota
  confirmando "● Video en vivo real (scrcpy) — no es polling", cero errores JS.
- Tests: `tests/dashboard/test_scrcpy_video.py` (16, todo mockeado — subprocess/PyAV/tiempo — no
  depende de hardware ni de scrcpy real instalado) + 3 en `test_api.py` para los endpoints nuevos.

### Wiring del agente aipwn en el dashboard

**Streaming del razonamiento en vivo:** en vez de reimplementar un mecanismo de streaming nuevo
para aipwn, se le dio a `aipwn` un **kind de job nuevo en la cola** (`"aipwn"`, junto a
`"static"`/`"dynamic"`) que reusa el streaming de logs ya construido en Fase 3 para
`queue add`/`batch` — `orchestrator.build_job_cmd(..., aipwn=True)` genera
`nutcracker aipwn <package> [--serial <serial>]` como subproceso aislado (igual que todo lo demás
en la cola), y su stdout (que ya incluye "Nutcracker thinking", "Nutcracker says", cada tool call)
se streamea línea por línea al mismo `/ws/jobs/{id}` que ya usan los jobs estáticos/dinámicos — cero
plumbing nuevo del lado del streaming, solo una nueva forma de construir el comando. Los jobs
`aipwn` comparten el lock por device-serial con los jobs `"dynamic"` (`QueueEngine.drain()`), ya que
ambos usan el mismo teléfono físico en exclusiva.

**Chat operador→agente:** el WebSocket `/ws/chat/{package}` de Fase 3 ya publicaba mensajes a un
bus en memoria, pero nada los consumía — un job `aipwn` corre como **subproceso aislado** de la
cola (por diseño, ver nota de Fase 1 sobre estado mutuo de `orchestrator.py`), así que no puede ser
un suscriptor WebSocket. Se agregó un **mailbox pull-based** separado del bus
(`nutcracker_core/plugins/dashboard/chat_mailbox.py`): cada mensaje de operador también se guarda
ahí; `FridaAgent._check_operator_chat()` (nuevo método en `frida_agent.py`) hace polling HTTP
best-effort de `GET /api/chat/{package}/pending` **al inicio de cada iteración de su loop ReAct**
(antes de llamar al LLM), y si hay mensajes pendientes los inyecta como un turno `user` real en
`self.messages` — el LLM los ve genuinamente en su próxima respuesta. La URL del dashboard llega al
subproceso vía la variable de entorno `NUTCRACKER_DASHBOARD_URL`, que solo se define cuando
`QueueEngine.extra_env` la trae puesta (el
comando `dashboard` la fija tras calcular host/puerto reales; `serve`/CLI/tests no la definen, así
que `_check_operator_chat()` es un no-op inmediato y gratis para el 100% de las corridas normales
por CLI sin dashboard).

**Validado en vivo, con LLM real (aprobado explícitamente por el usuario, gasta tokens reales de su
`llm.api_key`):**
- Job `aipwn` #1 contra `com.example.tapjacking` (app de prueba aprobada, no una app bancaria real)
  encolado vía `POST /api/queue {"kind":"aipwn","serial":"ZY22GPM27J"}` — el razonamiento real del
  agente (`glm-5.1` vía z.ai) se vio en vivo por `/ws/jobs/{id}`: "Nutcracker thinking", "Nutcracker
  says", tool calls reales (`get_app_analysis()`, `probe_security_violations()`,
  `get_heuristic_bypass_script()`, `enumerate_runtime_classes()`, `take_screenshot()`).
- Job #2: se envió un mensaje real por `/ws/chat/com.example.tapjacking` a los pocos segundos de
  iniciado. El log en vivo mostró la línea inyectada
  `[PRUEBA DE VALIDACION] Operador: detente y confirma que recibiste este mensaje antes de
  seguir.` apareciendo exactamente antes de "Calling LLM (iteration 3)" — confirma el recorrido
  completo operador→WebSocket→mailbox→subproceso aipwn→turno real de la conversación del LLM.
- **Hallazgo colateral, no arreglado (fuera de alcance de esta tarea, pre-existente y ajeno al
  wiring):** ambos jobs terminaron en `LLM error: ... "messages.content.type is invalid, allowed
  values: ['text']"` tras el primer `take_screenshot()` — el proveedor configurado (z.ai/GLM vía
  endpoint OpenAI-compatible) rechaza el formato de mensaje multimodal (imagen) que
  `frida_agent.py` construye para enviar el screenshot al LLM. Bug real de compatibilidad
  proveedor↔formato de mensaje en `aipwn`, no relacionado con la cola/streaming/chat — no se tocó,
  documentado aquí para que quede registrado.

Tests: `tests/test_aipwn_chat_wiring.py` (7 — `FridaAgent._check_operator_chat` construido con
`__new__` para testear el método sin pagar el costo del constructor completo: no-op sin
`NUTCRACKER_DASHBOARD_URL`, inyecta mensajes reales, url-encoding del package, nunca propaga
excepciones de red), `tests/dashboard/test_chat_mailbox.py` (4), 2 nuevos en `test_api.py`
(`/api/chat/{package}/pending`), 1 nuevo en `test_ws.py` (el WS de chat también escribe al
mailbox), 3 en `test_orchestrator.py`/`test_queue_engine.py` (`build_job_cmd(aipwn=True)`, jobs
`aipwn` comparten device-lock con `dynamic`).

**Suite completa: 167/167 tests pasan.** Entorno (device + `nutcracker.db` de prueba) limpiado
después de cada corrida de validación en vivo.

### Follow-up post-entrega: 2 bugs reales encontrados por el usuario en uso normal (2026-07-27)

El usuario reportó "no se puede ver el dispositivo" al usar la pestaña Dispositivo recién
entregada. Investigación en vivo (no reproducible solo con mocks) encontró dos problemas reales:

**🔴 Bug — limpieza de huérfanos no cubría el caso normal de "detener la vista".**
`_kill_orphaned_remote_server()` solo corría al inicio de una sesión y tras un ciclo fallido — el
caso más común (el usuario cierra la pestaña, o el ciclo llega a su fin normal tras
`_RESTART_INTERVAL_S`) no la ejecutaba. Cada vez que eso pasaba, un `app_process` remoto quedaba
vivo para siempre ocupando el socket "scrcpy", haciendo fallar en cascada el siguiente intento
("Server connection failed"). **Fix:** la limpieza ahora corre incondicionalmente después de matar
el proceso local, sin importar por qué terminó el ciclo.

**🟡 UX — sin feedback visual durante los ~10-15s que tarda en aparecer el primer frame.**
Confirmado con capturas Playwright reales: a los 8s de clic en "ver dispositivo" seguía en blanco
(el `<img>` sin `src` cargado colapsa a 0px de alto), recién a los ~12s mostró contenido real.
Sin ningún indicador, esto es indistinguible de "no funciona". **Fix:** `#shot` con
`min-height:280px` (caja negra visible de inmediato) + mensaje explícito "Conectando con scrcpy…
puede tardar varios segundos" hasta que el evento `onload` del `<img>` confirma el primer frame
real, momento en que cambia a "● Video en vivo real".

**Hallazgo adicional del propio usuario, en forma de pregunta ("¿no puede ser más fluido como
webadb.com/scrcpy?")** que llevó a una investigación de rendimiento real:
- Medido contra una grabación real de scrcpy: **decodificar el archivo `.mkv` completo en cada
  poll cuesta 0.5s cuando el archivo lleva ~18s grabando (~5MB)** — muy por encima de
  `_POLL_INTERVAL_S` (0.35s original), y el costo **crece** cuanto más dura el ciclo de grabación
  — la vista se ponía cada vez más entrecortada según pasaba el tiempo dentro de un mismo ciclo de
  20s.
- **Fix (validado empíricamente, no solo razonado):** se agregó `--video-codec-options=i-frame-
  interval=1` al comando de scrcpy (keyframe cada ~1s en vez del default de 10s) y
  `decode_last_frame_jpeg()` ahora hace `container.seek()` a ~1.5s antes del final del archivo en
  vez de decodificar desde el principio. Medido: **0.5s/poll → 0.07s/poll (7x)**, con costo
  prácticamente constante sin importar cuánto lleve grabando — esto permitió subir
  `_RESTART_INTERVAL_S` de 20s a 120s (menos interrupciones de reconexión) sin volver a degradar.
- **Techo real, sin resolver — reportado con honestidad, no oculto:** incluso con el fix, la tasa
  efectiva medida en vivo fue **~1.3 fps** (frames nuevos por segundo), lejos de un video fluido
  real (15-30fps como ws-scrcpy/webadb.com). Sospecha, no confirmada: el propio muxer mkv de
  scrcpy probablemente agrupa varios frames por "cluster" antes de volverlos visibles/legibles en
  disco, imponiendo un techo independiente del costo de decode ya optimizado. Lograr fluidez real
  requeriría el enfoque original (stream H.264 crudo por socket + decode en el navegador vía
  WebCodecs) — exactamente lo que se intentó y no se pudo completar contra este hardware endurecido
  (ver sección "scrcpy real" más arriba). Queda documentado como limitación conocida del enfoque
  actual, no resuelta en esta sesión.

Tests: +2 en `tests/dashboard/test_scrcpy_video.py` (seek cerca del final cuando `container.duration`
lo permite; fallback a decode completo si el seek lanza excepción). **169/169 tests pasan.**

## Fase 4 — video realmente fluido vía WebUSB + WebCodecs (2026-07-27)

El usuario preguntó por qué el video no es tan fluido como
[app.webadb.com](https://app.webadb.com) y si se podía replicar ese enfoque. Investigado,
**confirmado técnicamente factible**, y luego **implementado** a pedido explícito del usuario
("Implementa la fase 4"). Ver más abajo ("Implementación real") para el detalle de qué se
construyó y cómo se verificó — la sección original de investigación queda tal cual, como
referencia de las decisiones tomadas.

### Qué es y por qué sería mejor
app.webadb.com usa **Tango** (antes `ya-webadb`, de yume-chan) — una reimplementación completa y
madura del protocolo ADB **en TypeScript, corriendo en el navegador vía WebUSB**. El navegador se
conecta directo al teléfono por USB (sin pasar por ningún `adb`/proceso del lado servidor), empuja
el `scrcpy-server` real, y decodifica el H.264 crudo con la **WebCodecs API** nativa del navegador,
dibujando en un `<canvas>`. Esto es fundamentalmente distinto al enfoque actual
(`scrcpy_video.py`): no hay archivo intermedio, no hay proceso `scrcpy` del lado del servidor, no
hay polling — es el mismo tipo de decodificación continua que hace un reproductor de video nativo.
Resolvería de raíz el techo de ~1.3fps documentado arriba.

### Paquetes reales (verificados, no supuestos)
```bash
npm install @yume-chan/adb @yume-chan/adb-daemon-webusb @yume-chan/scrcpy \
            @yume-chan/adb-scrcpy @yume-chan/scrcpy-decoder-webcodecs \
            @yume-chan/fetch-scrcpy-server
```
(`adb-daemon-webusb`, no `adb-backend-webusb` — el nombre cambió en algún punto de la evolución del
proyecto; `fetch-scrcpy-server` resuelve en build-time la descarga del `scrcpy-server.jar` con la
versión exacta que espera `@yume-chan/scrcpy`, evitando el mismatch de versión que fue tan
problemático en la investigación de "scrcpy real" de más arriba). Documentación:
[tangoadb.dev/scrcpy](https://tangoadb.dev/scrcpy/).

### Restricciones reales a documentar honestamente (no ocultar)
- **Solo navegadores basados en Chromium** (Chrome, Edge, Opera, Samsung Internet) — WebUSB no
  existe en Firefox ni Safari (macOS/iOS), ninguna versión. ~76% de soporte global. Debe
  degradarse con gracia: feature-detect `"usb" in navigator`, y si no está disponible caer al
  `scrcpy_video.py` actual (que a su vez cae a polling de screenshots) — la misma cadena de
  degradación honesta ya establecida, con un escalón nuevo arriba.
- **Requiere "secure context"**: `http://127.0.0.1`/`http://localhost` cuentan como seguros sin
  necesitar TLS (el uso local típico del dashboard sigue funcionando tal cual), pero si el
  dashboard se expone en la LAN (`--host 0.0.0.0`) y se accede por `http://<ip-lan>:puerto` desde
  otra máquina, **WebUSB no estaría disponible** en ese caso sin HTTPS.
- **El teléfono debe estar conectado por USB a la MISMA máquina que corre el navegador.** Esto es
  una restricción más estrecha que el enfoque actual: `scrcpy_video.py` funciona con cualquier
  device alcanzable por `adb` (USB o red, vía el `adb`/`scrcpy` que ya tenga el usuario) — WebUSB
  no puede alcanzar un device conectado a una máquina remota ni reenviado por red.
- **Posible conflicto de exclusividad de USB**: mientras el navegador tiene el device reclamado
  vía WebUSB, no está confirmado si el `adb`/`scrcpy` del sistema (WSL, Windows, o el propio
  `scrcpy_video.py`) puede seguir usándolo en simultáneo — a validar en vivo antes de dar esto por
  cerrado; podría requerir que el usuario libere la pestaña de video antes de correr un job
  dinámico/aipwn, o viceversa.
- **Primer build step de JS del proyecto.** El dashboard (Fase 3) es deliberadamente un único
  HTML self-contained, sin CDN ni dependencias externas. Estos paquetes de Tango son sustanciales y
  necesitan bundling (Vite/esbuild) — la propuesta es un subproyecto npm nuevo
  (`nutcracker_core/plugins/dashboard/webusb/`, con su propio `package.json`) cuyo `npm run build`
  produce un único archivo `.js` bundleado que se commitea al repo y se sirve como estático más —
  el HTML servido en runtime sigue siendo self-contained (nada se descarga de un CDN en tiempo de
  ejecución), pero el *desarrollo* de ese archivo sí requiere Node/npm, algo que hoy el proyecto no
  necesita para nada del dashboard.

### Integración propuesta (no implementada)
- Nuevo escalón en la cadena de degradación de la pestaña "Dispositivo": **WebUSB+WebCodecs**
  (mejor, si `navigator.usb` existe y el usuario autoriza el device) → `scrcpy_video.py` actual
  (si hay `dashboard.scrcpy_path` configurado) → polling de screenshots (siempre disponible).
- El backend Python **no necesita lógica nueva para el video en sí** — todo el streaming ocurre
  en el navegador, directo al USB. El backend solo serviría el bundle `.js` compilado y el
  `scrcpy-server.jar` correcto como estáticos (ya resueltos en build-time por
  `@yume-chan/fetch-scrcpy-server`).
- Verificación honesta pendiente antes de implementar: confirmar en vivo (contra el mismo Moto G23)
  que el WebUSB del navegador puede reclamar el device sin conflicto con `adb`/`scrcpy` del
  sistema, y medir el fps real resultante contra la promesa de fluidez.

### Implementación real (2026-07-27)

**Investigación de API sin adivinar:** en vez de confiar en resúmenes de la documentación web
(`tangoadb.dev` — el `WebFetch` la resume con un modelo intermedio, con pérdida real de detalle:
varias veces devolvió "esta página no incluye el código completo"), se instalaron los paquetes
reales (`npm install`) y se leyeron directamente sus archivos `.d.ts` — la fuente de verdad exacta.
De ahí salió la API real completa: `AdbDaemonWebUsbDeviceManager.BROWSER.requestDevice()` →
`device.connect()` → `AdbDaemonTransport.authenticate({serial, connection, credentialStore})` →
`new Adb(transport)` → `AdbScrcpyClient.pushServer()` + `AdbScrcpyClient.start()` →
`client.videoStream` → `WebCodecsVideoDecoder` + `WebGLVideoFrameRenderer` sobre un `<canvas>`.

**Nuevo subproyecto:** `nutcracker_core/plugins/dashboard/webusb/` (npm + TypeScript + Vite —
primer build de JS del proyecto, tal como se documentó como trade-off arriba). Ver su propio
`README.md` para el detalle de build/verificación. Resumen:
- `src/main.ts`: `isSupported()` (feature-detect `navigator.usb` + `AdbDaemonWebUsbDeviceManager.BROWSER`
  + `WebCodecsVideoDecoder.isSupported`) y `connect(canvas, onStatus)` implementando el flujo
  completo de arriba. Sin control táctil en esta primera versión (`control: false`) — solo video.
- **El typecheck (`tsc --noEmit` contra los tipos reales) encontró y forzó a arreglar 2 bugs reales**
  antes de siquiera correr nada: un `ReadableStream` nativo del DOM no es estructuralmente
  idéntico al `ReadableStream` propio de `@yume-chan/stream-extra` que `pushServer()` espera
  (difieren en el tipo resuelto de `.closed`) — hubo que importar su implementación explícitamente
  en vez de asumir compatibilidad con la global del navegador.
- **El binario real del scrcpy-server (v3.3.1, descargado en build-time desde GitHub Releases de
  Genymobile vía `@yume-chan/fetch-scrcpy-server`, sin vendorear nada) queda embebido como data URI
  dentro del bundle final** — Vite detecta automáticamente el patrón `new URL(..., import.meta.url)`
  del paquete y lo inlinea. Verificado **byte a byte**: los 90.788 bytes decodificados del bundle
  coinciden exactamente con el `server.bin` descargado. Un solo archivo `.js` (~290KB, +un chunk de
  worker de ~174KB) sirve todo — sin archivos `.bin` sueltos que gestionar ni versionar aparte.
- **Fricción de entorno real, resuelta (y reconfirmada por el usuario en su propio WSL):** `npm`/
  `node` en WSL a veces resuelven al `node.exe`/`npm.cmd` de Windows vía interop — funcionaba para
  `npm install` en este agente, pero **Vite fallaba resolviendo su propio paquete** al cruzar el
  path UNC `\\wsl.localhost\...`. Se resolvió descargando un Node.js Linux nativo
  (`nodejs.org/dist`, tarball, sin necesitar root) y usándolo en vez del `node.exe` de Windows para
  todo el toolchain — evita el cruce de filesystem por completo. **El usuario pegó exactamente este
  mismo problema al correr `npm install && npm run build` en su propio WSL** (`npm warn cleanup
  ... EISDIR` limpiando `node_modules`, luego `vite build` con
  `CMD.EXE ... No se permiten rutas UNC`) — confirma que no era una rareza de este entorno aislado
  sino un problema real y reproducible para cualquiera con Node solo instalado en Windows. Se
  documentó el fix (instalar Node nativo vía `nvm` dentro de WSL) de forma prominente en
  `webusb/README.md`, con el error exacto para que sea buscable.
- **Frontend (`index.html`):** nuevo botón "🔌 USB directo (fluido)" en la pestaña Dispositivo,
  visible solo si `isSupported()` da `true` (import dinámico perezoso del bundle, `await
  import("/static/webusb-video.bundle.js")` — si el archivo no existe porque nadie corrió `npm run
  build`, falla en silencio y el botón simplemente no aparece, cero impacto para quien no lo usa).
  Nuevo `<canvas id="webusb-canvas">` junto al `<img id="shot">` existente — se togglea cuál está
  visible según el modo activo (WebUSB / scrcpy_video.py / polling), solo una fuente de video a la
  vez.

**Validado en un navegador real (Playwright, Chromium headless):** `navigator.usb` presente, el
módulo se importa sin errores, `isSupported()` devuelve `true`, el botón aparece correctamente,
layout limpio (captura real revisada).

### Validación en vivo por el usuario contra su Moto G23 real (2026-07-27, post-entrega)

El usuario probó la conexión real de punta a punta desde Chrome/Edge en Windows, con el teléfono
por USB. Dos problemas reales encontrados y arreglados en el camino, ninguno hipotético:

**🔴 Confirmado el conflicto de exclusividad de USB** (la incógnita explícita de la investigación
original): primer intento falló con
`The device is already in used by another program` — el servidor `adb` de Windows (con el que
scrcpy/frida ya venían trabajando toda la sesión) mantiene una conexión USB persistente al
teléfono, y WebUSB necesita acceso **exclusivo** — no puede coexistir. Fix: `adb kill-server` desde
Windows antes de conectar por WebUSB. No hay forma de evitar esto desde el código — es una
restricción real del modelo de permisos de WebUSB, documentada así en vez de prometer que
"simplemente funciona" en paralelo con `adb`.

**🔴 Bug real de código encontrado y arreglado — `WebGLVideoFrameRenderer` sin fallback.**
Tras resolver el conflicto de USB, la conexión avanzó mucho más lejos (autenticación, push del
server, arranque de scrcpy, stream de video) y falló recién al crear el renderer:
`WebGL not supported` — en una máquina Windows normal, no una VM ni nada exótico (WebGL puede
fallar por aceleración de hardware deshabilitada en el navegador, drivers de GPU, políticas
empresariales, etc. — no es raro en la práctica). El código original solo usaba
`WebGLVideoFrameRenderer` sin condicional. Fix: `WebGLVideoFrameRenderer.isSupported` (el getter
estático que ya existía en el paquete, sin usar) decide entre WebGL y `BitmapVideoFrameRenderer`
(canvas 2D + `createImageBitmap`, más lento pero sin dependencia de WebGL) — mismo patrón de
degradación honesta que el resto del proyecto. Recompilado y re-typechecked limpio
(`npm run check` + `vite build`, bundle 293.62KB → 293.95KB).

Ambos hallazgos vinieron de probar contra hardware/software real, no de imaginar casos borde —
exactamente el patrón que se repitió durante toda la sesión con `scrcpy_video.py` y el resto del
dashboard: el código que parece correcto en el papel (y pasa el typecheck) sigue topándose con
comportamiento real del sistema operativo/navegador que solo aparece al usarlo de verdad.

**✅ Confirmado funcionando por el usuario, con video real:** tras el fix del renderer, la conexión
mostró la pantalla real del Moto G23 (launcher con apps reales) dentro del `<canvas>` — captura
real revisada. **Fase 4 queda validada de punta a punta contra hardware físico real**, no solo con
mocks/Playwright headless.

**🟡 Bug de CSS encontrado y arreglado en el mismo intercambio** — la imagen se veía distorsionada
(estirada horizontalmente). Causa: `#webusb-canvas` tenía `width:100%; max-height:70vh` pero **sin
`object-fit:contain`** (a diferencia de `#shot`, que sí lo tenía desde el fix de UX anterior) —
sin eso, cuando `max-height` recorta la altura, el ancho no se reduce en proporción y la imagen se
deforma. Fix de una línea en `index.html` (agregar `height:auto; object-fit:contain`) — **no
requiere recompilar el bundle de npm**, es CSS puro del HTML servido directamente. Verificado con
un canvas de prueba a la proporción real de un teléfono (720×1600): un círculo dibujado se ve
círculo perfecto, no óvalo, con las barras de letterboxing correctas a los costados (captura real
revisada, no solo razonado).

Sin tests de Python nuevos (este es código TypeScript/CSS que corre 100% en el navegador, fuera del
alcance de `pytest`) — la verificación es el typecheck + build + Playwright + la validación en vivo
del usuario, dos veces (conexión real, luego el fix visual). **181/181 tests de Python siguen
pasando** (sin cambios en el backend) en cada paso.

## Fix: "re-analizar" fallaba para apps analizadas desde un .apk local (2026-07-27)

El usuario reportó, al usar el dashboard real: clic en "re-analizar" para `sh.nutcracker.nutbank`
(su propia app de prueba, no una app bancaria real) devolvía
`Download error: APK no encontrada en 'downloads' tras la descarga`.

**Causa raíz:** `reQueue(pkg)` en el frontend siempre reencola usando el **package id** como
target. Cuando el target no es una ruta local (`_is_local_apk()` da False), `build_job_cmd` arma
un job `scan <package>`, que intenta *descargar* la app desde Google Play/APKPure. Esto funciona
para apps reales publicadas en una store, pero `sh.nutcracker.nutbank` es la app de demo propia del
usuario (`downloads/nutbank.apk`, analizada originalmente vía `analyze <ruta-local>`) — nunca
estuvo publicada en ninguna store, así que la re-descarga por package id no puede funcionar nunca,
sin importar credenciales.

Investigando más a fondo: el esquema de `apps` ya modelaba justo esta distinción
(`source TEXT -- apkpure | google-play | local | url`, ver `schema.sql`) pero **la columna nunca se
escribía en ningún lado del código** — un campo muerto desde Fase 0, no algo que esta sesión rompió.
Sin ese dato, el dashboard no tenía forma de saber "esta app se puede re-analizar desde tal ruta
local" en vez de "hay que volver a descargarla".

**Fix:**
- `orchestrator._run_analysis()`: justo antes de disparar el post-hook `after_analysis`, si
  `keep_apk=True` (el archivo va a seguir existiendo después de esta función) deja la ruta
  resuelta en la variable de entorno `NUTCRACKER_APK_SOURCE`; si no, la limpia. No se pudo pasar
  como kwarg directo al hook porque `fire_post_hooks("after_analysis", ...)` comparte firma fija
  con otros post-hooks (p.ej. `aireview`) que no aceptan kwargs extra — mismo patrón ya usado para
  `NUTCRACKER_QUEUE_JOB_ID`.
- `store/hooks.py::_persist_after_analysis()`: si esa env var está presente, llama
  `repository.upsert_app(conn, package, source=f"local:{ruta}")` (el `UPSERT` de `upsert_app` ya
  soportaba actualizar `source` en un run posterior, vía `COALESCE(excluded.source, apps.source)`
  — solo faltaba que alguien lo llamara con un valor real).
- Frontend (`reQueue()`): si `app.source` empieza con `"local:"`, usa esa ruta como target en vez
  del package id — la app se re-analiza desde el mismo `.apk` que la primera vez, sin intentar
  descargar nada.
- **Dato ya existente corregido a mano** en la `nutcracker.db` real del usuario (no una prueba):
  `UPDATE apps SET source = 'local:/home/hiteek/nutcracker2/nutcracker/downloads/nutbank.apk'
  WHERE package = 'sh.nutcracker.nutbank'` — la única fila de `apps` en su base, analizada antes
  de que este fix existiera, así que "re-analizar" funciona ya mismo sin esperar un ciclo nuevo.
- **No requiere reiniciar el dashboard**: cada job de la cola corre como subproceso aislado
  (`python nutcracker.py analyze/scan ...`), que importa el código actualizado desde disco en cada
  invocación — solo el cambio de frontend necesita recargar la página del navegador.

Tests: 2 en `tests/test_store_hooks.py` (con/sin `NUTCRACKER_APK_SOURCE` en el entorno), 2 en
`tests/test_orchestrator.py` (`_run_analysis` fija/limpia la env var según `keep_apk`, con todas
las dependencias pesadas mockeadas — `APKAnalyzer`, `_post_analysis_flow`,
`save_analysis_json`, `fire_post_hooks`, etc.). **181/181 tests pasan.**

## Feature: batch estático+aipwn desde un .txt, con fuente de APK elegible (2026-07-27)

Pedido del usuario: programar en cola varios escaneos estáticos + aipwn desde un archivo `.txt`
con package ids, pudiendo elegir si el `.apk` de cada uno se descarga de una store o se extrae del
ya instalado en el dispositivo conectado. Confirmado con el usuario vía `AskUserQuestion`: (1) todo
package del archivo recibe **siempre** estático + aipwn encadenado (no control mixto por línea), y
(2) la elección de fuente device-vs-descarga es un **flag global** para todo el archivo, no por app.

**Diseño:** reusa 100% la infraestructura existente en vez de construir un motor nuevo —
`QueueEngine.drain(on_result=...)` ya notifica cada job terminado; basta con acumular los packages
cuyo job estático terminó OK y llamar `engine.submit(pkg, kind="aipwn", ...)` + un segundo
`drain()` tras el primero (el patrón multi-`drain()` ya lo ejercitaba
`test_drain_picks_up_jobs_queued_by_a_different_engine_instance`, así que no era terreno nuevo).

**Cambios:**
- `downloader.py`: nueva clase `DeviceInstalledDownloader` — `adb shell pm path <pkg>` (soporta App
  Bundle con varios splits) + `adb pull` de cada uno a `downloads/<pkg>/`, identificando `base.apk`
  como el archivo principal. `download_apk_from_config()` gana un branch `source == "device"` y un
  parámetro `serial` que dispatchea a esta clase antes de la auto-selección google-play/apk-pure.
- `cli/scan.py`: `--source` gana la opción `"device"`, nueva opción `--serial` (solo usada con
  `--source device`).
- `orchestrator.build_job_cmd()`: nuevo parámetro `source`; en el branch `scan` (target no es un
  `.apk` local) agrega `--source <source>` y, si `source == "device"`, también `--serial`.
- `queue/job.py`: `Job.source` — igual que `is_local_apk`, vive solo en memoria (no persiste en
  SQLite); un job recuperado tras un reinicio del daemon cae a la descarga normal en vez de
  perderse (documentado en el propio campo).
- `queue/engine.py`: `QueueEngine.submit(source=...)` lo pasa al `Job` construido; `_run_job()` lo
  reenvía a `build_job_cmd(source=job.source)`. **Fix encontrado durante la implementación** (no
  reportado por el usuario, detectado al revisar el código): el heurístico existente
  `_resolve_local_apk()` (que reutiliza un `.apk` ya en `downloads/<pkg>/` para evitar una descarga
  repetida) corría *antes* de mirar `job.source` — si el usuario pedía explícitamente
  `--source device` pero `downloads/<pkg>/` ya tenía un `.apk` de un intento anterior con otra
  fuente, el pedido explícito se ignoraba en silencio. Fix: el heurístico ahora se salta por
  completo cuando `job.source` está seteado (un pedido explícito de fuente siempre manda).
- `cli/queue_cmd.py`: `queue add` gana `--source`/`--serial` (se pasan tal cual a cada
  `engine.submit()`) y `--then-aipwn` — con `--run`, agrupa el resultado de cada job estático en
  `pending_aipwn` (dentro de `on_result`, solo si `kind == "static"` y `ok` y hay `package`), y tras
  el primer `drain()` encola un job `aipwn` por cada uno y hace un segundo `drain()`. Rechaza
  combinarse con `--dynamic`/`--aipwn` (son kinds de job distintos al estático que `--then-aipwn`
  asume como base).

Uso: `nutcracker queue add packages.txt --then-aipwn --source device --serial <serial> --run`.

Tests nuevos (16, **197/197 pasan** en total): `tests/test_downloader_device.py`
(`DeviceInstalledDownloader` — split de App Bundle, package no instalado, adb ausente, `pull`
fallido, dispatch desde `download_apk_from_config`), 4 en `test_orchestrator.py` (`--source`/
`--serial` en el cmd de `scan`, ausentes cuando no aplica, ignorado en el branch `analyze`), 2 en
`test_queue_engine.py` (`source` llega al cmd construido, y el fix del heurístico de reuso de APK
local), 4 en `test_queue_cmd.py` (encadena aipwn tras estático OK, no encadena tras uno fallido,
rechaza `--then-aipwn --dynamic`, `--source`/`--serial` se propagan a cada `submit()` del archivo).
Alcance solo CLI (`queue add <archivo.txt> ...`) — el dashboard web no se tocó para esta feature,
ya que su API REST opera sobre un target a la vez, no sobre un archivo.

## Feature: la misma cola batch, ahora también desde el dashboard (2026-07-27, follow-up)

El usuario preguntó si el batch anterior (estático+aipwn encadenado desde un `.txt`, con fuente de
`.apk` elegible) se podía disparar también desde el dashboard, no solo por CLI. Confirmado que sí y
pedido implementarlo.

**Diseño:** reusa la misma lógica de encadenado que `cli/queue_cmd.py` (mismo patrón `submit` →
`drain(on_result=...)` acumulando packages OK → `submit(kind="aipwn")` por cada uno → segundo
`drain()`), pero corriéndola en un hilo de fondo — igual que ya hacía `_drain_in_background()` para
un solo job — en vez de bloquear la respuesta HTTP.

**Cambios:**
- `plugins/dashboard/api.py`: nuevo `POST /api/queue/batch` (`QueueBatchPayload`: `targets: list[str]`,
  `source`, `serial`, `then_aipwn: bool = True`). Encola un job estático por cada target (filtrando
  líneas vacías/`#comentarios`) y lanza `_drain_batch_in_background()`, que replica exactamente la
  lógica de `queue_cmd.py` (acumula `pending_aipwn` dentro de `on_result`, solo si `kind == "static"`
  y `ok` y hay `package`; tras el primer `drain()` encola los `aipwn` y hace un segundo `drain()`).
  Reusa el mismo guard `engine._dashboard_draining` que ya usaba `_drain_in_background()` para el
  path de un solo job — extraje el publish-a-bus común (`_publish_job_status`) para no duplicarlo
  entre ambos caminos.
- `static/index.html`: nuevo bloque "Batch desde archivo" en la tarjeta de cola — `<input
  type="file" accept=".txt">` + un `<select>` de fuente (store/device) + botón. El archivo se lee
  100% client-side con `File.text()` y se parte en líneas en JS — no hace falta subir un multipart
  al backend, el body que viaja es un JSON `{targets: [...], source, serial, then_aipwn: true}`
  igual que cualquier otro POST del dashboard. El progreso no necesita UI nueva: los jobs (estáticos
  y luego los `aipwn` encadenados) aparecen solos en la tabla de cola ya existente, que ya hacía
  polling cada 5s (`refreshQueue()` vía `setInterval(refreshAll, 5000)`).

**Verificación:** 5 tests nuevos en `tests/dashboard/test_api.py` vía `TestClient` (rechaza lista
vacía, ignora líneas en blanco/comentarios, encadena aipwn tras estático OK — simulando
`repository.link_job_run` como ya hacía `test_reschedule_sets_next_due_at_after_job_completes` en
`test_queue_engine.py` para que `outcome.package` quede seteado —, no encadena tras uno fallido,
`source`/`serial` llegan a cada `submit()` vía un spy). **202/202 tests pasan.**

Además, verificación en vivo (no solo mocks): arrancado `nutcracker dashboard` de verdad contra un
`config.yaml` aislado en el scratchpad (`store.db_path` propio, nunca toca la `nutcracker.db` real
del usuario — mismo cuidado que en la sesión de Fase 4/WebUSB) y golpeado `POST /api/queue/batch`
por HTTP real dos veces: una con un package inexistente (fuente store, termina en `error` con el
mensaje esperado "APK no encontrada tras la descarga") y otra con `source=device` sin ningún
dispositivo conectado (termina en `error` con "no está instalado en el dispositivo", confirmando
que el branch `DeviceInstalledDownloader` se ejecuta de verdad). Confirma que el wiring
HTTP→QueueEngine→subprocess→SQLite funciona end-to-end, no solo bajo `TestClient`. No se pudo
probar el click real en el navegador (sin herramienta de automatización de navegador disponible en
esta sesión) — el HTML servido sí se verificó (`curl` confirma que `batch-file`/`batch-source`/
`runBatch()` están presentes en la página).

## Reimplementación completa del dashboard tras pérdida del plugin (2026-07-27)

`nutcracker_core/plugins/dashboard/` desapareció por completo del disco entre sesiones (la carpeta
está gitignoreada a propósito — regla `plugins/` sin anclar, ver nota de Fase 3 — así que nunca
estuvo en git; se perdió por algún motivo ajeno a git, no identificado). Se reconstruyó desde cero
usando este mismo documento (Fases 3/4 más arriba) y los tests ya existentes (`tests/dashboard/*.py`,
escritos contra la implementación anterior) como especificación ejecutable: `events.py`,
`store_reader.py`, `device.py`, `chat_mailbox.py`, `scrcpy_video.py`, `api.py`, `ws.py`, `server.py`,
`__init__.py` (comando `nutcracker dashboard`), `static/index.html`, y el subproyecto
`webusb/` completo (instalado, typechequeado contra los `.d.ts` reales — encontró y corrigió 7
desajustes de API respecto a lo escrito de memoria — y compilado: bundle real de 315KB + worker de
173KB, con el `scrcpy-server` v3.3.1 embebido verificado byte a byte). Node.js nativo se instaló en
WSL para esto (evitando el `node.exe`/`npm.cmd` de Windows vía interop, misma fricción ya
documentada en Fase 4). 202/202 tests de Python vuelven a pasar.

Dos bugs reales encontrados y arreglados durante la reconstrucción, ambos por reporte directo del
usuario contra el dashboard real corriendo:
- **`connectWebUsb()` sin manejo de errores**: cualquier fallo de la cadena WebUSB (el más común:
  conflicto de exclusividad USB con `adb`, ya documentado en Fase 4) quedaba como unhandled
  promise rejection — el canvas se veía en blanco sin ningún mensaje visible. Fix: try/catch en
  `main.ts` (con detección específica del mensaje "already in used" para sugerir `adb kill-server`)
  y en `connectWebUsb()` como defensa adicional, restaurando la vista de polling si falla.
- **`multipart/x-mixed-replace` en `<img src>` poco confiable en Chrome/Edge**: el backend
  producía frames JPEG válidos (confirmado con `curl`/`requests` reales), pero Chrome no
  re-renderizaba el `<img>` con cada parte nueva — reportado como "no se ve nada". Fix: parsear el
  stream a mano vía `fetch()` + `ReadableStream`, actualizando `<img>` con un Blob URL por frame.

### Eliminación de "Video en vivo (scrcpy)" a pedido del usuario (2026-07-27)

El usuario pidió explícitamente sacar el modo de video basado en archivo (`scrcpy_video.py`:
`scrcpy --no-window --record=<mkv>` + PyAV releyendo el archivo, ~1-2fps) y dejar únicamente el
modo fluido (WebUSB + WebCodecs, Fase 4) más el polling de screenshots existente como fallback
universal. Justificación implícita: el modo scrcpy-archivo también arrancaba automáticamente al
abrir la pestaña "Dispositivo" (otro fix de esta misma sesión, antes de esta eliminación) y quedó
redundante frente a WebUSB para quien ya tiene un navegador Chromium — y añadía una dependencia
pesada (PyAV/`av`) y una superficie de bugs (huérfanos de `scrcpy` en el device, fps limitado) que
ya no aportaba nada que WebUSB o el polling no cubrieran mejor.

**Eliminado:** `nutcracker_core/plugins/dashboard/scrcpy_video.py` (módulo completo),
`tests/dashboard/test_scrcpy_video.py`, los endpoints `GET /api/device/video` y
`GET /api/device/video/status` (`api.py`), sus 3 tests en `test_api.py`, el botón "▶ Video en vivo
(scrcpy)" y las funciones `toggleDeviceView()`/`streamMjpeg()`/`_findCRLFCRLF()` (`index.html`),
`dashboard.scrcpy_path` de `config.yaml`/`config.yaml.example`, la dependencia `av>=11.0`
(`requirements.txt` del plugin y el extra `[dashboard]` de `pyproject.toml`), y las secciones
correspondientes de `README.md`. `create_app()`/`create_router()` perdieron el parámetro `config`
(ya no lo necesita ningún endpoint).

**Sin tocar:** el modo WebUSB (Fase 4) y el polling de screenshots (`/api/device`,
`/api/device/screenshot`) quedan exactamente igual — la pestaña "Dispositivo" ahora solo tiene esos
dos modos, con WebUSB 100% opt-in vía su botón.

Suite completa vuelta a correr tras la eliminación (ver más abajo en la sesión) para confirmar que
nada quedó roto.

### Eliminación también del polling de screenshots (2026-07-27, mismo día)

El usuario pidió sacar también el fallback de polling de screenshots (`GET /api/device`,
`GET /api/device/screenshot`, `device.py`) y dejar el tab "Dispositivo" con **únicamente** el modo
WebUSB ("🔌 USB directo (fluido)") — sin ningún fallback de video/captura vía `adb`.

**Eliminado:** `nutcracker_core/plugins/dashboard/device.py` (módulo completo — `list_serials()`,
`screenshot_png()`), los endpoints `GET /api/device` y `GET /api/device/screenshot` (`api.py`), sus
2 tests en `test_api.py`, el selector de serial (`<select id="device-serial">` + botón "Refrescar
seriales"), el `<img id="shot">`, y las funciones `refreshDevices()`/`pollScreenshot()` de
`index.html`. El tab "Dispositivo" ahora solo tiene el botón WebUSB + el `<canvas>`; sin bundle
compilado o sin soporte del navegador, el botón simplemente no aparece y el status lo indica
explícitamente (sin fallback silencioso a ningún otro modo).

Verificado contra el dashboard real reiniciado con el código nuevo: `GET /` → 200, `GET
/api/summary` → 200, `GET /api/device` → 404 (confirma que se eliminó, no que quedó roto), bundle
WebUSB sigue sirviéndose bien. Suite completa: **179/179 tests pasan** (181 → 179, los 2 tests de
`device.py` eliminados junto con el módulo).

## adb-over-Wi-Fi para evitar el conflicto de exclusividad USB con WebUSB (2026-07-27)

El usuario reportó `--source device` fallando en un batch job con `adb.exe: no devices/emulators
found` al tener WebUSB conectado en el dashboard (conflicto de exclusividad ya documentado en Fase
4, esta vez en la dirección USB→adb en vez de adb→USB). Propuso — y se implementó — la solución
estándar: mover `adb` a TCP/IP (`adb tcpip 5555` + `adb connect <ip>:5555`) para que use la red en
vez del mismo cable USB que WebUSB reclama en exclusiva.

Encontrado en el camino, con uso real: la conexión `adb connect <ip>:puerto` vive en el servidor
adb local y **no sobrevive un reinicio del daemon** (a diferencia de `adb tcpip` en el propio
teléfono, que sí persiste) — cualquier reinicio (crash, `adb kill-server`, sleep/wake de Windows)
deja el serial de red "perdido" hasta reconectar a mano. Peor aún, conectar WebUSB parece
desestabilizar momentáneamente el propio `adb.exe` de Windows (visto en vivo: "could not read ok
from ADB Server", "failed to start daemon", "cannot connect to daemon" — errores *transitorios*
del daemon, no del device/paquete, que se resuelven solos en 1-2s).

**Fix en `downloader.py`** (`DeviceInstalledDownloader`, usado por `--source device`):
- `_ensure_network_serial_connected()` — si el serial tiene forma `host:puerto`, corre `adb
  connect` antes de cualquier `pm path`/`pull` (idempotente, no rompe nada si ya estaba
  conectado). No-op para seriales USB normales.
- `_is_daemon_transient_error()` — detecta los 4 patrones de falla transitoria del daemon listados
  arriba.
- `_run_adb()` — envuelve cada invocación de `adb` (connect, pm path, pull) con hasta 3 reintentos
  (pausa de 1.5s) cuando la salida matchea un error transitorio del daemon, en vez de abortar el
  job entero por un problema que se resuelve solo.

Documentado para el usuario: usar la IP:puerto del teléfono (no el serial USB) como `--serial` al
correr batch/queue con `--source device` mientras WebUSB está conectado.

**Limpieza de la cola**: a pedido del usuario, se vació `queue_jobs` (2736 filas acumuladas de
pruebas repetidas, 2566 en estado `queued`) vía SQL directo — confirmado explícitamente con el
usuario que el alcance era solo la cola, sin tocar `apps`/`runs`/`findings`/`schedule` (historial
real de análisis de apps bancarias reales del usuario).

Tests nuevos (6, `tests/test_downloader_device.py`): detección de los 4 marcadores de error
transitorio, no-op para seriales USB, `adb connect` disparado para seriales `ip:puerto`,
reintentos ante falla transitoria (tanto en `_ensure_network_serial_connected` como en el propio
`pm path` de `download()`). **187/187 tests pasan.**

## Borrado de jobs pendientes en la cola (2026-07-27)

Pedido del usuario: poder borrar un job de la cola antes de que corra.

**`repository.delete_job(conn, job_id) -> bool`** — borra solo si `status='queued'` (`DELETE ...
WHERE id = ? AND status = 'queued'`, `rowcount > 0` como señal de éxito). Deliberadamente NO borra
jobs `running` (ya tienen un subproceso real corriendo en algún proceso -- borrar la fila no lo
mata, solo lo desincroniza) ni `done`/`error` (son historial). Expuesto en:
- CLI: `nutcracker queue rm <id> [<id> ...]` (`cli/queue_cmd.py`).
- Dashboard: `DELETE /api/queue/{job_id}` (`api.py`, 404 si no existe o no está `queued`) + botón
  🗑 por fila en la tabla de cola, visible solo para jobs en estado `queued`.

**Limitación conocida, no resuelta (inherente al diseño de la cola)**: si un job ya fue cargado a
memoria por un `drain()` en curso en OTRO proceso (CLI/scheduler/dashboard corriendo en paralelo),
borrar la fila de SQLite no cancela ese despacho en memoria -- `_load_queued_from_db()` solo
recarga al *empezar* un `drain()`, no re-consulta por cada job durante la ejecución. Cubre el caso
común (sacar un job de la cola antes de que cualquier proceso lo levante), no una cancelación en
caliente de un job ya en pleno `drain()`.

Tests nuevos (5): `test_delete_job_removes_queued_job`/`_returns_false_for_unknown_id`/
`_refuses_to_delete_running_job` (`test_queue_engine.py`, este último con un job real bloqueado en
un hilo de fondo vía `threading.Event` para confirmar que `delete_job` no lo toca mientras está
`running`), `test_queue_delete_removes_pending_job`/`_404_for_unknown_job`
(`tests/dashboard/test_api.py`). **192/192 tests pasan.**

## Respuesta: ¿paralelo o secuencial? + investigación de crashes de WSL (2026-07-27)

El usuario preguntó si los jobs corren en paralelo o secuencial (sospechando que eso causaba
crashes periódicos de su entorno WSL), pidiendo investigar de paso.

**Respuesta:** en paralelo, por diseño (`config.yaml`, bloque `queue:`): `static_workers: 4` (hasta
4 análisis estáticos -- decompilación jadx/apktool, semgrep, OSINT -- corriendo a la vez, cada uno
en su propio subproceso `nutcracker analyze/scan`) y `dynamic_workers: 2` (hasta 2 *dispositivos*
distintos en paralelo para jobs Frida/ADB, pero siempre serializado dentro del mismo serial vía el
lock por device de Fase 1). `static_workers: 1` en `config.yaml` lo vuelve estrictamente
secuencial si se prefiere.

**Investigación de los crashes** (sin poder reproducirlos en vivo -- ya habían pasado):
- `dmesg` no tenía entradas de OOM-killer al momento de revisar (el propio reinicio de la VM de
  WSL2 borra su buffer de kernel, así que esto no descarta un OOM previo, solo que no quedó rastro).
- **No existe `.wslconfig`** en el usuario (`/mnt/c/Users/*/.wslconfig` no encontrado) -- WSL2 usa
  el límite de memoria *default* (mínimo entre 50% de la RAM del host y 8GB). `free -h` reportó
  **7.7GB de total** para la VM de WSL2, confirmando ese techo.
- Disco descartado como causa: 933GB libres de 1TB, `downloads/`+`decompiled/` juntos solo 750MB.
- **Hipótesis más plausible** (no confirmada en vivo, pero consistente con la evidencia): 4 jobs
  estáticos en paralelo (jadx/semgrep/androguard, herramientas conocidas por su uso pesado de
  memoria con APKs grandes/ofuscados) + video WebUSB/scrcpy + el propio VS Code Server (~1.5GB de
  base, visto en `ps aux --sort=-%mem`) pueden acercarse o superar ese techo de 7.7GB, llevando a
  Windows/Hyper-V a matar o reiniciar la VM de WSL2 -- indistinguible para el usuario de "se me
  crashea WSL".
- **Recomendado al usuario** (no aplicado automáticamente -- requiere tocar un archivo del lado de
  Windows y reiniciar WSL, decisión del usuario): crear `.wslconfig` con un `memory=` más alto si
  el host tiene RAM de sobra, y/o bajar `queue.static_workers` de 4 a 2 en `config.yaml` para
  reducir el pico de uso concurrente.

---

## Keepalive del transporte adb-over-wifi (2026-07-27)

**Síntoma reportado:** el video WebUSB del dashboard fallaba a cada rato con *"el dispositivo ya
está reclamado por otro programa (adb)"*, sin que el usuario ejecutara nada a mano. Además, jobs de
la cola morían con `device '172.20.10.6:5555' not found` pese a que el teléfono estaba accesible.

### Diagnóstico (con evidencia del entorno real, no supuestos)

1. **El driver ya era WinUSB, así que Zadig no aplicaba.** Se había sugerido reasignar el driver de
   la interfaz ADB a WinUSB; la inspección del dispositivo real lo desmintió:

   ```
   DEVPKEY_Device_Service        : WINUSB
   DEVPKEY_Device_DriverDesc     : WinUsb Device
   DEVPKEY_Device_DriverProvider : Microsoft
   ClassGuid                     : {88BAE032-5A81-49F0-BC3D-A4FF138216D6}   (clase USBDevice)
   ```

   El teléfono ya estaba en WinUSB. Zadig habría sido un no-op. **La causa real del conflicto es que
   WinUSB permite un solo handle abierto por interfaz**, y tanto `adb.exe` (que en Windows accede vía
   WinUSB) como Chrome (WebUSB → WinUSB) piden *la misma* interfaz. Ningún cambio de driver lo
   resuelve: es exclusividad estructural.

2. **El daemon adb reclama el cable aunque solo se lo use por TCP.** Demostrado en vivo: con
   `strategies.default_device_id = "172.20.10.6:5555"` y los 3 jobs en vuelo usando ese serial,
   `adb devices -l` mostraba igual **ambos** transportes:

   ```
   ZY22GPM27J         device ... transport_id:2   <- el CABLE (nadie lo pidió)
   172.20.10.6:5555   device ... transport_id:3   <- el WiFi
   ```

   Un daemon recién arrancado enumera y abre toda interfaz ADB USB que vea, sin importar el
   transporte que se le pida. Por eso la receta popular de "conectá el daemon por TCP y el cable
   queda libre" **es falsa**: separar canales es necesario pero no suficiente.

3. **La causa inmediata del fallo: el canal TCP se caía solo.** `adb -s 172.20.10.6:5555 get-state`
   devolvía `device not found` mientras el teléfono respondía a ping y tenía el 5555 abierto — lo
   caído era el *transporte*, no el dispositivo. Esa conexión vive en el daemon adb y no sobrevive a
   un reinicio suyo (crash, `adb kill-server`, sleep/wake) ni a cortes de red del teléfono (doze,
   hotspot). Con el canal 2 muerto, **todo colapsa sobre el cable** — exactamente el conflicto que la
   separación buscaba evitar. La reconexión automática existente vivía solo en `downloader.py`, y se
   dispara al descargar: no cubría los huecos entre jobs ni jobs que no descargan.

### Implementación

- **Nuevo `nutcracker_core/adb_transport.py`** — hogar canónico de la salud del transporte adb:
  - `is_network_serial()` distingue serial de red de serial USB.
  - `is_transport_alive()` usa `adb -s <serial> get-state` en vez de parsear `adb devices`: con un
    solo comando distingue *ausente* (rc≠0), *presente pero inutilizable* (`offline`,
    `unauthorized`) y *listo* (`device`).
  - `ensure_connected()` = la lógica que estaba en `downloader.py` (`adb connect` idempotente con
    reintentos ante fallas transitorias del daemon, ver `_DAEMON_TRANSIENT_MARKERS`).
  - `ensure_available()` chequea estado **antes** de reconectar. Esto es deliberado: `adb connect`
    arranca el daemon si no corre, y un daemon nuevo re-reclama el cable USB — si el transporte ya
    está vivo, no hay razón para arriesgar ese efecto colateral contra el video WebUSB.
  - `TransportKeepAlive` — hilo daemon que revalida periódicamente los seriales de red vigilados.
- **`downloader.py`** pasa a usar el módulo compartido (reexporta los nombres privados anteriores
  para no romper a quien los importaba) y `download()` usa `ensure_available()` en vez del `connect`
  a ciegas.
- **`QueueEngine._ensure_transport()`** revive el transporte antes de cada job con serial de red, y
  registra ese serial en el keepalive si hay uno asignado (cubre jobs con serial distinto al del
  config). Es best-effort: si no se puede restablecer, el job se lanza igual para que falle con su
  propio mensaje, más específico que cualquiera que diéramos acá.
- **`nutcracker dashboard` y `nutcracker serve`** levantan un `TransportKeepAlive` sobre
  `strategies.default_device_id` y lo paran al salir. Nuevo `dashboard.adb_keepalive_seconds`
  (default 60, piso efectivo 10) en `config.yaml`/`config.yaml.example`.

### Verificación

- Suite completa: **214 tests pasan**. 20 nuevos (`tests/test_adb_transport.py` + integración con el
  engine en `tests/test_queue_engine.py`), incluido el caso de que un transporte ya vivo **no** se
  reconecte (protege al WebUSB), y que un chequeo que explota no tumbe el job ni el hilo.
- Prueba funcional contra el teléfono real: se simuló la falla exacta con `adb disconnect`, y
  `ensure_available()` la detectó y la recuperó (`alive: False` → reconexión → `alive: True`, nuevo
  `transport_id`).

### Límite que sigue en pie (no lo arregla este cambio)

WebUSB y `adb.exe` **no pueden compartir el cable**: WinUSB es de handle exclusivo. Lo que este
cambio garantiza es que el canal TCP no se muera, para que el cable pueda quedar para el navegador
sin que la cola se quede sin vía al teléfono. El orden sigue importando: conviene conectar el video
WebUSB **primero**; cuando el daemon reintente el cable y lo encuentre ocupado, sigue funcionando
por TCP.

---

## WebUSB: sobrevivir a un F5 (2026-07-27)

**Pedido:** que el video "USB directo (fluido)" no muera al recargar la página.

### Lo que no se puede, y por qué

Un reload destruye el contexto JS y con él el handle USB, el decoder y el canvas. **La sesión no
puede seguir viva**: no existe API de navegador que preserve una conexión WebUSB a través de una
navegación. Lo que sí persiste es el **permiso** que el usuario otorgó al origen — y eso alcanza
para reconstruir la sesión sin intervención.

### Implementación

- `AdbDaemonWebUsbDeviceManager.getDevices()` (verificado en `manager.d.ts`: *"Get all connected and
  requested devices"*) devuelve los dispositivos ya autorizados **sin abrir el picker**, que además
  exigiría un gesto del usuario. El F5 pasa de "elegí el device otra vez y esperá" a "esperá un par
  de segundos".
- `main.ts` se reestructuró: `connect()` (con picker, requiere click) y `reconnect()` (sin picker)
  comparten `startVideo()`. Ambos devuelven el serial conectado (o `null`), que el frontend usa para
  recordar a qué dispositivo volver.
- `reconnect()` **reintenta ante "device busy"** (3 intentos, 600ms): justo después de un reload
  Chrome puede tardar un instante en liberar el handle de la sesión anterior, y el primer intento se
  topa con su propio fantasma.
- Nuevo `disconnect()` + botón **"⏏ Soltar USB"**. El cable es exclusivo (WinUSB, un handle por
  interfaz), así que sin una forma explícita de soltarlo la única salida era cerrar la pestaña —
  inaceptable en este flujo, donde el usuario alterna entre video y jobs por adb. Cierra decoder,
  client y adb, cada uno en su propio `try`: si el primero falla (device desenchufado en caliente),
  los demás igual corren, o el handle queda tomado hasta recargar.
- `startVideo()` llama a `disconnect()` primero: dos sesiones sobre el mismo cable se pisan (el
  device queda "busy" contra sí mismo).
- El pipe del stream ahora usa un `AbortController`; un abort es cierre normal, no un error que
  reportar en la UI.

### Decisión de diseño: sessionStorage, no localStorage

El serial se guarda en **`sessionStorage`**, que tiene exactamente la vida útil que se busca:
sobrevive a un reload, muere al cerrar la pestaña. Así la reconexión automática ocurre **solo en la
pestaña donde el usuario ya pidió video explícitamente**. Abrir el dashboard de cero nunca reclama
el USB por su cuenta — reclamarlo sin que lo pidan dejaría a adb sin cable y rompería los jobs de la
cola (el usuario ya había rechazado un auto-arranque parecido, ver la sección de eliminación del
auto-trigger de scrcpy). `disconnect()` borra el serial *antes* de cerrar: si el cierre falla a
mitad, el próximo F5 no debe volver a reclamar el cable solo.

### Verificación

- `tsc --noEmit` limpio; `npm run build` OK (bundle de 317KB — no el bundle vacío de 0 bytes que
  producía el modo "app" de Vite antes del fix a `build.lib`).
- Los 5 exports (`isSupported`, `connect`, `reconnect`, `disconnect`, `isConnected`) verificados
  presentes en el bundle emitido, no solo en el fuente.
- Script inline del dashboard parsea sin errores de sintaxis; el server sirve el HTML y el bundle
  actualizados (HTTP 200).
- Suite Python: 214 tests siguen pasando (sin cambios de backend en esta tanda).

**Pendiente de prueba manual del usuario:** el ciclo real conectar → F5 → reconexión automática
contra el teléfono, que no se puede ejercitar sin un navegador con el cable enchufado.

---

## Fix: aipwn no persistía nada + diagnóstico de "hallazgos vacíos" (2026-07-27)

**Reportado por el usuario:** "los cambios no se guardaron de los análisis" + "en apps, al revisar hallazgos no encuentro ninguno".

### Hallazgo 1 (bug real, corregido): jobs `aipwn` nunca tocaban SQLite

`fire_post_hooks("after_analysis", ...)` (el único punto que dispara `store/hooks.py`, que escribe
`runs`/`findings`/`artifacts`) vive **solo** dentro de `orchestrator._run_analysis()` — la función
que usan `scan`/`analyze`. `aipwn.py` es un comando aparte que nunca pasa por ahí. Confirmado en
vivo: jobs `aipwn` con `status: done` pero `run_id: None` (`2798`, `2796`, `2792`), mientras que jobs
`static` de los mismos paquetes sí quedaban enlazados a un run real.

Además, incluso lo que `aipwn` **sí** puede volcar a disco (`exploit_report_<pkg>.json/.pdf`, con los
PoC confirmados/no confirmados) solo se genera `if report:` (flag `--report` del CLI) — y
`orchestrator.build_job_cmd(aipwn=True)` nunca lo pasaba. Un job de aipwn lanzado desde el dashboard
corría el `ExploitAgent` completo y tiraba el resultado: lo único que sobrevivía era el `.txt` crudo
en `logs/`.

**Fix aplicado (alcance acordado con el usuario -- mínimo, no el rediseño completo de persistencia a
SQLite):** `build_job_cmd` ahora siempre agrega `--report` para `aipwn=True`, con o sin serial. Así
el JSON/PDF de exploits queda en disco sin importar si aipwn corrió desde el dashboard o a mano.
Tests actualizados en `tests/test_orchestrator.py` (`test_build_job_cmd_aipwn_always_passes_report`).

**Pendiente, fuera de este alcance (el usuario puede pedirlo después):** que aipwn cree su propio
`run` en SQLite (kind="aipwn") con los PoC confirmados como findings, para que aparezca en el
historial del dashboard con su propio veredicto/evidencia, en vez de solo el JSON/PDF en disco.

### Hallazgo 2 (no es un bug): "hallazgos vacíos" en varias apps es real, no un fallo de guardado

Para `org.credicorp.bcpglobal`, `com.bcp.bo.discounts` y `bo.com.bcp.credinetweb`, la API sí
devuelve los `findings` que existen (verificado con `curl /api/runs/{id}` -- ninguno perdido) — pero
son genuinamente **cero**, porque:

- Ni la decompilación estática (jadx) ni el volcado runtime (Frida) produjeron código fuente
  utilizable (`decompiled/<pkg>/` y `decompiled/runtime_dump_<pkg>/` no existen para ninguna de las
  tres). `vuln.json` confirma `"files_scanned": 0`.
- `protection_broken: False` en las tres -- la protección de estas apps (bancarias reales, DexGuard/
  similar) todavía no se rompió.
- Sin código fuente, `leak_scan.native`/`gitleaks` no tienen nada sobre qué correr. Solo
  `leak_scan.apkleaks` (escanea el `.apk` crudo sin decompilar) podría encontrar algo igual, y para
  estas tres no encontró nada notable.
- Contraste: `com.bcp.bo.wallet` sí tiene 4 findings (HC004, credenciales de Firebase/Google) --
  pero vienen de `apkleaks` sobre `AndroidManifest.xml`/`strings.xml` crudos, no de código
  decompilado (`decompiled/com.bcp.bo.wallet/` existe pero con 0 archivos .java/.smali).

Es exactamente el escenario que el pipeline "protected" (runtime/Frida) existe para resolver, y lo
mismo que el usuario venía probando con `aipwn`: romper la protección primero habilita un dump/
decompile real, que a su vez habilita hallazgos de código reales en corridas futuras.

---

## Fix: colisión de directorios entre jobs estáticos concurrentes (2026-07-28)

**Reportado por el usuario:** "en el dashboard el apartado de apps no aparece los hallazgos" para
varias apps bancarias reales analizadas por la cola.

### Diagnóstico

Verificado con la API (`/api/runs/{id}`) que el dashboard no pierde nada: refleja fielmente lo que
hay en SQLite. El run problemático (`org.credicorp.bcpglobal`, 2026-07-27 19:38) realmente tenía
`files_scanned: 0` -- pero no por protección/DexGuard sin romper, como se había concluido en un
diagnóstico anterior de esta misma sesión.

**Causa real, encontrada al reproducir el pipeline en vivo:**

1. `jadx` no está instalado en este entorno (`jadx: command not found`), así que el pipeline cae al
   fallback `apktool` (sí presente, `2.7.0-dirty`). Corriendo `apktool` a mano contra el `.apk` de
   `org.credicorp.bcpglobal`, decodifica sin problema: 30,156 archivos `.smali`.
2. **El bug**: `decompiler.decompile()` nombraba el directorio de salida con `apk_path.stem` (el
   nombre del archivo, sin extensión) -- pero `apkeep` guarda el APK base de un Android App Bundle
   literalmente como `base.apk` **para cualquier paquete**. Confirmado: `bo.com.bcp.credinetweb`,
   `org.credicorp.bcpglobal` y `com.bcp.bo.discounts` se descargaron los tres como `base.apk`.
3. Esos 3 jobs estáticos se encolaron **en el mismo segundo** (19:36:22-23, con `static_workers` > 1
   en ese momento), así que los 3 procesos decompilaban **simultáneamente hacia el mismo
   `decompiled/base/`** -- `apktool --force` de un job borraba/recreaba el directorio que otro job
   estaba escaneando a mitad de camino, dejando `files_scanned: 0` para el que perdía la carrera.

Confirmado corriendo `analyze downloads/org.credicorp.bcpglobal/base.apk --static-only` de forma
aislada (sin concurrencia): decompiló bien y encontró 2 hallazgos reales (`HC004`, credenciales de
Firebase) -- quedó persistido como run #12, visible en el dashboard tras refrescar.

### Fix

- `decompiler.decompile(apk_path, output_dir, dest_name=None)`: nuevo parámetro opcional que nombra
  el subdirectorio de salida. Default `apk_path.stem` (compatible hacia atrás). `_decompile_jadx`/
  `_decompile_apktool` reciben `dest_name` en vez de derivarlo internamente.
- `orchestrator._do_decompile()` (único call site) pasa `dest_name=package` -- cada paquete
  decompila a `decompiled/<package>/`, nunca a un nombre compartido derivado del archivo.
- `decompiler.extract_manifest()` tenía el mismo riesgo (`_manifest_apktool_{apk_path.stem}`,
  `_manifest_jadx_{apk_path.stem}` -- usado en el flujo de runtime dump sin manifest). Mismo fix:
  nuevo parámetro `name_hint`, `manifest_analyzer.py` pasa `decompiled_dir.name` (ya único, incluye
  el package).

### Verificación

- 5 tests nuevos en `tests/test_decompiler.py`, incluido uno que documenta explícitamente la
  colisión tal cual se manifestaba antes del fix (`test_decompile_two_base_apks_without_dest_name_would_collide`).
  Suite completa: 220 tests pasan.
- Reproducido en vivo el bug arreglado: el run recién corrido (#12) para `org.credicorp.bcpglobal`
  ya no colisiona y persiste hallazgos reales.

---

## Fix adicional: hallazgos COMP del manifest se perdían con sast_scan:false (2026-07-28)

Al re-correr `com.bcp.bo.discounts` para verificar el fix de colisión de directorios, encontré un
segundo bug real (independiente): con `features.sast_scan: false` en `config.yaml` (la config real
del usuario), `_do_vuln_scan()` toma un camino "solo leak scan" que **retorna antes** de llegar al
bloque que inyecta hallazgos COMP004/006/007/008 (componentes exportados sin `android:permission`,
detectados desde el manifest, sin relación con SAST/semgrep). Resultado: esos hallazgos nunca se
generaban para ningún análisis mientras `sast_scan` estuviera deshabilitado -- independientemente de
si la app realmente tenía componentes exportados sin proteger.

**Fix:** la lógica de inyección se extrajo a `_inject_manifest_component_findings(scan_result)` y se
llama desde **ambos** caminos de `_do_vuln_scan` (con y sin `include_vuln_scan`). 5 tests nuevos en
`tests/test_orchestrator.py`, incluida la regresión directa (`test_do_vuln_scan_leak_only_path_still_adds_comp_findings`).
Suite completa: **225 tests pasan**.

Nota: para `com.bcp.bo.discounts` específicamente esto no cambió el resultado (sus componentes
exportados ya tenían `android:permission` seteado, y el launcher se excluye a propósito) -- pero el
gap era real y afectaba a cualquier app con componentes realmente desprotegidos mientras
`sast_scan: false`.

---

## Toolbox estático en Docker (2026-07-28)

**Pedido:** un "toolbox" que contenga las herramientas de análisis estático en Docker, con una capa
de acceso uniforme -- inspirado en la arquitectura de auto_pentest (toolbox estático en contenedor,
dinámico en el host contra el dispositivo físico real).

### Alcance de esta pasada (acotado explícitamente antes de empezar)

Implementado: el toolbox estático + la capa de acceso (`nutcracker_core/toolbox/`) + integración
en `decompiler.py` (jadx/apktool). **Deferido** (no implementado, a pedido explícito si hace falta
después): servidor MCP (`--serve-mcp`), dashboard de monitoreo separado (nutcracker ya tiene uno),
`device_pool.py`, `mitm_addon.py`, librería curada de scripts Frida -- todo eso son subsistemas
propios, no una extensión natural de "empaquetar binarios en Docker".

### Diseño

- `nutcracker_core/toolbox/client.py`: `is_enabled(config)`, `run(tool, args, config, timeout)`,
  `ensure_image()`/`image_exists()` (build local, sin depender de un registry propio).
- `nutcracker_core/toolbox/docker/Dockerfile.static`: imagen con 14 herramientas -- aapt, aapt2,
  apktool, baksmali, smali, jadx, r2 (radare2), readelf, nm, objdump, blint, gitleaks, apkid,
  apksigner.
- **Opt-in real**: `toolbox.enabled: false` (default en config.yaml/config.yaml.example) preserva
  100% el comportamiento preexistente -- cada módulo sigue invocando binarios locales vía
  `shutil.which()`. Con `true`, las mismas llamadas se enrutan a `toolbox.run()` sin cambiar su
  propia lógica (`decompiler._find_tool()` devuelve un sentinel `TOOLBOX` en vez de `None`).
- Montaje de volumen: `Path.cwd()` del host al contenedor en la **misma ruta absoluta** (todo el
  código de nutcracker opera sobre rutas relativas al proyecto, nunca fuera de él) -- evita lógica
  de traducción de rutas host↔contenedor.

### Verificación real (no solo diseño en papel)

Docker no estaba accesible al principio (WSL integration desactivada en Docker Desktop) -- se
verificó cada URL de descarga a mano (`curl`) antes de fijar versiones en el Dockerfile. Una vez el
usuario activó la integración a mitad de la tarea, se **construyó la imagen real** y se encontraron
y corrigieron dos problemas que el diseño en papel no hubiera revelado:

1. **`radare2` no está en los repos base de Ubuntu 22.04** (`E: Package 'radare2' has no
   installation candidate` -- solo disponible en `jammy-backports`, no habilitado en la imagen base
   de Docker). Fix: instalar desde el `.deb` oficial publicado en cada release de GitHub en vez de
   depender de qué repos apt tenga habilitados la imagen base.
2. **`pip3 install --break-system-packages` no existe en el pip3 de Ubuntu 22.04** (esa protección
   -- PEP 668 -- es de versiones de pip más nuevas). Fix: sacar el flag, innecesario en un
   contenedor aislado de un solo uso.
3. **Bug de permisos encontrado recién al decompilar un APK real vía el toolbox**: sin `--user
   {uid}:{gid}` en `docker run`, todo lo que el contenedor escribe queda con dueño `root` -- el
   usuario del host ni siquiera podía sobreescribir su propio output después (un `apktool --force`
   en un rerun habría fallado con "Permission denied", confirmado en vivo intentando borrar el
   resultado). Fix: `client.run()` ahora pasa `--user {os.getuid()}:{os.getgid()}`.

Tras los 3 fixes: build completo exitoso, las 14 herramientas responden dentro del contenedor
(`--version`/`-v`/`--help` según cada una), y **dos pruebas end-to-end reales** a través del propio
código de nutcracker (no solo `docker run` suelto):
- `decompiler.decompile()` con `toolbox.enabled: true` contra un APK real (`com.bcp.bo.wallet`) →
  25,706 archivos `.java` reales vía jadx, dueño correcto (UID del host).
- `decompiler._decompile_apktool()` (el camino de fallback) contra otro APK real
  (`com.bcp.bo.discounts`) → 21,939 archivos `.smali`, mismo resultado correcto.

### Tests

19 tests nuevos (`tests/test_toolbox_client.py` + los de modo toolbox en `tests/test_decompiler.py`),
todos mockeando subprocess/docker -- no requieren Docker instalado para correr en CI. Suite completa:
**244 tests pasan**.

### Pendiente si se quiere extender

- Integrar el toolbox en `native_scanner.py` (nm/objdump/readelf/radare2) y `leak_scanner.py`
  (gitleaks) -- mismo patrón, no hecho todavía.
- Los subsistemas deferidos arriba (MCP server, device pool, mitmproxy addon, frida_scripts
  curados), si el usuario los pide explícitamente.

---

## Historial de análisis por app + descarga de reportes (2026-08-03)

**Pedido:** ¿existe un histórico de evaluación por app y la posibilidad de descargar cada reporte?

No existía en la UI, pero los datos ya estaban en la base (`runs` completo por paquete vía
`repository.history()`, `artifacts` con rutas JSON/PDF vía `repository.artifacts_for_run()`) --
solo faltaba exponerlo.

### Implementado

- `store_reader.list_runs_for_app(conn, package, limit=50)`: historial completo de runs de un
  paquete, con conteo de hallazgos y artefactos (`{"json": path, "pdf": path}`) por run.
- `GET /api/apps/{package}/runs` y `GET /api/runs/{run_id}/download/{json|pdf}` (este último sirve
  el archivo real vía `FileResponse`, con 404 si el tipo es inválido, no hay artefacto registrado, o
  el archivo ya no existe en disco).
- Nueva sección "Historial de análisis" en el modal de detalle de app: fecha, veredicto, score,
  hallazgos, y botones de descarga.

### Decisión de diseño importante: el botón de PDF solo aparece en el run más reciente

El PDF es un archivo **canónico por paquete** (`reporter.py` lo sobreescribe en cada análisis, ver
`store/hooks.py::_find_artifacts` -- no genera un PDF distinto por run). Mostrar un botón "Descargar
PDF" en un run viejo daría a entender que ese archivo corresponde a *ese* análisis puntual, cuando en
realidad sería el contenido del análisis más reciente. El JSON sí es único por run (nombre con
timestamp), así que ese botón se muestra siempre que exista. El endpoint de descarga no impone esta
restricción (sirve lo que haya registrado para ese run_id, tal cual) -- la decisión de qué mostrar
vive en el frontend.

### Verificación

- 6 tests nuevos en `tests/dashboard/test_api.py`. Suite completa: 278 tests pasan (6 fallos
  preexistentes sin relación, ver sección de `dispatch_tool` más arriba).
- Verificado en vivo contra el dashboard real: `/api/apps/com.bcp.bo.wallet/runs` devuelve el
  historial real con artefactos, y `/api/runs/11/download/pdf` sirve un PDF válido (7 páginas,
  14992 bytes) directamente desde disco.

---

## Hallazgos consolidados por app + navegación a hallazgos por corrida (2026-08-03)

**Pedido:** ¿existe un consolidado de hallazgos por app de todas las corridas, y otro por corrida?

Ninguno de los dos existía. Confirmé con datos reales (com.bcp.bo.wallet, runs #10/#11 -- mismo
build re-analizado) que ambos runs tienen **hallazgos idénticos byte a byte** -- un consolidado sin
deduplicar sería puro ruido (8 filas en vez de 4 reales).

### Implementado

- `store_reader.consolidated_findings_for_app(conn, package)`: deduplica por `(rule_id, file, line)`
  a través de TODAS las corridas del paquete. Cada hallazgo distinto trae `status` ("present" si
  sigue en el run más reciente, "resolved" si ya no) + `seen_in_runs` + `first_seen_at`/`last_seen_at`.
  `repository.history()` devuelve runs más reciente primero, así que se recorre en ese orden: la
  primera vez que aparece una clave es su `last_seen`, y se empuja `first_seen` hacia atrás a medida
  que se la encuentra en runs más viejos.
- `GET /api/apps/{package}/findings` -- expone lo anterior.
- Nueva sección "Hallazgos consolidados" en el modal de app: estado (badge presente/resuelto),
  regla, severidad, ubicación, veces visto, primera/última vez.
- Historial de análisis ahora es **clickeable por fila**: `loadRunFindings(runId)` reemplaza la
  tabla "Hallazgos del último run" (rebautizada dinámicamente a "Hallazgos del run #X") por los
  hallazgos de ESE run puntual -- antes solo se podía ver el run más reciente sin forma de
  inspeccionar corridas viejas.

### Verificación

- 3 tests nuevos en `tests/dashboard/test_api.py` (dedup, marcado resolved, caso vacío). Suite
  completa: 281 tests pasan (6 fallos preexistentes sin relación, ver sección de `dispatch_tool`).
- Verificado en vivo contra datos reales: `/api/apps/com.bcp.bo.wallet/findings` devuelve
  correctamente 4 hallazgos distintos (no 8), cada uno con `seen_in_runs: 2` y `status: present`.

---

## Botón "reanudar +5 iteraciones" para sesiones de aipwn sin conclusión (2026-08-03)

**Pedido:** un botón que, cuando aipwn se queda sin iteraciones, le dé 5 iteraciones más al job y
continúe todo desde donde se quedó, recordando lo que hizo.

### Diseño

Distinto de `agent_memory.save_session()`/`load_sessions()` (ya existente): eso es un RESUMEN
(hooks que funcionaron/fallaron, clases clave) para inyectar como contexto en una sesión NUEVA.
Esto persiste la **conversación completa** (`self.messages` tal cual, con cada tool call y su
resultado) para continuar literalmente la misma sesión, no empezar de cero con un resumen.

`analysis_result`/`decompiled_dir` NO se persisten -- la CLI (`plugins/aipwn/__init__.py::aipwn_cmd`)
ya los re-deriva frescos desde disco en cada invocación, así que reanudar simplemente los vuelve a
cargar igual que una corrida normal.

### Implementado

- `agent_memory.save_resume_state()`/`load_resume_state()`/`clear_resume_state()`/`has_resume_state()`.
- `FridaAgent.run()`: 3 cortes SIN conclusión ahora guardan estado reanudable (límite de
  iteraciones, LLM sin tool_calls -- el caso real de `com.krealo.tenpo`/job 2801 -- y error de LLM
  agotando reintentos). Un `report_success`/`report_failure` real limpia cualquier sesión pendiente
  vieja: ahí sí hubo conclusión, no hay nada que continuar.
- `FridaAgent.__init__(resume_state=..., extra_iterations=...)`: si hay `resume_state`, usa
  `self.messages` guardados tal cual (no reconstruye el prompt inicial ni reinyecta memoria) y el
  tope de iteraciones queda en `iteración_guardada + extra_iterations` (no vuelve al default de
  config) -- "+5 iteraciones" son 5 de verdad, no 5 desde cero.
- `nutcracker aipwn <pkg> --resume --extra-iterations 5` (CLI) → `orchestrator.build_job_cmd(...,
  aipwn_resume=..., aipwn_extra_iterations=...)` → `QueueEngine.submit(..., aipwn_resume=...)` (nuevos
  campos en memoria en `Job`, mismo patrón que `source`: no persisten en SQLite).
- Dashboard: `GET /api/queue` marca `resumable: true` solo en el job aipwn **más reciente** de cada
  target con sesión pendiente en disco (dos jobs viejos del mismo target no se marcan ambos -- la
  sesión guardada es una sola, la de la corrida más reciente). `POST
  /api/queue/{job_id}/resume-aipwn` encola una corrida nueva con `aipwn_resume=True` y dispara el
  drenado en background (mismo `run_now` que el resto de la cola).
- Botón "▶ +5 iteraciones" en la tabla de la cola, junto al de borrar -- solo visible si el job es
  `resumable`.

### Verificación

- 20 tests nuevos (7 `agent_memory` resume, 5 `FridaAgent` resume, 8 dashboard). Suite completa: 301
  tests pasan (6 fallos preexistentes sin relación).
- Verificado en vivo contra el dashboard real: `/api/queue` devuelve `resumable: false` para el job
  `2801` (`com.krealo.tenpo`, el caso real que motivó el pedido) -- correcto, corrió *antes* de este
  fix, nunca se guardó su conversación. Cualquier corrida de aipwn que termine sin conclusión de acá
  en adelante sí quedará reanudable.

---

## Fix: RuntimeError sin manejar en los WebSockets del dashboard (2026-08-03)

**Reportado por el usuario:** traceback en el proceso del dashboard --
`RuntimeError: Unexpected ASGI message 'websocket.send', after sending 'websocket.close'` en
`ws.py::ws_job`.

### Causa

`ws_job`/`ws_chat` solo atrapaban `WebSocketDisconnect`. Un cliente que cierra la conexión justo
cuando el servidor está por mandar un evento (p.ej. `goToJobLog()` en el frontend cierra el socket
viejo y abre uno nuevo al hacer clic en otra fila del historial de análisis, feature agregada esta
misma sesión) no siempre produce `WebSocketDisconnect` -- a veces Starlette/uvicorn ya cerró el
transporte del lado ASGI y levanta un `RuntimeError` genérico al intentar mandar después. Sin
atraparlo, tumbaba el `await self.app(...)` de uvicorn con un traceback completo en la consola del
dashboard (aunque no mataba el proceso -- uvicorn aísla cada conexión).

### Fix

`ws_job` y `ws_chat` ahora atrapan `(WebSocketDisconnect, RuntimeError)` en vez de solo
`WebSocketDisconnect`. De paso, en `ws_job` se movió `bus.subscribe()` a ANTES del `try` (en vez de
después del replay de historial) -- así `bus.unsubscribe()` en el `finally` corre siempre, incluso si
el primer `send_json()` del replay ya falla.

### Verificación

`TestClient.websocket_connect()` no reproduce esta carrera de forma determinística (es un timing real
de la capa ASGI/transporte) -- los tests nuevos invocan `ws_job`/`ws_chat` directo con un WebSocket
falso cuyo `send_json()` levanta el `RuntimeError` real visto en producción, confirmando que ya no
propaga y que `bus.unsubscribe()` sigue corriendo. 4 tests nuevos en `tests/dashboard/test_ws.py`.
Suite completa: 304 tests pasan (6 fallos preexistentes sin relación).

---

## Fix: la VM de WSL2 se caía entera al correr jadx vía el toolbox de Docker (2026-08-03)

**Reportado por el usuario:** tras activar `toolbox.enabled: true` (para que aipwn dejara de estar
ciego con `com.krealo.tenpo` -- ver fix anterior sobre `.smali` vs `.java`), el sistema entero se
congelaba/reiniciaba corriendo `jadx --deobf` contra ese APK vía Docker, tumbando el dashboard y
dejando el job huérfano (`running` en SQLite, sin proceso vivo). Pasó dos veces.

### Causa

Docker Desktop en modo "WSL2 based engine" (confirmado vía `dmesg`:
`docker-desktop-user-distro proxy --distro-name Ubuntu`) comparte el MISMO pool de memoria de la VM
de WSL2 que la distro principal donde corre nutcracker -- no son VMs independientes. Sin
`.wslconfig`, ese pool es ~50% de la RAM del host (7.7GB en esta máquina de 16GB). El contenedor de
`toolbox/client.py::run()` no tenía ningún límite propio de memoria (`docker run` sin `--memory`), así
que un `jadx --deobf` desbocado contra un APK grande/ofuscado podía consumir memoria sin techo hasta
agotar TODO ese pool compartido -- llevándose puesto con él al dashboard y cualquier otro job, en vez
de fallar solo el contenedor.

Primer intento de fix (`/mnt/c/Users/Hitee/.wslconfig` con `memory=10GB, swap=4GB`, el host tiene
16GB) subió el techo compartido -- verificado en vivo (`docker info` "Total Memory" pasó de 7.695GiB
a 9.713GiB en lockstep con el cambio) -- pero no fue suficiente: la demanda real de `jadx --deobf`
contra ese APK en particular superó igual el nuevo techo, esta vez agotando el swap (`dmesg`: `Free
swap = 0kB` repetido) a los ~6 minutos de un boot limpio de WSL2.

### Fix

El fix de fondo era el que faltaba: ponerle techo propio al contenedor, no solo al pool compartido.
`toolbox/client.py::run()` ahora agrega `--memory {limit} --memory-swap {limit}` al `docker run` (la
misma cifra en ambos flags le niega swap adicional al contenedor: si se pasa del límite, el kernel lo
mata rápido por su propio cgroup en vez de dejarlo entrar en swap thrashing, que es justamente lo que
pone lenta/inestable a toda la VM antes de que ocurra ningún OOM-kill real). Límite configurable vía
`toolbox.memory_limit` en `config.yaml` (default `'4g'` -- dado el pool actual de ~9.7GB, deja margen
para el propio proceso de nutcracker + el resto del sistema, y el contenedor ya no puede por sí solo
agotar todo el pool).

### Verificación

3 tests nuevos en `tests/test_toolbox_client.py` (`memory_limit()` default/override, `--memory`/
`--memory-swap` en el comando `docker run` construido, override real vía config). Jobs huérfanos por
los crashes anteriores (2804, 2805, 2806) limpiados manualmente a `status='error'`. Confirmado en vivo
tras el fix: sin procesos ni jobs `running` colgados en el sistema.

---

## Fix: mensaje con JSON truncado quedaba pegado en el historial del LLM para siempre (2026-08-03)

**Reportado por el usuario:** log del job `2809` (`com.krealo.tenpo`) -- el agente diagnosticaba bien
el bypass (classloader-level hooking de `HSTSecNative`/`HSTSecurity`), pero al llamar al LLM en la
iteración 23 el proveedor (Azure) devolvía 400 `"Assistant tool call function.arguments must be valid
JSON"` -- **idéntico en los 3 reintentos**, matando el job entero (`✘ Bypass failed`).

### Causa

Relacionado con el fix anterior del `KeyError: 'script_js'`, pero más de fondo: en
`LLMClient._do_completion()` (`nutcracker_core/plugins/aipwn/frida_agent.py`), cuando el modelo
devuelve JSON truncado para un tool call, el `try/except json.JSONDecodeError` (líneas ~464-468) SÍ
corregía el valor a `{}` -- pero solo para el `_ToolCall` usado en el despacho local. El
`raw_message` que se guarda en `self.messages` (el historial que se reenvía en cada llamada
siguiente) releía `tc.function.arguments` **crudo, directo del objeto del SDK**, en un bloque de
código totalmente separado que nunca pasaba por ese try/except. El fix del `KeyError` (que atrapa el
crash al despachar la tool) no tocaba esto -- el mensaje del asistente con el JSON roto ya había
quedado guardado en el historial ANTES de que `dispatch_tool` corriera. A partir de ahí, cada
`self.llm.chat(self.messages, ...)` reenviaba ese mismo mensaje estructuralmente inválido, y Azure lo
rechazaba siempre igual -- no es un error transitorio, así que reintentar no ayuda, y el resto de la
sesión (todo lo que el agente ya había descubierto sobre `com.krealo.tenpo`) se perdía.

### Fix

`_do_completion()` ahora sanea el `arguments` de cada tool call UNA sola vez (dict `sanitized_args_json:
{tc.id: json_string_válido}`, `"{}"` si falló el parseo) y ese mismo valor se usa tanto para construir
el `_ToolCall` de despacho local como para el `raw_message` que se guarda en `self.messages` -- las dos
lecturas independientes de `tc.function.arguments` que antes podían divergir ahora comparten una sola
fuente de verdad. El JSON truncado del modelo nunca vuelve a persistir en el historial.

### Verificación

3 tests nuevos en `tests/test_aipwn_llm_client_tool_call_json.py`: JSON válido queda intacto, JSON
truncado se sanea a `{}` tanto en el historial como en el despacho local (y ambos coinciden). Suite
completa: 309 tests pasan (los mismos 6 fallos preexistentes sin relación, ver fix anterior de
`.smali`/toolbox).

---

## Feature: relay "browser-as-bridge" -- frida/adb desde un backend remoto (M1, 2026-08-04)

**Motivación:** el usuario quiere nutcracker viviendo en la web, accedido por cualquiera desde su
propia máquina, con el requisito de tener un dispositivo rooteado conectado a esa máquina. Un
servidor remoto no puede alcanzar por sí solo un USB enchufado en la PC de otra persona -- hace
falta que algo del lado del usuario haga de puente. Investigando alternativas (reFrida, frida-ui,
Frida-Script-Runner) se confirmó que frida SÍ se puede hablar desde el navegador (reFrida conecta
directo a frida-server por WebSocket), lo cual reabrió la arquitectura: el navegador puede ser ese
puente, reusando el handle `Adb` (Tango/ya-webadb) que el dashboard YA tiene para el video WebUSB.

Decisión de diseño (túneles TCP crudos, ver plan completo en el historial de sesión): `frida`/`adb`
reales siguen corriendo en el backend sin cambios de lógica -- se les apunta a listeners TCP locales
que un tunnel manager nuevo multiplexa por WebSocket hacia el navegador, que reenvía los bytes al
device real vía `adb.createSocket("tcp:PORT")`. Como el `serial` resuelto (`127.0.0.1:PA`) es
indistinguible de un serial de red real (`ip:5555`) para el resto del código (`adb_transport.py`,
`queue/engine.py`), TODA la maquinaria existente de adb-over-network (`ensure_available`,
`TransportKeepAlive`, `_device_lock`) funciona sin tocarla.

### Componentes nuevos

- **`nutcracker_core/plugins/dashboard/relay.py`** (nuevo): `RelaySession` abre un listener TCP
  loopback por túnel (`frida`, `adb`; puerto efímero, `port=0`) y multiplexa cada conexión aceptada
  como un "canal" (`conn_id`) dentro de una sola WebSocket -- necesario porque frida abre varias
  conexiones concurrentes contra frida-server, un socket 1:1 no alcanza. Protocolo: control en JSON
  (`open`/`close`) + datos en frames binarios con header de 4 bytes big-endian = `conn_id`.
  `RelayManager` registra sesiones por `session_id` (hoy: elegido libremente por el operador, ata la
  conexión del navegador con los jobs que la usan).
- **`plugins/dashboard/ws.py`**: nuevo endpoint `/ws/relay/{session_id}` -- el navegador se conecta
  acá; usa `websocket.receive()` de bajo nivel (no `receive_text`/`receive_bytes`) porque mezcla
  ambos tipos de frame en el mismo socket. Mismo manejo de `(WebSocketDisconnect, RuntimeError)` que
  `ws_job`/`ws_chat`.
- **`plugins/dashboard/api.py`**: `GET /api/relay/{session_id}` (estado + puertos). `QueuePayload`
  gana `relay: bool`; `queue_add` con `relay=True` trata `serial` como el `session_id` de una sesión
  YA conectada, la resuelve a `serial=127.0.0.1:PA` / `frida_host=127.0.0.1:PF`, y rechaza con 400 si
  no hay navegador conectado ("no hay un navegador con el relay conectado para '<serial>'...").
- **`queue/job.py`/`queue/engine.py`**: `Job.frida_host` (en memoria, mismo patrón que `source`);
  `submit(frida_host=...)`; `_run_job` inyecta `NUTCRACKER_FRIDA_HOST` al env del subproceso si está
  seteado. El engine queda deliberadamente ciego al relay -- solo transporta el valor ya resuelto por
  `api.py` (capa correcta: `queue/` es core, `relay.py` vive en el plugin dashboard).
- **`plugins/aipwn/aipwn.py`**: `_frida_host` ahora prioriza `NUTCRACKER_FRIDA_HOST` (env) sobre
  `strategies.frida_host` (config) -- apunta frida al túnel de la corrida en vez del host fijo.
- **`webusb/src/relay.ts`** (nuevo): `RelayClient` -- consume `/ws/relay/{session_id}`, reusa el
  MISMO handle `Adb` que el video (`getAdb()`, nuevo export en `main.ts`; abrir un segundo handle
  pisaría al del video, WebUSB es exclusivo por interfaz), abre `adb.createSocket("tcp:27042"|"tcp:5555")`
  por cada canal que el backend anuncia, y bombea bytes en ambas direcciones. Re-exportado desde
  `main.ts` para que `vite build` (modo lib, un solo entry) lo incluya en `webusb-video.bundle.js`.
- **`static/index.html`**: sección "Relay" en la pestaña Dispositivo -- input de session id +
  botones activar/detener, reusando la conexión USB ya activa.

### Estado de verificación

- **Backend: probado de verdad.** `tests/dashboard/test_relay.py` (13 tests, TCP loopback real, sin
  mocks de socket) + `tests/dashboard/test_ws.py` (3 tests nuevos, integración completa vía
  `TestClient` -- conexión TCP real contra el puerto anunciado, WebSocket real, round-trip de bytes
  en ambas direcciones) + `tests/dashboard/test_api.py` (6 tests, gating de `queue_add`) +
  `tests/test_queue_engine.py` (2 tests, inyección de env) + `tests/test_aipwn_relay_frida_host.py`
  (4 tests, precedencia env > config > None). Suite completa: 337 tests pasan (los mismos 6 fallos
  preexistentes sin relación).
- **Frontend: compila y tipa correcto, NO verificado contra hardware.** `tsc --noEmit` limpio contra
  los `.d.ts` reales de Tango (`AdbSocket`, `MaybeConsumable`); `vite build` genera el bundle con
  `RelayClient`/`getAdb` exportados. Pendiente: probar `adb.createSocket()` contra un device físico
  real (M1 de plan.md dice explícitamente "bootstrap manual" -- frida-server y `adb tcpip 5555` los
  arranca el operador a mano por ahora, sin automatizar).

### Pendiente (fuera de este cambio, ver plan.md completo)

M2 (túnel adb en un run real end-to-end), M3 (bootstrap automático de frida-server/tcpip desde el
navegador), M4 (reconexión/hardening). Multi-tenant (auth, aislamiento, API key por usuario)
explícitamente diferido -- decisión del usuario, single-user primero.

---

## Fix de diseño: el túnel TCP crudo para adb no es viable -- pivot a RPC (M2, 2026-08-04)

**Contexto:** probando M1 en vivo contra hardware real (deploy en 0.0.0.0 + mirrored networking +
relay), el túnel de **frida** funcionó pero el de **adb** daba siempre `device offline` al instalar
el APK. Diagnóstico en vivo, aislado del resto del pipeline (snippet directo en la consola del
navegador contra `adb.createSocket("tcp:5555")`): la conexión resuelve OK (el device acepta el OPEN)
pero después no fluye un solo byte -- ni error, ni cierre, silencio total, incluso escribiendo datos
primero. Conclusión: **Android bloquea reenviar tráfico `tcp:` hacia el propio puerto de control de
adbd (5555)** -- evita que una sesión adb ya autorizada se use para colarse al canal de control
completo sin la RSA key de una PC autorizada. No es un bug del mecanismo `tcp:` en general: scrcpy
(video, ya funcionando) usa exactamente el mismo mecanismo contra el puerto de scrcpy-server sin
problema, porque ese es un proceso de terceros, no adbd. Mismo motivo por el que frida (27042) sí
funciona -- frida-server tampoco es adbd.

### Decisión

Túnel TCP crudo se mantiene tal cual para **frida** (funciona, no se toca). Para **adb** se reemplaza
por **RPC estructurado sobre la misma WebSocket**: el navegador resuelve las operaciones con los
métodos NATIVOS de Tango (`adb.subprocess.spawnAndWait()`, `adb.sync()`) en vez de un socket crudo --
confirmado que no es una técnica nueva sin probar: es el mismo mecanismo que ya usa scrcpy y que usa
`app.webadb.com` (construida sobre la misma librería ya-webadb/Tango) para shell/install/pull.

Alcance por etapas (decisión explícita del usuario: "por etapas, empezando por shell" en vez de las
4 piezas juntas, para poder aislar fallas una por una como pasó con el túnel crudo). **Etapa 1 (esta
sesión): solo `shell`.** `install`/`pull`/`push`/`exec-out`/`logcat` quedan pendientes de etapas
siguientes -- fallan explícito con mensaje claro en vez de colgarse o dar "device offline" confuso.

### Componentes nuevos

- **`relay.py`**: `RelaySession.rpc(op, **fields)` -- manda `{"type":"rpc_request","request_id":N,...}`
  y espera `{"type":"rpc_response","request_id":N,...}` correlacionado (dict de Futures por
  request_id). `RpcTimeoutError`/`RpcError`/`RelayError` distinguen timeout, fallo reportado por el
  navegador, y sesión no conectada. `detach_websocket()`/`stop()` fallan cualquier RPC pendiente en
  vuelo en vez de dejarlo colgado hasta agotar su timeout completo.
- **`api.py`**: `POST /api/relay/{session_id}/rpc/shell` (`async def`, awaits `session.rpc(...)`) --
  mapea `RpcTimeoutError→504`, `RpcError→502`, `RelayError→409`.
- **`relay.ts`**: `handleRpcRequest()` -- op `"shell"` resuelve con `adb.subprocess.spawnAndWait(command)`
  (API real de Tango: `{stdout, stderr, exitCode}`), responde con `rpc_response`.
- **`toolbox/relay_adb_shim/adb`** (nuevo, ejecutable con shebang): intercepta CUALQUIER invocación
  de "adb" del resto del código sin tocar esos call sites -- `engine.py::_run_job` antepone su
  directorio al `PATH` del subproceso cuando `job.relay_session_id` está seteado, así todo lo que
  resuelva "adb" vía `shutil.which()` (aipwn.py, frida_capture.py, frida_agent_tools.py) lo recoge
  automático. Traduce `adb -s <ignorado> shell <cmd...>` a un POST a
  `/api/relay/{NUTCRACKER_RELAY_SESSION_ID}/rpc/shell`, junta los tokens con espacios (mismo
  comportamiento que adb real), y propaga stdout/stderr/exit_code tal cual. Cualquier subcomando que
  no sea `shell` falla explícito (exit code 2, mensaje claro) -- no silencioso.
- **`queue/job.py`**: `Job.relay_session_id` -- a diferencia de como se diseñó `frida_host`/`serial`
  originalmente, ACÁ `serial` deja de reescribirse a una dirección de loopback (`api.py::queue_add`
  ya no lo hace) -- queda como el session_id tal cual, porque el shim lo ignora igual y así
  `is_network_serial()` no lo confunde con un serial de red real.
- **`queue/engine.py`**: `_ensure_transport()` gana un skip explícito para `job.relay_session_id`
  (doble resguardo, por si el operador elige un session_id con forma `ip:puerto`). `_run_job` inyecta
  `NUTCRACKER_RELAY_SESSION_ID` + antepone `_RELAY_ADB_SHIM_DIR` al `PATH`.

### Verificación

23 tests nuevos: `test_relay.py` (RPC round-trip, timeout, error del navegador, desconexión a mitad
de request, dos RPCs concurrentes con request_ids independientes), `test_api.py` (mapeo de status
codes del endpoint, con un doble de sesión -- una WebSocket real + POST concurrente desde otro hilo
choca con una limitación conocida del TestClient de Starlette, deadlock del harness confirmado en
vivo, no del código real), `test_queue_engine.py` (env vars + PATH, skip de `_ensure_transport`),
`test_relay_adb_shim.py` (el shim como subproceso real contra un servidor HTTP de juguete: round-trip,
exit code no-cero, sin `-s`, comando de un solo token pre-armado sin romper, comando no soportado
falla explícito, env vars faltantes, error HTTP del dashboard, dashboard inalcanzable). Suite completa:
360 tests pasan (los mismos 6 fallos preexistentes sin relación).

**Pendiente de verificación en vivo** (requiere al usuario, con hardware real): la Etapa 1 (`shell`)
compila y pasa todos los tests locales, pero todavía no se probó contra un device físico end-to-end.

---

## Verificación en vivo del relay contra hardware real (job 2819, 2026-08-04)

**Resultado: parcialmente exitoso.** Primera corrida end-to-end de aipwn completa a través del relay
(dashboard en 0.0.0.0 + mirrored networking + WebUSB en una PC distinta a la del backend):

- ✅ **Túnel de frida vía CLI** (`frida -H 127.0.0.1:PF -f pkg -l script.js`, engine A): conectó,
  spawneó la app, corrió el script, capturó resultado -- funciona end-to-end contra un device real.
- ✅ **Shell RPC (Etapa 1)**: `check_app_installed`, lectura de clases decompiladas, y demás
  operaciones vía el shim funcionaron sin errores de transporte durante toda la sesión.
- ✅ **Fallo explícito para comandos no soportados**: `logcat`/`pull` (no implementados todavía)
  fallaron con el mensaje claro diseñado ("comando 'X' todavía no soportado vía relay"), sin colgarse
  ni dar el "device offline" confuso de antes -- el agente LLM los interpretó correctamente como una
  limitación del entorno y siguió adaptando su estrategia.
- ✅ El fix de JSON truncado de una sesión anterior también se vio funcionar en vivo: un script
  truncado por el LLM se manejó sin crashear, el agente reintentó con uno más corto.
- ❌ **Deadlock confirmado en `run_frida_script(spawn_gated=True)`** (engine B, bindings de Python de
  frida -- `device.spawn()`/`device.attach()`, NO la CLI): la app real (`com.krealo.tenpo`) tiene RASP
  nativo agresivo (`libh5t5cr7.so`, DexGuard/HST) que mata el proceso antes de que los hooks Java
  instalen, así que el agente escaló a `spawn_gated=true` como estrategia -- y esa corrida se colgó
  indefinidamente (16+ min sin progreso, 0% CPU, conexión TCP establecida con colas tx/rx vacías y sin
  cambios -- deadlock de aplicación confirmado, no transferencia lenta). Es la PRIMERA vez que se
  ejercita este camino contra el relay; la CLI (misma sesión, mismo target) funcionó bien.

### Causa raíz: NO CONFIRMADA

Diagnóstico intentado en vivo: `py-spy dump --pid <pid>` para un stack trace real de Python -- bloqueado
por `ptrace_scope=1` del kernel, requiere `sudo` con password interactiva no disponible en este entorno.
Sin eso, no hay evidencia directa de en qué línea exacta está bloqueado. Revisión de `relay.py` buscando
un bug obvio de multiplexado para una segunda conexión concurrente no encontró nada evidente, pero eso
no descarta que el problema esté ahí -- podría también ser un patrón de conexión distinto del lado de los
bindings de frida-python que el relay no maneja igual que la CLI.

**Decisión: no aplicar un fix especulativo sin evidencia** (mismo error que casi se repite del episodio
de `tcp:5555` -- ahí sí hubo verificación en vivo antes de declarar la causa; acá todavía no la hay).
Job 2819 matado manualmente y marcado `error` en la DB con el diagnóstico completo.

### Próximo paso (cuando se retome)

Conseguir un stack trace real la próxima vez que se reproduzca -- con `sudo` disponible para `py-spy`,
o alternativa: correr aipwn con `FRIDA_DEBUG`/logging verboso de frida-python para ver en qué llamada de
red exacta se cuelga. Mientras tanto: **usar el camino CLI (sin `spawn_gated=true`) como el confiable vía
relay** -- es el que cubre la mayoría de los casos; `spawn_gated` es una estrategia de escalada del
agente para RASP nativo agresivo específicamente, no el camino principal.

**Actualización -- segunda reproducción idéntica (job 2820, mismo día):** el job 2820
(`sh.nutcracker.nutbank`, app distinta a la del 2819) llegó también a `run_frida_script(spawn_gated=True)`
por primera vez en esa corrida (diagnóstico del agente sobre si los hooks nativos disparaban el RASP) y
se colgó exactamente igual -- conexión TCP establecida, colas tx/rx vacías y sin cambios, 0% CPU, 8+ min
sin progreso. Mismo patrón: es la PRIMERA vez que ESA corrida usa `spawn_gated`, igual que en el 2819.
Esto sube la confianza de que el problema es **sistemático del camino spawn_gated/bindings vs. el
relay**, no una casualidad de una app en particular. Matado manualmente y marcado `error` en la DB. Sigue
sin causa raíz confirmada (mismo bloqueo de `py-spy`/`sudo`) -- pero con 2/2 reproducciones limpias, es
la pieza de mayor prioridad para la próxima sesión con acceso a mejores herramientas de diagnóstico.

---

## Etapas 2-4 del relay: install, pull, screencap (2026-08-04)

Continuación directa de la Etapa 1 (shell), disparada por un caso real en vivo: el job 2820
(`sh.nutcracker.nutbank`) necesitaba `pull_apk_from_device` para inspeccionar librerías nativas
sospechosas de RASP, y el agente vio explícito el mensaje "comando 'pull' todavía no soportado" -- la
señal de que tocaba seguir con el plan por etapas.

### Diseño

Mismo patrón que shell: RPC estructurado sobre `session.rpc(op, **fields)`, resuelto en el navegador con
métodos NATIVOS de Tango -- nunca el túnel TCP crudo (que ya sabemos bloqueado para el puerto de control
de adbd).

- **`install`/`install-multiple`**: el shim lee el/los APK(s) local(es), los manda en base64 en un solo
  `rpc_request`. El navegador sube cada uno a `/data/local/tmp/<nombre>` con `adb.sync().write()` (el
  mismo método que usa `AdbScrcpyClient.pushServer` para subir el propio server de scrcpy -- no una
  técnica nueva), corre `pm install[-multiple] <flags> <rutas>` con `adb.subprocess.spawnAndWait()`, y
  borra los temporales (best-effort, un fallo de cleanup no debe tumbar una instalación ya exitosa).
- **`pull`**: el navegador lee el archivo remoto con `adb.sync().read()`, lo manda entero en base64 en
  un solo `rpc_response` (sin streaming por chunks -- explícitamente diferido, ver "Pendiente" abajo). El
  shim lo escribe en la ruta local que pidió el caller.
- **`screencap`** (`exec-out screencap -p`, único uso real de `exec-out` en el codebase -- no se
  generalizó a comandos arbitrarios): usa `adb.subprocess.spawn()` (NO `spawnAndWait`, que decodifica
  stdout como texto UTF-8 y corrompería el PNG binario) para leer el stream crudo de bytes. El shim
  escribe con `sys.stdout.buffer.write()`, no `sys.stdout.write()`, por el mismo motivo.
- Helpers nuevos en `relay.ts`: `bytesToBase64`/`base64ToBytes` (btoa/atob operan sobre "binary strings",
  no directo sobre `Uint8Array`, de ahí la conversión manual troceada en chunks de 0x8000 para no reventar
  el límite de argumentos de `String.fromCharCode(...arr)` con archivos grandes), `readAllBytes` (junta
  un `ReadableStream<Uint8Array>` entero en memoria).
- `api.py` ganó `_require_attached_relay_session()`/`_run_relay_rpc()` -- el chequeo de sesión conectada
  y el mapeo de errores (`RpcTimeoutError→504`, `RpcError→502`, `RelayError→409`) estaban duplicados
  4 veces, factorizado a dos helpers reusados por los 4 endpoints RPC.

### Verificación

14 tests nuevos: `test_relay_adb_shim.py` (install single/multiple APK con verificación de base64
correcto, archivo local faltante falla antes de contactar al dashboard, exit code no-cero propagado,
pull escribe bytes exactos en el path local, pull sin argumentos suficientes falla explícito, exec-out
screencap escribe bytes binarios crudos a stdout -- incluyendo bytes no-UTF8 a propósito para probar que
no se corrompen -- exec-out con comando no soportado sigue fallando explícito), `test_api.py` (6 tests,
mapeo de status codes de los 3 endpoints nuevos, mismo patrón de doble de sesión que shell). Frontend:
`tsc --noEmit` limpio contra los tipos reales de `AdbSync`/`AdbSubprocess` (tuvo que ajustarse a los tipos
propios de `@yume-chan/stream-extra`, distintos del `ReadableStream` global del navegador aunque
estructuralmente parecidos), `vite build` genera el bundle actualizado. Suite completa: 374 tests pasan
(los mismos 6 fallos preexistentes sin relación).

**Pendiente de verificación en vivo**: igual que la Etapa 1 en su momento, este código compila y pasa
tests locales pero install/pull/screencap todavía NO se probaron contra hardware real -- pull en
particular es justo lo que el job 2820 necesitaba, buen candidato para la próxima verificación en vivo.

### Fuera de alcance de esta pasada (deferred a propósito)

- **`push`/`logcat`** (streaming continuo): no encajan en el modelo pedido-respuesta de `rpc()` tal como
  está. `logcat` en particular sigue fallando explícito -- el agente ya demostró en el job 2820 que lo
  maneja bien sin romperse.
- **Streaming real de archivos grandes** para pull/install: hoy todo el archivo se junta en memoria y se
  manda en un solo mensaje base64. Suficiente para APKs/librerías nativas típicas (cientos de KB a pocos
  MB); un archivo genuinamente grande sería una optimización a futuro, no bloqueante ahora.

---

## FIX RAÍZ: el túnel de frida vía Tango no sostenía el handshake WebSocket -- resuelto con un bridge WS nativo (2026-08-04)

**El hallazgo más importante de toda esta sesión de relay.** Todo lo anterior (M1, la verificación "en
vivo" del job 2812/2815/2819) daba por buena la CLI de frida por túnel crudo (`adb.createSocket("tcp:27042")`)
solo porque el log mostraba "Connected"/"Spawning" -- **nunca se verificó que algo real volviera del
device**. El usuario notó, mirando la pantalla real del teléfono, que ninguna app cambiaba de
comportamiento pese a esos logs "exitosos". Eso disparó una investigación a fondo que terminó en un
hallazgo real, no una sospecha.

### Diagnóstico (instrumentación byte-a-byte, no conjetura)

Se agregó logging temporal en `relay.py` (`_debug_log`, gateado tras `NUTCRACKER_RELAY_DEBUG=1`) para
ver exactamente qué bytes cruzan el túnel en cada dirección. Con eso, un test aislado (`frida -H
127.0.0.1:PF -f pkg -l script.js` con un script de dos líneas, sin aipwn de por medio) mostró:

```
conn=1 tunnel=frida local->ws 177 bytes: b'GET /ws HTTP/1.1\r\nUpgrade: websocket\r\nConnection: Upgrade\r\nSec-W...'
13ms después: navegador reporta 'closed'
```

**Frida moderno (17.x) no habla el protocolo TCP crudo histórico de frida-server -- habla WebSocket.**
El CLI arma a mano un `GET /ws HTTP/1.1` con upgrade a WebSocket, y después manda/espera frames WS
crudos por el mismo socket TCP. El túnel entregaba esos 177 bytes PERFECTOS al device (confirmado
byte a byte) -- el problema es que `adb.createSocket("tcp:27042")` (Tango) no sostiene la conexión más
de ~13ms después de ese handshake. No es el bug de `tcp:5555`/adbd (puerto de control) -- 27042 es
frida-server, un proceso de terceros normal; es específico de cómo Tango maneja una conexión que
arranca con un upgrade HTTP/WebSocket.

Se descartó "no hay forma de aislar esto" con dos tests aislados más, directo en la consola del
navegador contra `frida-server`:
1. `adb.createSocket("tcp:27042")` + escribir bytes de prueba → mismo patrón (conecta, después
   silencio total) -- confirma que el bug es de Tango, no de aipwn ni de mi relay.py.
2. `new WebSocket("ws://192.168.1.42:27042/ws")` (nativo del navegador, **sin Tango, directo por la
   LAN**) → conecta y se **mantiene estable** -- confirma que frida-server funciona bien, el problema
   es específicamente el camino Tango/ADB.

El usuario aportó la pista decisiva: **reFrida** (la IDE web de referencia investigada al principio de
esta sesión) nunca usa Tango/ADB para frida -- se conecta con un `WebSocket` nativo del navegador
directo a la IP de LAN del device (`ws://192.168.1.42:27042`), y el navegador pide el permiso "Acceso a
Red Local" (Private Network Access de Chrome) la primera vez. Eso es justo lo que el test aislado #2
confirmó que funciona.

### Fix: bridge de protocolo WS crudo <-> WebSocket nativo (`relay.ts`)

No alcanza con "usar `WebSocket` nativo en vez de `createSocket`" -- `frida` habla su protocolo WS A
MANO sobre bytes crudos (arma el handshake HTTP él mismo, después manda/espera frames WS crudos),
mientras que el objeto `WebSocket` del navegador hace SU PROPIO handshake y framing internamente, sin
exponer bytes crudos. Hace falta un traductor completo (RFC 6455, subset) en el medio:

- `_FridaWsChannel` (nuevo, junto a `_TangoChannel` -- `_Channel` es ahora una unión discriminada por
  `kind`): NO abre el `WebSocket` nativo apenas se anuncia el canal -- espera a ver los bytes crudos del
  `GET /ws` que arma `frida`, le extrae el header `Sec-WebSocket-Key`.
- `completeFridaWsHandshake`: con esa key, calcula `Sec-WebSocket-Accept` (`base64(SHA1(key +
  "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"))`, vía `crypto.subtle.digest("SHA-1", ...)` nativo del
  navegador), abre el `WebSocket` nativo real hacia `ws://<ip-lan-device>:27042/ws`, y una vez que ESE
  conecta, sintetiza a mano la respuesta HTTP 101 que `frida` espera recibir de vuelta por el canal
  crudo -- como si viniera de un frida-server real.
- De ahí en más: cada frame WS que manda `frida` (parseado con `parseClientWsFrame` -- enmascarado, por
  spec de cliente) se desenmascara y su payload se manda por el `WebSocket` nativo
  (`ch.ws.send(payload)`); cada mensaje que llega del `WebSocket` nativo (`onmessage`) se re-empaqueta
  como frame WS de servidor (`buildServerWsFrame` -- sin máscara, por spec de servidor) y se manda como
  bytes crudos de vuelta a `frida` por el túnel. Pings de `frida` se responden con pong directo (no hace
  falta ida y vuelta al device).
- Requiere que el navegador tenga alcance de RED (no solo USB) al device -- input nuevo "IP LAN del
  device" en la sección Relay del dashboard (`index.html`), que arma `fridaWsUrl` y se lo pasa al
  constructor de `RelayClient`. El túnel de adb (shell/install/pull/screencap, vía Tango) no cambia --
  ese no tiene este problema.
- Limitación documentada: no reensambla frames de continuación (fragmentación WS) -- no se vio hacer
  falta en las pruebas en vivo, los mensajes de control/RPC de frida son chicos.

### Verificación: end-to-end contra hardware real, primera prueba

El mismo test aislado (`frida -H 127.0.0.1:PF -f sh.nutcracker.nutbank -l script.js`, script de dos
líneas) que antes se colgaba en silencio, con el bridge nuevo:

```
[RELAY-TEST] script cargado y ejecutando en el device - 2026-08-04T18:21:13.059Z
Spawned `sh.nutcracker.nutbank`. Resuming main thread!
[RELAY-TEST] Java.perform disparo correctamente
```

Confirmado además con una captura de pantalla real (vía el screencap RPC de la Etapa 3, primera vez que
se usa en vivo también): la app "Nut Bank" corriendo de verdad en el device, mostrando SU PROPIO diálogo
"Security Check Failed -- Dynamic instrumentation detected (Frida)" -- prueba visual directa de que la
instrumentación llegó, se adjuntó, y la app la detectó. Es el mismo diálogo que el agente LLM del job
2820 estaba razonando cómo bypassear -- ahora, con el transporte realmente funcionando, esas estrategias
tienen una oportunidad real de funcionar.

### Pendiente

- No hay tests automatizados del parser/builder de frames WS (`parseClientWsFrame`/`buildServerWsFrame`)
  -- no existe infraestructura de test para TS en `webusb/` (sin vitest/jest configurado). Validado por
  ahora con `tsc --noEmit` + revisión manual del RFC 6455 + el test end-to-end en vivo de arriba. Si se
  toca este código de nuevo, vale la pena montar vitest.
- El deadlock de `spawn_gated` (bindings de Python, ver sección anterior) sigue sin retestear con el
  bridge WS nuevo -- dado que el bug real estaba en el túnel crudo de Tango (que también usaba el camino
  CLI, no solo bindings), es muy posible que este mismo fix lo resuelva también. Buen próximo test.
- IP de LAN del device hardcodeada a mano por el operador en el dashboard -- podría autodetectarse
  (`adb shell ip route`/`getprop` vía el shell RPC que ya funciona) en una iteración futura.

---

## Puntos de mejora encontrados revisando los jobs 2822/2823 (2026-08-04)

Revisión post-mortem de la primera corrida exitosa de aipwn vía relay (job 2823, bypass completo de
`sh.nutcracker.nutbank` confirmado con screenshot real). El bypass funcionó, pero el log mostraba varias
ineficiencias -- 6 fixes aplicados:

### 1. [ALTO] El paso "probar script previo" no pasaba `frida_host`

`aipwn.py` Paso 1 (`launch_frida_capture()` para testear el script cacheado de una sesión anterior)
pasaba `serial=serial` pero no `frida_host=_frida_host` -- con relay activo, caía a `frida -D <serial>`
(busca device por USB) en vez de `-H 127.0.0.1:PF`, y fallaba siempre con "Device not found" pese a que
el script cacheado hubiera funcionado. En el job 2823 esto le costó al agente 13 iteraciones de LLM
re-explorando desde cero en vez de 0. Fix de una línea + test (`tests/test_aipwn_relay_frida_host.py`).

### 2. [BAJO] `%detach` incompatible con frida 17.x

`frida_capture.py` mandaba el meta-comando viejo `%detach\n` a la REPL para desconectarse limpio -- frida
17.16.4 ya no lo reconoce ("Unknown command: detach"), y el error se tragaba en silencio, cayendo siempre
al `terminate()`/`kill()` de más abajo (funcional pero no realmente "graceful"). El propio banner de la
CLI documenta `exit/quit -> Exit` como el comando real -- reemplazado.

### 3. [BAJO] 4 hooks GMS fallaban siempre en emuladores AOSP (ruido idéntico repetido)

`frida_bypass.py` (`_HOOK_GOOGLE_PLAY`) intentaba `Java.use()` sobre 4 clases de Google Play Services
(`GoogleApiAvailability`, `GoogleApiAvailabilityLight`, `DeferredLifecycleHelper`,
`GooglePlayServicesUtilLight`) de forma independiente -- en un emulador AOSP sin GMS instalado, las 4
fallaban con el mismo `ClassNotFoundException`, ensuciando `hooks_fallidos` con ruido repetido en cada
corrida. Ahora se chequea UNA vez con la clase "ancla" (`GoogleApiAvailability`) y se saltan las 4 juntas
si no está. Validado con `node --check` sobre el script generado completo (28KB, sintaxis JS real) +
3 tests nuevos en `tests/test_frida_bypass_script.py` (primer archivo de tests para este módulo).

### 4. [MEDIO] Falso positivo de cert-pinning OkHttp

`detectors/certificate_pinning.py` marcaba `detected=True` con solo ver la clase `okhttp3.CertificatePinner`
en el APK -- pero esa clase viene incluida en OkHttp y aparece en CUALQUIER app que use la librería, la
use o no para pinning real. En el job 2822, un método dummy `NetworkClient.disableCertificatePinning()`
que solo loguea (la app usa `HttpsURLConnection`, no OkHttp para pinning) disparaba el falso positivo, y
el agente LLM perdió ~3 iteraciones descartándolo a mano. Fix: la clase `CertificatePinner` de OkHttp
ahora es un indicador "ambiguo" que solo cuenta si viene acompañado de un hash de pin real (`sha256/`,
NSC `<pin-set>`) -- los indicadores realmente inequívocos (TrustKit, `PinningTrustManager`,
`PublicKeyPinning`) siguen bastando por sí solos. Bug encontrado ESCRIBIENDO el test:
`"CertificatePin"` (sin "ner") en la lista de indicadores "fuertes" originales resultó ser PREFIJO
literal de `"CertificatePinner"`, colisionando con el gate nuevo y anulándolo -- sacado. 7 tests nuevos
en `tests/test_certificate_pinning_detector.py` (primer archivo de tests para este detector).

### 5. [BAJO] `logcat` en modo streaming, implementado vía RPC

Completa la Etapa 5 del relay (después de shell/install/pull/screencap). Diseño pragmático: NO es
streaming línea-a-línea real -- el único consumidor real (`frida_capture.py::stream_logcat`) solo junta
las líneas en una lista para razonar sobre ellas DESPUÉS de que termina la ventana de captura, nunca las
usa mientras llegan. Así que alcanza con el mismo patrón pedido-respuesta que las otras Etapas: el
navegador corre `logcat` (sin `-c`) durante una ventana acotada (`reader.cancel()` vía `setTimeout`, no
`Promise.race` -- soltar el lock con un `read()` en vuelo es inseguro), junta todo lo capturado, y lo
manda entero en un solo `rpc_response`. `logcat -c` (limpiar buffer, comando de una sola pasada) se
reenvía tal cual al RPC de `shell` que ya existía -- no necesitaba nada nuevo. 9 tests nuevos (shim +
endpoint).

### 6. [ALTO] `get_app_analysis()` no veía las protecciones reales -- bug real, no falta de detección

**El hallazgo más valioso de esta tanda.** El detector de RootBeer/anti-Frida (`KnownLibrariesDetector`)
en sí funciona perfecto -- confirmado corriéndolo en vivo contra el APK real de `sh.nutcracker.nutbank`
(`detected=True`, encontró las 7 clases de RootBeer sin problema). El bug estaba en
`plugins/aipwn/__init__.py::_load_analysis_json()`: elegía el .json "más reciente" con
`sorted(glob("*.json"), reverse=True)` y una lista negra de 3 nombres reservados
(`vuln.json`/`osint.json`/`bypass_result.json`) para filtrar los que NO son un `AnalysisResult` real --
pero esa lista no cubría `exploit_report_<package>.json` (escrito por el exploit agent, con orden
alfabético que lo hace "ganar" el sort antes que los .json de análisis reales con nombre
`<YYYYMMDD>_<HHMMSS>.json`). Ese archivo tiene una key `"package"` pero NO `"detections"` --
`AnalysisResult.from_dict()` lo aceptaba en silencio como un análisis VACÍO (`results=[]`) en vez de
tirar `KeyError`, así que `get_app_analysis()` reportaba "sin protecciones" pese a que el análisis
estático real (guardado aparte) sí había encontrado RootBeer, anti-Frida y verificación de firma.
**Afecta a CUALQUIER app con un exploit_report previo** -- es decir, después del primer bypass exitoso,
cada corrida siguiente de aipwn perdía la detección estática por completo, forzando al agente a
depender solo de memoria de sesiones pasadas o exploración a ciegas. Fix: en vez de mantener una lista
negra que hay que actualizar cada vez que se agrega un nuevo tipo de reporte, se filtra por el patrón de
nombre POSITIVO real que usa `reporter.save_analysis_json()` (regex `^\d{8}_\d{6}\.json$`), más un
resguardo extra validando que el dict cargado tenga la key `"detections"` antes de aceptarlo. Verificado
en vivo: `_load_analysis_json('sh.nutcracker.nutbank')` ahora carga 8 detectores con RootBeer/anti-Frida/
firma correctamente detectados. 6 tests nuevos en `tests/test_aipwn_load_analysis_json.py`.

### Verificación

38 tests nuevos en total entre los 6 fixes. Suite completa: 395 tests pasan (los mismos 6 fallos
preexistentes sin relación, ver fixes anteriores de esta sesión).

## Auditoría: ¿se expone la API key del LLM al frontend? (2026-08-04)

Pregunta del usuario tras la discusión de despliegue en VPS público. Verificado, no supuesto:

- Cero referencias a `api_key`/`llm_cfg`/config del LLM en todo `nutcracker_core/plugins/dashboard/`
  (backend y frontend, grep exhaustivo).
- `/api/agent/prompt` solo devuelve el system prompt hardcodeado (`_SYSTEM_PROMPT`), nunca config.
- `/api/config/default-serial` solo devuelve `strategies.default_device_id`.
- El mount de `StaticFiles` apunta a `dashboard/static/`, un árbol de directorios separado de donde
  vive `config.yaml` (raíz del repo) -- no hay forma de que ese mount lo sirva.
- `build_job_cmd()` (orchestrator.py) para el job de aipwn nunca pasa la key por CLI args ni por env
  inyectado desde el dashboard -- el subproceso `aipwn` lee `config.yaml` él mismo, server-side
  (`llm_cfg = config.get("llm", {})` en `aipwn.py`).
- Los logs del job SÍ se streamean en vivo al navegador (EventBus + `/ws/jobs/{id}`) -- pero no hay
  ningún `print`/`console.print` que vuelque `config`/`api_key`/`client_args` crudos, y los mensajes de
  error del LLM (`f"LLM error: {e}"` en `frida_agent.py`) usan `str(exception)` de los SDKs de
  OpenAI/Anthropic, que no eco-an el header de Authorization en el texto del error.

**Veredicto: confirmado seguro**, ningún endpoint ni ruta de logs expone la key. Sigue siendo una key
única compartida en texto plano en `config.yaml` (problema de multi-tenancy, fuera de alcance mientras
sea single-user -- ver sección de arquitectura del relay más arriba).

## HTTPS para despliegue en VPS (2026-08-04)

Usuario no tiene dominio propio ni VPS elegido todavía (eligió Ubuntu/Debian como SO). Se armó el setup
en `deploy/` (Caddy -- TLS automático vía Let's Encrypt, renovación sola, proxea WebSocket sin config
extra):

- `deploy/Caddyfile` -- reverse proxy a `127.0.0.1:8765`.
- `deploy/nutcracker-dashboard.service` -- unit de systemd, dashboard bindeado a `127.0.0.1` (NO
  `0.0.0.0` como en uso local LAN) para que solo Caddy lo alcance; firewall bloquea el puerto directo.
- `deploy/README.md` -- guía paso a paso, incluye recomendación de DuckDNS (subdominio gratis con DNS
  real, válido para Let's Encrypt) ya que el usuario no tiene dominio propio comprado.

Pendiente explícito documentado en el propio README: con esto el dashboard queda en HTTPS pero sigue
**sin autenticación** -- cualquiera con la URL puede operarlo. Es el siguiente bloqueador real, no
cubierto por este cambio. (Resuelto después, ver la sección de login más abajo.)

## Deploy actualizado: VM real en la nube, cert ya existente, nginx en vez de Caddy (2026-08-05)

El usuario ya tiene la VM concreta: acceso solo por SSH, con un certificado TLS YA EMITIDO como
archivo en la VM (cubre el hostname público que el proveedor de nube le asignó). Cambios sobre el
setup anterior:

- **Sin Let's Encrypt/DuckDNS**: el cert ya existe, no hace falta que nada lo emita -- se cargan los
  archivos directo. Sin ACME, tampoco hace falta el puerto 80 abierto (ni en el security group de la
  nube ni en `ufw`).
- **Proxy: nginx en vez de Caddy** (decisión del usuario, tras preguntarle -- también se ofreció
  "uvicorn con TLS nativo, sin proxy" como opción más minimalista, pero eligió nginx). `deploy/Caddyfile`
  se borró; nuevo `deploy/nutcracker.nginx.conf` -- a diferencia de Caddy, nginx NO maneja WebSocket
  solo: hace falta el bloque `map $http_upgrade $connection_upgrade` + `proxy_set_header
  Upgrade/Connection` a mano, si no el handshake de `/ws/jobs`, `/ws/chat`, `/ws/relay` falla. También
  se subió `proxy_read_timeout`/`proxy_send_timeout` a 3600s -- el default de nginx (60s) cortaría
  WebSocket idle a mitad de un job largo o una sesión de relay.
- **Higiene open source**: los certs de la VM del usuario en la práctica viven dentro del directorio
  de otro proyecto suyo (no-público) que resulta compartir la máquina -- el usuario marcó explícito que
  no quiere ninguna ruta específica de ese proyecto en el repo público de nutcracker. `deploy/` quedó
  genérico: `nutcracker.nginx.conf` usa `/etc/nutcracker/tls/cert.pem`/`key.pem` como convención de
  ejemplo (paso 0 nuevo del README, "dónde poner el certificado"), y el README documenta el riesgo de
  acoplamiento en términos genéricos ("si tu cert vive en el directorio de otra app...") sin nombrar
  nada específico del usuario. Mismo criterio aplicado acá arriba (sin el hostname real ni el nombre
  del otro proyecto).
- Resto sin cambios: puerto público 8765, dashboard interno 127.0.0.1:8766, login vía
  `dashboard-hash-password` (paso 3.5 del README).

## Login / autenticación del dashboard (2026-08-05)

Pedido del usuario antes de desplegar en VPS: "implementa un login a la página y que
todo el flujo sea autenticado". Es el bloqueador #2 que veníamos difiriendo (después del
HTTPS).

**Diseño (sin dependencias externas — itsdangerous NO está instalado):**
- `plugins/dashboard/auth.py` (nuevo): hashing pbkdf2-sha256 (stdlib), tokens de sesión
  firmados con HMAC-SHA256 (stateless: payload=usuario:expiración + firma; no hay store de
  sesión en el server, sobrevive reinicios si el `secret_key` es fijo), y un middleware ASGI
  puro que protege HTTP + WebSocket. Cookie HttpOnly/SameSite=Lax/Secure.
- Credenciales en `config.yaml` (`dashboard.auth`): username + password_hash (nunca texto
  plano) + secret_key + session_hours. Se activa solo con `enabled: true` -- sin config, el
  dashboard queda abierto (uso local/dev, retrocompat: los 97 tests de dashboard existentes
  pasan sin tocar).
- **Token interno (machine credential)**: generado al arrancar, inyectado al subproceso aipwn
  por env (`NUTCRACKER_DASHBOARD_TOKEN`), que lo manda en el header `X-Nutcracker-Token` al
  pollear el mailbox de chat. NECESARIO porque detrás de Caddy toda request llega desde
  127.0.0.1 -- no se puede eximir al subproceso por IP de origen (un bypass de localhost
  eximiría también a todo el tráfico proxeado, anulando el login).
- Middleware: /api/* no auth → 401 JSON; SPA no auth → 302 /login; WS no auth → close 1008.
- Frontend: pantalla `static/login.html` self-contained; `j()` redirige a /login en 401; los
  WebSocket redirigen en close 1008; botón "Salir" (logout).
- CLI nuevo `dashboard-hash-password` para generar el hash + bloque YAML listo para pegar.

**Verificación:** 24 tests nuevos en `tests/dashboard/test_auth.py` (hashing, tokens firmados
con casos de manipulación/expiración, AuthConfig, y flujo end-to-end vía TestClient: 401 sin
sesión, redirect del SPA, login→cookie→acceso→logout, bypass por token interno, rechazo de WS
no autenticado, WS aceptado con cookie, retrocompat sin auth). Suite completa: 419 pasan
(los mismos 6 fallos preexistentes de test_aipwn_adb_transport_reconnect.py, sin relación --
un monkeypatch a frida_agent.adb_transport que no existe como atributo).

**deploy/**: README actualizado con el paso 3.5 (activar login antes de exponer); config.yaml
público interno pasó a puerto 8766 (Caddy sirve el 8765 público, ver arriba); la sección
"Pendiente" ahora es solo multi-usuario real (una sola credencial compartida + API key del LLM
compartida).

## Migración de npm a pnpm en webusb/ (2026-08-05)

Pedido del usuario: revisar si `setup.sh` usaba npm, con la hipótesis de que está "deprecado por
supply chain" y hoy se usa pnpm. Aclaración hecha al usuario: npm no está deprecado formalmente, pero
sí hay una razón de seguridad real y vigente para preferir pnpm -- bloquea por defecto los scripts
`postinstall` de las dependencias, justo el vector que explotaron varios ataques de supply-chain
recientes en el ecosistema npm (paquetes populares comprometidos que exfiltran datos/se propagan al
`install`). Para un proyecto de seguridad como este, es una elección con fundamento, no solo moda.

**Cambios, todos verificados en vivo (Node 24 + corepack, disponibles en este entorno):**
- `webusb/package.json`: agregado `"packageManager": "pnpm@11.20.0+sha512..."` (vía `corepack use
  pnpm@latest`) -- fija la versión exacta, instalación reproducible sin depender de qué pnpm tenga
  cada máquina. Los scripts `check`/`build` tenían una indirección `npm run fetch-server` interna --
  cambiada a invocar `fetch-scrcpy-server 3.3.1` directo, así quedan agnósticas de qué manager las
  invoca.
- Nuevo `webusb/pnpm-workspace.yaml` con `allowBuilds: { esbuild: true }` -- pnpm bloqueó el
  postinstall de esbuild por default (`[ERR_PNPM_IGNORED_BUILDS]`); se aprobó explícitamente porque es
  necesario (baja el binario nativo de la plataforma, lo usa vite) y su propósito es conocido. Ningún
  otro paquete de las 125 dependencias necesita scripts de instalación -- quedan bloqueados por
  default, que es justo la protección que motivó el cambio.
- Borrado `webusb/package-lock.json` (lockfile de npm), generado `webusb/pnpm-lock.yaml`.
- `setup.sh`: reemplazado todo el bloque de `npm install`/`npm run build` por `corepack enable` +
  `pnpm install`/`pnpm run build`, con los mismos fallbacks (Node no encontrado, node/npm resolviendo
  al .exe de Windows en WSL, corepack no encontrado).
- `README.md` (raíz) y `webusb/README.md`: comandos actualizados a pnpm/corepack.

**Verificación (no asumida, corrida de verdad):** `pnpm install` (125 paquetes, mismo conteo que con
npm) + `pnpm run build` produce un bundle **byte a byte idéntico** al que había generado npm
(335166 bytes, diff sin salida) -- confirma que el swap de package manager no cambió el output.
`pnpm run check` (`tsc --noEmit`) sigue en 0 errores. `node --check` sobre el bundle: sintaxis válida.

## Fix: setup.sh — apkeep 0.10.0 daba 404 (2026-08-05)

Reportado en vivo por el usuario corriendo `setup.sh` en la VM: `curl: (22) The requested URL
returned error: 404` al descargar apkeep. Causa real: apkeep cortó un major (0.10.0 -> 1.0.0,
2026-04-29) y el release viejo ya no existe en GitHub; de paso, el asset `x86_64-unknown-linux-musl`
que usaba `setup.sh` tampoco existe más en el release nuevo (ahora es `-gnu`).

Revisado el CHANGELOG real de apkeep (0.18.0 -> 1.0.0): son cambios aditivos (dex metadata, device
properties propios, auth-token alternativo) + un fix de auth de Google Play -- nada que rompa las
flags `-a`/`-d`/`OUTPATH` que usa `nutcracker_core/downloader.py`. Upgrade seguro, sin tocar código.

Fix en `setup.sh`: en vez de fijar `APKEEP_VERSION` a mano (lo que causó esta rotura), resuelve el
tag `latest` real vía la API de GitHub (`api.github.com/.../releases/latest`), con fallback a un tag
fijo si la API no responde (rate limit/sin red). Asset corregido a `-gnu`. Verificado en vivo: la
versión se resuelve a `1.0.0`, el binario descarga y corre (`apkeep --version`, `apkeep --help`
confirma el mismo shape de CLI).

## Fix #2 en el mismo bloque de apkeep: curl:(23) real, y uno cosmético (2026-08-05)

Usuario reportó `curl: (23) Failure writing output to destination` corriendo `setup.sh` de nuevo tras
el fix anterior -- pero corriendo como root. Eso descarta el permiso de `/usr/local/bin` como causa de
ESE mensaje puntual (aunque el fallback a `~/.local/bin` para usuarios no-root sigue siendo válido y se
mantiene, ver abajo). Diagnóstico correcto reproducido en vivo: `curl ... | grep -m1 ... | sed ...` --
`grep -m1` corta la lectura apenas encuentra el primer match y cierra su extremo del pipe; curl, que
todavía está escribiendo el resto del JSON, recibe SIGPIPE y reporta exactamente
"curl: (23) Failure writing output to destination, passed N returned M" -- cosmético (el valor ya
había sido capturado antes del corte) pero indistinguible a simple vista de un error real, confundió
al usuario con razón.

Dos fixes en el mismo bloque de `setup.sh`:
1. **Permisos** (para usuarios no-root, sigue siendo válido): si `/usr/local/bin` no es escribible,
   cae a `~/.local/bin` (creándolo si hace falta) y avisa si ese directorio no está en `$PATH`.
2. **El pipe roto** (la causa real de lo reportado corriendo como root): `curl` ahora se captura
   primero entero en una variable (`$(...)` lee hasta EOF, sin pipe en vivo hacia grep) y RECIÉN
   DESPUÉS se le hace `grep -m1` sobre la variable ya completa -- curl nunca queda escribiendo a un
   pipe que se cierra a mitad de camino.

Verificado en vivo (reproducido el mensaje cosmético primero para confirmar la causa, después
confirmado que el fix lo elimina): misma versión resuelta (1.0.0), mismo binario descargado y
funcional (`apkeep --version`), cero mensajes `curl:(23)` de ningún tipo.

## Deploy real verificado en vivo en la VM (2026-08-05)

Todo el checklist de `deploy/README.md` corrido de verdad contra la VM real del usuario (Azure,
certs de otro proyecto en la misma máquina, PentestAI con su propio contenedor Docker en :443):

- **Conflicto de puerto 443 real, no hipotético**: el sitio `default` de nginx (con un bloque SSL
  agregado por un `certbot --nginx` de algún momento anterior, para el mismo hostname
  `vm-sbx-1.eastus.cloudapp.azure.com`) competía por el 443 contra el contenedor `dashboard` de
  PentestAI (publicado directo vía `docker-proxy`). Causaba que el paquete `nginx` de apt quedara a
  medio instalar (`dpkg` con status roto) cada vez que cualquier otro `apt install` disparaba su
  postinst. Resuelto con `sudo unlink /etc/nginx/sites-enabled/default` (reversible, no toca el
  archivo real en `sites-available/`) + `sudo dpkg --configure -a` -- nginx arrancó limpio, sirviendo
  SOLO lo que le pedimos (8765), sin tocar el 443 de PentestAI.
- **Hallazgo que corrige una instrucción previa del README**: el `chown root:www-data` +
  `chmod 440` que había indicado para los certs NO hacía falta -- nginx lee `ssl_certificate`/
  `ssl_certificate_key` en su proceso MASTER, que corre como root (antes de que los workers bajen
  privilegios a `www-data`), así que root puede leerlos sin importar su dueño/permisos actuales.
  Confirmado en vivo: el usuario NO corrió ese paso y funcionó igual. De hecho dejar la key legible
  SOLO por root es más restrictivo que lo que había sugerido -- se sacó ese paso del README.
- Verificación de punta a punta contra la VM real: `curl -kI https://127.0.0.1:8765/` -> 302 a
  `/login` (auth middleware funcionando); `curl https://127.0.0.1:8765/login` -> 200 con el HTML real
  del login. `nutcracker-dashboard.service` activo con `auth=on` en el log de arranque.
- Nota menor encontrada (no bloqueante): `/login` solo registra `GET` explícito, no `HEAD` --
  `curl -I` (que manda HEAD) da 405. Un navegador real navega con GET, así que no afecta el flujo de
  login; queda documentado por si en el futuro algo (monitoring/health-check) le pega con HEAD.

## Feature: subir .apk desde el dashboard (2026-08-05)

Pedido del usuario pensando en el despliegue en VPS: sin terminal en el servidor, no hay forma de
dejar un .apk local en `downloads/` antes de encolar un job "analyze"/"dynamic" con target local --
antes solo se podía con package id (descarga automática) o URL directa.

- **Backend**: `POST /api/apks/upload` (nuevo, `api.py`) -- recibe `multipart/form-data`
  (`UploadFile`), valida extensión `.apk` + firma de magic bytes real (`PK\x03\x04`/`PK\x05\x06`/
  `PK\x07\x08`, ZIP/APK), sanea el nombre (`Path(filename).name` descarta cualquier componente de
  directorio -- incluida una traversal tipo `../../etc/passwd.apk` -- + regex a caracteres seguros),
  nunca pisa un archivo existente (numera `_1`, `_2`, ...), y guarda en `downloads/<nombre>.apk`
  (mismo directorio/convención que ya usa `downloader.py`). Devuelve `path` en el formato exacto que
  espera el campo `target` de `/api/queue` (`_is_local_apk()` en `queue/engine.py` solo chequea que el
  path exista y termine en `.apk`).
  Requirió agregar `python-multipart` a `plugins/dashboard/requirements.txt` (FastAPI no trae soporte
  de multipart por sí solo).
- **Frontend**: input de archivo + botón "Subir" arriba del form existente de "Cola de análisis" --
  al subir, autocompleta el campo `target` con el path devuelto, listo para encolar.
- Cubierto por auth: al vivir bajo `/api/`, el middleware de login ya lo protege sin cambios extra
  (verificado con un test dedicado).

**Verificación**: 7 tests nuevos en `tests/dashboard/test_api.py` (upload válido + path relativo
correcto, extensión rechazada, firma de magic bytes rechazada -- sin dejar el archivo en disco --,
traversal saneado, no pisa archivo existente, caracteres raros saneados, 401 sin sesión cuando auth
está activado). Suite completa: 426 pasan (los mismos 6 fallos preexistentes sin relación).

## Fix: nginx 413 al subir un APK real (2026-08-05)

Reportado en vivo probando el feature de upload recién agregado: `413 Request Entity Too Large`.
Causa: el default de nginx para `client_max_body_size` es apenas 1MB -- cualquier .apk real lo supera
sin esfuerzo. No es un límite que pusimos nosotros a propósito, se me pasó al armar
`nutcracker.nginx.conf` (el backend en sí no tiene límite propio, ver `POST /api/apks/upload`).

Fix: agregado `client_max_body_size 500m;` al server block de `deploy/nutcracker.nginx.conf`. En la
VM hace falta volver a copiar el archivo y recargar:
```
sudo cp deploy/nutcracker.nginx.conf /etc/nginx/conf.d/nutcracker.conf
sudo nginx -t && sudo systemctl reload nginx
```

## Fix: mensaje de "no hay decompilador" asumía macOS (2026-08-05)

Reportado en vivo con un job real corriendo en la VM: el log mostraba "brew install jadx" como única
sugerencia -- Homebrew no existe en Linux, inútil en el VPS (o en cualquier WSL/Linux). Causa: el
mensaje estaba hardcodeado a instrucciones de macOS sin chequear la plataforma real.

Fix en `decompiler.py::install_instructions()`: ahora usa `platform.system()` para dar instrucciones
reales según el SO (`brew install` en macOS, link al release de GitHub + apktool.org en Linux), y
**siempre** menciona primero el toolbox de Docker (cross-platform, la opción correcta para un
servidor/VPS sin querer instalar binarios en el host -- justo el caso de este VM). `DecompilerError`
(en `decompile()`) dejó de tener su propio texto hardcodeado aparte y ahora reusa
`install_instructions()`, para no tener dos mensajes que puedan desincronizarse.

4 tests nuevos en `tests/test_decompiler.py` (mensaje por plataforma Darwin/Linux, toolbox siempre
mencionado sin importar el SO, DecompilerError comparte el mismo texto). Suite completa: 430 pasan
(los mismos 6 fallos preexistentes sin relación).

Nota para el usuario: este mensaje es un síntoma, no la causa raíz -- en su VM, la causa real es que
el toolbox de Docker (pasos dados un rato antes en la conversación: `toolbox.enabled: true` en
config.yaml + usuario en el grupo `docker` + reiniciar el servicio) todavía no está terminado de
activar.

## Fix real: el toolbox de Docker nunca se detectaba en el flujo de decompilación (2026-08-05)

Reportado en vivo: usuario activó `toolbox.enabled: true` en su VM (grupo docker, imagen, todo según
los pasos dados antes) pero el job seguía fallando con "No se encontró ningún decompilador". La causa
NO era su config -- era un bug real en `orchestrator.py`, en DOS puntos separados que chequeaban
disponibilidad de jadx sin pasarle el config cargado (`_CFG`):

1. `_do_decompile()` (línea ~1187): `tool, _ = get_available_tool()` -- sin `config=_CFG`. Como
   `toolbox.is_enabled(None)` siempre da False, este chequeo SIEMPRE reportaba "no hay decompilador"
   sin importar `toolbox.enabled`, cortando antes de llegar a la línea de más abajo que sí pasaba
   `config=_CFG` correctamente a `decompile()` -- esa línea nunca se alcanzaba.
2. `_validate_all_dependencies()` (línea ~395): `if not _shutil.which("jadx")` a secas, ignorando el
   toolbox por completo (ni siquiera pasaba por `decompiler.get_available_tool`/`_find_tool`).

Fix: `_do_decompile()` ahora pasa `config=_CFG` a `get_available_tool()`; `_validate_all_dependencies()`
usa `decompiler._find_tool("jadx", _CFG)` en vez de `shutil.which("jadx")` directo. Verificado en vivo
(antes de escribir los tests formales): con `_CFG = {"toolbox": {"enabled": True}}` y
`shutil.which` mockeado a `None` (nada instalado local), ambos puntos ahora devuelven `"toolbox"`
correctamente.

3 tests nuevos en `tests/test_orchestrator.py`: toolbox detectado sin jadx local (confirma que
`decompile()` se invoca, no que corta antes), retrocompatibilidad (sin toolbox y sin binario local
sigue fallando limpio, no "inventa" disponibilidad), y `_validate_all_dependencies` acepta el toolbox
para la dependencia jadx. Suite completa: 433 pasan (los mismos 6 fallos preexistentes sin relación).
