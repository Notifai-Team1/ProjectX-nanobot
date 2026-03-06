# Módulo: AgentLoop

## Objetivo

`AgentLoop` es el orquestador principal del runtime. Coordina:

1. Entrada de mensajes (`MessageBus`).
2. Contexto conversacional (`ContextBuilder` + `SessionManager` + memoria).
3. Interacciones LLM iterativas con tool calling.
4. Persistencia de turnos y consolidación de memoria.
5. Salida de respuestas/progreso y control de tareas (`/stop`).

---

## Responsabilidades clave

## 1) Bootstrapping de herramientas

En `_register_default_tools()` registra herramientas base en `ToolRegistry`:

- filesystem (`read_file`, `write_file`, `edit_file`, `list_dir`)
- `exec`
- `web_search`, `web_fetch`
- `message`
- `spawn`
- `cron` (si existe `cron_service`)

Además puede conectar tools MCP de manera perezosa (`_connect_mcp`) con `AsyncExitStack` para cierre ordenado.

## 2) Bucle principal de consumo

`run()` mantiene el loop activo:

- Consume mensajes de `bus.consume_inbound()` con timeout corto.
- Si recibe `/stop`, delega en `_handle_stop`.
- Caso contrario, crea una task por mensaje con `_dispatch` y la indexa por `session_key`.

Aunque hay task por mensaje, `_dispatch` ejecuta `_process_message` bajo `_processing_lock`, serializando procesamiento para estabilidad global.

## 3) Control de cancelación (`/stop`)

`_handle_stop` cancela:

- Tasks activas en la sesión actual.
- Subagentes asociados (`cancel_by_session`).

Luego publica una respuesta con total de tareas detenidas.

## 4) Pipeline de procesamiento (`_process_message`)

Ruta general:

1. Resuelve sesión (`get_or_create`).
2. Atiende slash commands:
   - `/new`: intenta consolidar snapshot pendiente con `archive_all=True`; si falla, no limpia sesión.
   - `/help`: devuelve comandos disponibles.
3. Dispara consolidación asíncrona incremental si supera `memory_window` y no hay consolidación en curso.
4. Propaga contexto de ruteo a tools (`message`, `spawn`, `cron`).
5. Construye mensajes iniciales con historial + contexto runtime + contenido usuario.
6. Ejecuta `_run_agent_loop(...)`.
7. Persiste turno (`_save_turn`) y sesión (`sessions.save`).
8. Decide respuesta final:
   - Si `message` tool ya envió en ese turno, retorna `None` para no duplicar.
   - En otro caso publica `OutboundMessage` normal.

Incluye ruta especial para `msg.channel == "system"`, resolviendo origen real desde `chat_id` en formato `channel:chat`.

## 5) Motor iterativo LLM + tools (`_run_agent_loop`)

Por iteración:

- Llama a `provider.chat(...)` con tools, modelo y parámetros de generación.
- Si hay tool calls:
  - Persiste mensaje assistant con tool_calls.
  - Ejecuta cada tool vía `ToolRegistry.execute`.
  - Inserta respuesta de tool como mensaje role `tool`.
  - (Opcional) emite progreso y hints de tools hacia el canal.
- Si no hay tool calls:
  - Limpia bloques `<think>...</think>` de contenido.
  - Corta por error de provider (`finish_reason == "error"`) sin contaminar historial.
  - Cierra con respuesta final.

Si agota `max_iterations`, retorna un mensaje de límite alcanzado.

---

## Persistencia de turnos y sanitización (`_save_turn`)

`_save_turn` agrega mensajes nuevos a sesión con reglas defensivas:

- Omite assistant vacío sin `tool_calls`.
- Trunca outputs de tools extensos a `_TOOL_RESULT_MAX_CHARS`.
- Elimina bloque de runtime context del mensaje user antes de guardar.
- Para multimodal, reemplaza imágenes embebidas base64 por marcador `[image]`.
- Añade `timestamp` si falta.

Objetivo: evitar ruido contextual y crecimiento explosivo del historial.

---

## Concurrencia y consistencia

- `_consolidation_locks`: un lock por sesión para no solapar consolidaciones.
- `_consolidating`: set de sesión en curso (evita duplicación).
- `_consolidation_tasks`: referencias fuertes para evitar GC prematuro.
- `_active_tasks`: seguimiento de tasks por sesión para `/stop`.

---

## Extensión

`AgentLoop` es el punto natural para:

- Registrar nuevas tools por defecto.
- Inyectar proveedores LLM alternativos.
- Ajustar políticas de iteración, consolidación y prompt.
- Añadir nuevas rutas de control (nuevos slash commands).
