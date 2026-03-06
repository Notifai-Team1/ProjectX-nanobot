# Flujos de datos del sistema (con Mermaid)

## 1) Flujo principal: mensaje entrante hasta respuesta

```mermaid
sequenceDiagram
    participant U as Usuario
    participant CH as Channel/CLI
    participant BUS as MessageBus
    participant AL as AgentLoop
    participant SB as SessionManager
    participant CB as ContextBuilder
    participant LLM as LLMProvider
    participant TR as ToolRegistry

    U->>CH: Envía mensaje
    CH->>BUS: publish_inbound(InboundMessage)
    AL->>BUS: consume_inbound()
    AL->>SB: get_or_create(session_key)
    AL->>SB: get_history(memory_window)
    AL->>CB: build_messages(history + current_message)
    AL->>LLM: chat(messages, tools)

    alt LLM devuelve tool_calls
        loop por cada tool_call
            AL->>TR: execute(tool, args)
            TR-->>AL: resultado tool
            AL->>LLM: chat(mensajes + tool_result)
        end
    end

    AL->>SB: _save_turn(...)
    AL->>SB: save(session)
    AL->>BUS: publish_outbound(OutboundMessage)
    CH->>BUS: consume_outbound()
    CH-->>U: Respuesta final
```

## 2) Flujo de consolidación de memoria

```mermaid
flowchart TD
    A[Session messages] --> B{¿unconsolidated >= memory_window?}
    B -- No --> Z[No consolidar]
    B -- Sí --> C[Seleccionar mensajes antiguos]
    C --> D[Construir prompt de consolidación]
    D --> E[provider.chat con tool save_memory]
    E --> F{¿hubo tool_call válido?}
    F -- No --> G[Registrar warning y salir]
    F -- Sí --> H[append HISTORY.md]
    H --> I[write MEMORY.md]
    I --> J[Actualizar session.last_consolidated]
```

## 3) Flujo de salida y streaming de progreso

```mermaid
flowchart LR
    AL[AgentLoop] -->|OutboundMessage metadata._progress=true| BUS[MessageBus Outbound]
    BUS --> CM[ChannelManager dispatcher]
    CM --> F{send_progress/send_tool_hints}
    F -- permitido --> CH[Canal concreto]
    F -- bloqueado --> X[Descartar mensaje de progreso]
    CH --> USER[Usuario]
```

## 4) Flujo de heartbeat

```mermaid
sequenceDiagram
    participant HB as HeartbeatService
    participant LLM as LLMProvider
    participant AL as AgentLoop
    participant CH as ChannelManager

    HB->>HB: Leer HEARTBEAT.md
    HB->>LLM: _decide() vía tool heartbeat(action,tasks)
    alt action=skip
        HB-->>HB: no-op
    else action=run
        HB->>AL: on_execute(tasks)
        AL-->>HB: respuesta
        HB->>CH: on_notify(respuesta)
    end
```

## Observaciones de flujo

- El diseño tolera errores de tools devolviendo strings de error con hint de reintento.
- El loop de tools está acotado por `max_iterations` para evitar ciclos infinitos.
- El canal de progreso es opcional y configurable por `channels.send_progress` y `channels.send_tool_hints`.

