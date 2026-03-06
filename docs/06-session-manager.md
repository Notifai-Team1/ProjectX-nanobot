# Módulo: SessionManager

## Objetivo

`SessionManager` administra el ciclo de vida de sesiones conversacionales persistidas en JSONL, con cache en memoria para acceso rápido durante el procesamiento.

La clase `Session` modela una conversación append-only y conserva metadatos de consolidación para integrar memoria de largo plazo sin perder trazabilidad.

---

## Estructura de datos

## `Session`

Campos principales:

- `key`: identificador lógico (`channel:chat_id`).
- `messages`: lista append-only de mensajes crudos.
- `created_at`, `updated_at`.
- `metadata`: diccionario libre.
- `last_consolidated`: índice lógico de corte para memoria consolidada.

Métodos:

- `add_message(...)`: agrega entrada con timestamp actual.
- `get_history(max_messages=500)`: retorna solo tramo no consolidado y ajustado para entrada LLM.
- `clear()`: vacía mensajes y reinicia estado de consolidación.

---

## Política de historial para LLM (`get_history`)

`get_history` aplica varias reglas importantes:

1. Parte desde `messages[last_consolidated:]` para ignorar tramo ya consolidado.
2. Limita a `max_messages` por ventana deslizante.
3. Si el slice comienza con roles no-user, recorta hasta el primer `user` para evitar bloques huérfanos (`tool`/`assistant`).
4. Serializa salida mínima por mensaje (`role`, `content`) y solo campos de tool protocol necesarios (`tool_calls`, `tool_call_id`, `name`).

Esta forma protege consistencia del prompt y compatibilidad con providers.

---

## Persistencia en disco

## Carpeta

- Ubicación principal: `workspace/sessions`.
- Migración legacy automática desde `~/.nanobot/sessions` cuando aplica.

## Formato JSONL

Cada archivo tiene:

1. Primera línea de metadata (`_type = "metadata"`) con:
   - `key`, `created_at`, `updated_at`, `metadata`, `last_consolidated`.
2. Líneas siguientes: mensajes individuales JSON.

Ventajas: append/read humanos, resiliencia y debugging sencillo.

---

## APIs de manager

- `get_or_create(key)`: cache-hit si existe, si no carga de disco o crea nueva.
- `_load(key)`: parsea JSONL, aplica migración legacy y tolera errores.
- `save(session)`: reescribe archivo completo con metadata + mensajes.
- `invalidate(key)`: expulsa del cache en memoria.
- `list_sessions()`: enumera sesiones leyendo solo primera línea metadata de cada archivo.

El orden de `list_sessions` es descendente por `updated_at`.

---

## Decisiones de diseño

- **Append-only lógico** en `Session.messages`: evita mutaciones complejas del historial.
- **Consolidación no destructiva**: memoria se externaliza, pero mensajes originales persisten hasta `clear()` explícito.
- **Cache en memoria** para rendimiento en loop de agente.
- **Migración backward-compatible** desde ruta histórica del usuario.

---

## Riesgos y notas operativas

- `save()` reescribe archivo completo (simple y seguro, pero O(n) por tamaño de sesión).
- Tolerancia de errores en lectura prioriza continuidad del sistema frente a sesiones dañadas.
- `safe_filename` y normalización de `key` reducen riesgo de rutas inválidas.
