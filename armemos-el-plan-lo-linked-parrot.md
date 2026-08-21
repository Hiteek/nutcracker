# Plan: Chat co-piloto de pentest (estático + dinámico interactivo)

## Context

Hoy el dashboard tiene una sola pestaña "Agente / Chat" cuyo chat (`/ws/chat/{package}`)
es un canal **operador→agente en vivo**: solo tiene efecto si hay un job `aipwn`
autónomo corriendo para ese package (el subproceso hace polling a
`/api/chat/{package}/pending` en cada iteración de su loop ReAct). No es interactivo ni
sirve para explorar por tu cuenta.

Lo que el usuario quiere es un **asistente conversacional de pentest asistido** que sea:

1. **Determinista/estático**: responder preguntas sobre el análisis estático y los
   resultados de aipwn ya generados (findings, componentes exportados, secretos y si son
   reales, leer/buscar en el decompilado) — disponible **sin ningún job corriendo**.
2. **Dinámico e interactivo**: cuando se lo pida en lenguaje natural, **usar Frida y ADB
   sobre el dispositivo**, **ver la pantalla del celular** (screenshot → visión), y
   **actuar sobre la UI** (ej: "¿Ves esta pantalla? Ingresá un usuario y contraseña" →
   escribe texto / hace taps), dándole al operador feedback sobre qué está viendo, qué
   consiguió y hacia dónde ir.

En esencia es un **co-piloto interactivo del agente aipwn**: reutiliza casi todo su
toolset (estático + runtime), pero manejado conversacionalmente por el operador en vez
de perseguir un bypass de forma autónoma.

**Decisiones confirmadas con el usuario:**
1. **Pestaña nueva separada** ("Pentest asistido"), sin tocar el chat de intervención en vivo.
2. **Reusar el bloque `llm:` de config.yaml** que ya usa aipwn (cero config nueva).
3. **Chat conversacional puro**: el LLM decide y ejecuta las tools vía tool-calling; el
   operador pide en lenguaje natural (sin slash-commands ni palette de botones).
4. **Ambos modos de conexión al device**: serial USB al host del dashboard **y** relay
   WebUSB del navegador (caso VPS).

## Enfoque

Un **agente ReAct conversacional** que corre **in-process en el dashboard** (los tools
de Frida/adb hacen shell-out a los binarios `frida`/`adb`; no requiere un proceso Python
aparte), reutilizando el `LLMClient`, el `ToolContext` y el grueso del toolset del plugin
aipwn, más tools nuevas de consulta (store/reportes) y de interacción de UI.

