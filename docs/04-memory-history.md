# Módulo: Memory and History

## Objetivo

`MemoryStore` implementa una memoria en dos capas para mantener contexto útil sin inflar indefinidamente el historial de sesión:

- **`MEMORY.md`**: estado de memoria de largo plazo (hechos persistentes y consolidados).
- **`HISTORY.md`**: bitácora append-only, optimizada para búsqueda tipo grep.

La consolidación se ejecuta mediante una llamada al LLM que **debe invocar** una tool virtual `save_memory` con dos salidas estructuradas: `history_entry` y `memory_update`.

---

## Componentes principales

### 1) Inicialización y estructura en disco

En `__init__`, el módulo crea/asegura `workspace/memory` y define dos rutas fijas:

- `memory_file = memory/MEMORY.md`
- `history_file = memory/HISTORY.md`

Esto permite que el resto del sistema trate la memoria como un recurso estable por workspace.

### 2) API de lectura/escritura básica

- `read_long_term()`: devuelve el contenido de `MEMORY.md` o string vacío si no existe.
- `write_long_term(content)`: sobrescribe completamente `MEMORY.md`.
- `append_history(entry)`: añade una entrada con doble salto de línea al final de `HISTORY.md`.
- `get_memory_context()`: devuelve un bloque `## Long-term Memory` listo para inyectar en el prompt del sistema.

### 3) Consolidación semántica (`consolidate`)

`consolidate(...)` aplica la política de archivo/resumen sobre mensajes de sesión:

- **Modo incremental (`archive_all=False`)**:
  - Define `keep_count = memory_window // 2`.
  - Omite consolidación si hay pocos mensajes o no hubo progreso desde `last_consolidated`.
  - Consolida el rango `session.messages[last_consolidated:-keep_count]`.
- **Modo total (`archive_all=True`)**:
  - Consolida todos los mensajes (`old_messages = session.messages`), usado por ejemplo antes de `/new`.

Después prepara líneas normalizadas por mensaje (`[timestamp] ROLE: content`) y llama al provider con:

1. Un mensaje system (agente de consolidación).
2. Un mensaje user con memoria actual + conversación a procesar.
3. La definición de tool `_SAVE_MEMORY_TOOL`.

Si el provider no retorna tool calls válidas, se considera fallo controlado (`False`).

---

## Contrato de datos: tool `save_memory`

El LLM debe invocar `save_memory` con:

- `history_entry` (string): resumen breve, fechado `[YYYY-MM-DD HH:MM]`, orientado a búsqueda.
- `memory_update` (string): contenido completo y actualizado de `MEMORY.md`.

El módulo además tolera variaciones de provider:

- Si `arguments` llega como JSON string, intenta parsearlo.
- Si algún valor no es string, lo serializa con `json.dumps(..., ensure_ascii=False)`.

---

## Integración con sesión

Al terminar correctamente:

- Se agrega `history_entry` en `HISTORY.md` (si existe).
- Se reemplaza `MEMORY.md` solo si cambia (`update != current_memory`).
- Se actualiza `session.last_consolidated`:
  - `0` en `archive_all=True` (porque típicamente luego la sesión se limpia).
  - `len(session.messages) - keep_count` en incremental.

Esto permite que `Session.get_history()` trabaje siempre sobre mensajes **no consolidados**.

---

## Manejo de errores y observabilidad

- Ausencia de tool call, tipo de argumentos inválido o excepciones internas retornan `False`.
- Todos los eventos relevantes emiten logs (`info`, `warning`, `exception`) con tamaño de lotes y estado final.

---

## Consideraciones de diseño

- El módulo separa claramente **hechos vivos** (`MEMORY.md`) de **trazabilidad cronológica** (`HISTORY.md`).
- La consolidación es delegada al LLM, pero con un contrato fuertemente estructurado por schema.
- Es idempotente en escritura de largo plazo cuando no hay cambios.