Vive en el plugin aipwn (reusa su `LLMClient`, `ToolContext`, tools estáticas y de
runtime). El dashboard lo importa de forma perezosa/tolerante igual que ya hace con
`agent_memory` ([api.py:21-24](nutcracker_core/plugins/dashboard/api.py#L21-L24)); si
aipwn no está → la pestaña reporta "no disponible".

### Transporte de dispositivo (serial USB o relay WebUSB)

Abstracción nueva `DeviceIO` (en `query_tools.py`) con `.screencap() -> png_bytes` y
`.shell(cmd) -> str`, con dos backends:
- **Serial**: `adb -s <serial> exec-out screencap -p` / `adb -s <serial> shell <cmd>`
  (mismo mecanismo que [tool_take_screenshot](nutcracker_core/plugins/aipwn/frida_agent_tools.py#L1653)).
- **Relay**: RPCs `screencap` y `shell` del relay ya existentes
  ([`POST /api/relay/{session}/rpc/{screencap,shell}`], `relay_manager` en
  [relay.py](nutcracker_core/plugins/dashboard/relay.py)).

Para **Frida** no hace falta abstracción: `ToolContext` ya rutea por `serial` (-D) o
`frida_host` (-H) vía [`_frida_connect_args`](nutcracker_core/plugins/aipwn/frida_agent_tools.py#L764).
La conexión (serial vs relay) se resuelve al conectar el WS reusando
[`_resolve_relay(serial, relay)`](nutcracker_core/plugins/dashboard/api.py#L537) →
`(relay_session_id, frida_host)`, igual que los jobs aipwn/batch.

### Piezas nuevas

**1. `nutcracker_core/plugins/aipwn/query_agent.py`** (nuevo) — el co-piloto.
- Clase `QueryAgent(package, ctx_data, llm_config, db_path, device)`.
- Construye `LLMClient(llm_config)` (reusa [frida_agent.py:406](nutcracker_core/plugins/aipwn/frida_agent.py#L406)).
- Arma un `ToolContext` (reusa [frida_agent_tools.py:721](nutcracker_core/plugins/aipwn/frida_agent_tools.py#L721))
  con `decompiled_dir`/`runtime_dump_dir`/`analysis_result` + `serial`/`frida_host`
  reales + `on_frida_run` = callback que envuelve
  [`launch_frida_capture`](nutcracker_core/plugins/aipwn/frida_capture.py#L257) (espejo de
  [`FridaAgent._execute_frida`](nutcracker_core/plugins/aipwn/frida_agent.py#L765)) +
  `capture_seconds` de config + `scripts_dir` en tmp.
- `self.messages` (system prompt + conversación) por conexión; soporta mensajes
  multimodales (imágenes de screenshots) que el `LLMClient` ya maneja.
- `ask(user_message) -> Iterator[event]`: loop ReAct (espejo reducido de
  [`FridaAgent.run`](nutcracker_core/plugins/aipwn/frida_agent.py#L789)) — llama
  `llm.chat(messages, tools=QUERY_TOOL_SCHEMAS)`, ejecuta cada tool_call vía
  `dispatch_query_tool`, emite eventos (`tool`, `tool_result`, `image`, `assistant`,
  `error`), con poda de mensajes y tope de iteraciones. Termina cuando el LLM responde
  sin tool_calls (espera el próximo mensaje del operador — es interactivo, no autónomo).
- `_QUERY_SYSTEM_PROMPT` (nuevo): rol "co-piloto de pentest interactivo"; describe que
  puede consultar resultados estáticos/aipwn, ver la pantalla, manejar la UI y correr
  Frida; que debe dar feedback de lo que ve/logra y sugerir próximos pasos; y pedir
  confirmación antes de acciones riesgosas.

**2. `nutcracker_core/plugins/aipwn/query_tools.py`** (nuevo) — catálogo del co-piloto.
`QUERY_TOOL_SCHEMAS` compone tres grupos + `dispatch_query_tool(ctx, name, args, ...)`:

- **Consulta estática/reportes** (read-only):
  - Reusadas de `frida_agent_tools.py` (import directo): `read_decompiled_class`,
    `search_in_decompiled`, `list_classes_matching`, `get_certificate_pins`,
    `get_app_analysis`, `strings_native_lib`, `disassemble_native_lib`, `get_apk_signature`.
  - **Nuevas** (store + JSON de reportes):
    - `list_findings(severity?, category?)` — [store_reader.consolidated_findings_for_app](nutcracker_core/plugins/dashboard/store_reader.py#L91)
      + `description`/`recommendation`/`matched_text` desde `reports/<pkg>/vuln.json`.
    - `get_finding_detail(rule_id | file+line)` — detalle de `vuln.json` + estado
      `confirmed` (aireview) desde `decompiled/vuln_<pkg>_review.json`.
    - `list_components()` — activities/services/receivers/providers exportados + permisos
      peligrosos + flags vía [manifest_analyzer.analyze_decompiled_dir](nutcracker_core/manifest_analyzer.py#L83).
    - `list_secrets()` — secretos (vuln.json M1 + apkleaks/gitleaks + manifest) con estado
      de validez (FP estático / aireview / confirmado por PoC de aipwn).
    - `get_exploit_results()` — findings de aipwn confirmed/unverifiable + PoC desde
      `reports/<pkg>/exploit_report_<pkg>.json`.

- **Runtime introspección** (reusadas de `frida_agent_tools.py`, requieren device; no
  modifican el APK): `enumerate_runtime_classes`, `get_class_methods`,
  `get_loaded_native_libs`, `enumerate_native_exports`, `resolve_native_symbol`,
  `take_screenshot`, `probe_security_violations`, `sniff_network_calls`,
  `trace_method_execution`. Para relay, `take_screenshot` usa `DeviceIO.screencap`
  (adb directo cuando es serial). El resto ya rutea por serial/frida_host.

- **Runtime acción**:
  - `run_frida_script` (reusada) — ejecuta un hook que el operador pide. Vía `on_frida_run`.
  - **Nuevas (interacción de UI, vía `DeviceIO.shell`)**: `ui_tap(x,y)`,
    `ui_input_text(text)`, `ui_swipe(x1,y1,x2,y2)`, `ui_press_key(keycode)` →
    `input tap|text|swipe|keyevent ...`. Funcionan en serial (adb) y relay (shell RPC).

- **Excluidas de v1** (destructivas / reinstalan el APK): `patch_native_lib`,
  `relaunch_with_gadget`, y las de terminación autónoma `report_success`/`report_failure`.

**3. Resolución de contexto por package** (helper en `query_agent.py` o `query_context.py`):
factorizar de [aipwn/__init__.py](nutcracker_core/plugins/aipwn/__init__.py) la
localización del analysis JSON más reciente + `decompiled_dir`/`runtime_dump_dir`.
`resolve_package_context(package) -> (decompiled_dir, runtime_dump_dir, analysis_result)`.
Default: run más reciente del package.

### Wiring en el dashboard

**4. `ws.py`** — nuevo WS `@router.websocket("/ws/query/{package}")`:
- `_reject_if_unauthenticated(websocket)` (patrón de [ws_chat](nutcracker_core/plugins/dashboard/ws.py#L84)) → `accept()`.
- Primer frame del cliente = JSON de conexión `{"serial": ..., "relay": bool}`; resuelve
  `_resolve_relay(...)` → serial/frida_host/relay_session, construye `DeviceIO` y el `QueryAgent`.
- Import perezoso de `query_agent`; si aipwn falta → `{"kind":"error"}` y cierra.
- Loop: `receive_text()` (mensaje del operador) → corre `agent.ask(text)` en threadpool
  (el loop es bloqueante; Frida serializado por el `_frida_lock` existente) → reenvía cada
  evento como `send_json`. Los eventos `image` llevan el PNG base64 para que el operador
  vea la misma pantalla que el LLM.
- `ws.set_query_config(llm_config, db_path)` (como `ws.set_auth`, [ws.py:25](nutcracker_core/plugins/dashboard/ws.py#L25)).

**5. `server.py`** — `create_app(...)` recibe `llm_config: dict | None`; llama `ws.set_query_config(...)`.

**6. `__init__.py`** — en `dashboard()` (ya hace `load_config` en [L113](nutcracker_core/plugins/dashboard/__init__.py#L113))
resolver el bloque `llm:` y pasarlo a `create_app(..., llm_config=...)`.

**7. `api.py`** — `GET /api/query/available` → `{"available": bool}` (aipwn importable + hay `llm_config`).

### Frontend

**8. `static/index.html`** — nueva pestaña "Pentest asistido":
- `<div class="tab" data-tab="query">` + `<div class="tabpanel" id="tab-query">`: input
  `package id`, selector de device (input serial + checkbox "usar relay", reusando el
  patrón de la pestaña Dispositivo), botón Conectar, `#querylog` (transcript) y caja de texto.
- JS `connectQuery()`/`sendQuery()` calcados de
  [connectChat/sendChat](nutcracker_core/plugins/dashboard/static/index.html#L620-L640),
  apuntando a `/ws/query/{pkg}` y mandando el JSON de conexión inicial. El transcript renderiza:
  `tool` → "▸ ejecutando `X`…"; `tool_result` → resumen; `image` → **miniatura del
  screenshot** (el operador ve la pantalla); `assistant` → respuesta; `error` → error.
- Al cargar: `GET /api/query/available`; si false, deshabilita la pestaña con tooltip.

## Archivos a crear / modificar

**Nuevos:** `aipwn/query_agent.py`, `aipwn/query_tools.py` (incluye `DeviceIO` + tools de
UI), `aipwn/query_context.py` (o helper en query_agent.py).

**Modificados:** `dashboard/ws.py` (WS `/ws/query`, `set_query_config`), `dashboard/server.py`
(param `llm_config`), `dashboard/__init__.py` (pasar `llm:`), `dashboard/api.py`
(`/api/query/available`), `dashboard/static/index.html` (pestaña + JS).

## Reutilización clave (no reimplementar)

- LLM: `LLMClient` — [frida_agent.py:406](nutcracker_core/plugins/aipwn/frida_agent.py#L406)
- Tools estáticas + runtime + `ToolContext` + `dispatch_tool` — [frida_agent_tools.py](nutcracker_core/plugins/aipwn/frida_agent_tools.py) (721, 764, 2699)
- Ejecución Frida: `launch_frida_capture` + patrón `_execute_frida` — [frida_capture.py:257](nutcracker_core/plugins/aipwn/frida_capture.py#L257), [frida_agent.py:765](nutcracker_core/plugins/aipwn/frida_agent.py#L765)
- Findings del store — [store_reader.py](nutcracker_core/plugins/dashboard/store_reader.py) + `repository.findings_for_run`/`get_run`
- Componentes/manifest — [manifest_analyzer.py::analyze_decompiled_dir](nutcracker_core/manifest_analyzer.py#L83)
- Relay screencap/shell + resolución serial-vs-relay — [relay.py](nutcracker_core/plugins/dashboard/relay.py), [`_resolve_relay`](nutcracker_core/plugins/dashboard/api.py#L537)
- Auth WS — `_reject_if_unauthenticated` / `websocket_authenticated` ([ws.py:30](nutcracker_core/plugins/dashboard/ws.py#L30), [auth.py:298](nutcracker_core/plugins/dashboard/auth.py#L298))
- Import perezoso tolerante de aipwn — [api.py:21-24](nutcracker_core/plugins/dashboard/api.py#L21-L24)
- Resolución decompiled_dir / analysis JSON — [aipwn/__init__.py](nutcracker_core/plugins/aipwn/__init__.py) (factorizar)

## Consideraciones

- **Concurrencia de dispositivo**: el co-piloto interactivo y un job aipwn autónomo
  compiten por el mismo proceso/Frida del device (spawn serializado por `_frida_lock`).
  Pensado para usarse de a uno por dispositivo; documentarlo en el system prompt y la UI.
- **Presupuesto de Frida**: aipwn limita `max_frida_runs`; en modo interactivo el tope se
  hace configurable/alto (lo maneja el operador). Config opcional `chat.max_frida_runs`.
- **Screenshot/visión**: `take_screenshot` ya inyecta la imagen como mensaje multimodal al
  LLM; además se emite `kind:"image"` al WS para que el operador la vea.

## Verificación

**Tests unitarios** (fixture `sh.nutcracker.nutbank`, con reportes completos):
- `query_tools` estáticas: `list_findings` (con/sin filtro), `get_finding_detail` (merge
  vuln.json + review.json), `list_components` (parseo real del manifest → exportados),
  `list_secrets` (estado FP/confirmado), `get_exploit_results`.
- `DeviceIO`: con adb mockeado (serial) y con `relay_manager` mockeado (relay) →
  `screencap`/`shell` rutean al backend correcto; `ui_tap`/`ui_input_text` arman el
  `input ...` esperado.
- `QueryAgent.ask`: `LLMClient` stubbeado que en it.1 pide `list_components`, en it.2
  `take_screenshot` (DeviceIO mock devuelve PNG), en it.3 responde → aseverar la secuencia
  de eventos (`tool`/`tool_result`/`image`/`assistant`) y que no se llama a tools excluidas.

**Integración** (FastAPI `TestClient`, patrón `tests/dashboard/test_api.py`/`test_ws.py`):
- WS `/ws/query/{pkg}` con LLM y device falsos: enviar JSON de conexión + un mensaje,
  recibir eventos de tool + imagen + respuesta.
- Auth: sin cookie/token válido → cierre 1008. `GET /api/query/available` → `{"available": true}`.

**Prueba manual end-to-end** (dashboard corriendo):
1. `python nutcracker.py dashboard` → pestaña "Pentest asistido".
2. **Estático** (sin device): conectar a `sh.nutcracker.nutbank`, preguntar "¿Qué
   componentes exportados tiene?", "¿Los secretos son reales?", "Leé la clase Secrets y
   buscá tokens" → confirmar que llama las tools y responde, sin job corriendo.
3. **Dinámico serial** (celular USB al host): "¿Ves esta pantalla?" → screenshot visible;
   "Ingresá usuario `test` y contraseña `1234`" → `ui_input_text`/`ui_tap`; "Enganchá los
   métodos de login con Frida" → `run_frida_script`/`sniff_network_calls`.
4. **Dinámico relay** (celular en el navegador, VPS): activar relay en la pestaña
   Dispositivo, conectar el chat con "usar relay" → repetir el paso 3 y confirmar que
   screencap/input/Frida funcionan vía los RPCs del relay.

## Fuera de alcance (v1)

- Tools destructivas: `patch_native_lib`, `relaunch_with_gadget` (reinstalan/parchean el APK).
- Slash-commands / palette de botones (chat puro, decisión del usuario).
- Streaming a nivel de token (el `LLMClient` devuelve la respuesta completa por iteración;
  el streaming es por-paso ReAct, no por-token).
- Persistir la conversación entre reconexiones (el estado vive lo que dura el WebSocket).
