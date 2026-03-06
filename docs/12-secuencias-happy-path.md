# Diagramas de Secuencia Happy Path

Secuencias principales en camino feliz para operación normal.

---

## 1) Happy path Mensaje simple sin tools

```mermaid
sequenceDiagram
    actor U as Usuario
    participant C as Canal
    participant B as MessageBus
    participant A as AgentLoop
    participant S as SessionManager
    participant P as Provider

    U->>C: Envia mensaje
    C->>B: publish inbound
    A->>B: consume inbound
    A->>S: get or create session
    A->>P: chat con contexto
    P-->>A: respuesta final assistant
    A->>S: save turn y save session
    A->>B: publish outbound
    C->>B: consume outbound
    C-->>U: entrega respuesta
```

## 2) Happy path Mensaje con tool calling

```mermaid
sequenceDiagram
    actor U as Usuario
    participant C as Canal
    participant B as MessageBus
    participant A as AgentLoop
    participant T as ToolRegistry
    participant P as Provider

    U->>C: Envia solicitud
    C->>B: publish inbound
    A->>B: consume inbound
    A->>P: chat con tools
    P-->>A: assistant con tool call
    A->>T: execute tool
    T-->>A: resultado tool
    A->>P: chat con resultado tool
    P-->>A: respuesta final
    A->>B: publish outbound
    C->>B: consume outbound
    C-->>U: respuesta final
```

## 3) Happy path Consolidacion incremental

```mermaid
sequenceDiagram
    participant A as AgentLoop
    participant M as MemoryStore
    participant P as Provider
    participant F as Filesystem
    participant S as Session

    A->>M: consolidate session incremental
    M->>M: seleccionar mensajes elegibles
    M->>P: chat con save memory tool
    P-->>M: tool call con history y memory update
    M->>F: append HISTORY
    M->>F: write MEMORY si cambia
    M->>S: actualizar last consolidated
    M-->>A: true
```

## 4) Happy path Comando stop

```mermaid
sequenceDiagram
    actor U as Usuario
    participant C as Canal
    participant B as MessageBus
    participant A as AgentLoop
    participant SA as SubagentManager

    U->>C: Envia stop
    C->>B: publish inbound
    A->>B: consume inbound
    A->>A: cancelar tasks por session key
    A->>SA: cancel by session
    SA-->>A: total subagentes cancelados
    A->>B: publish outbound confirmacion
    C->>B: consume outbound
    C-->>U: mensaje de tareas detenidas
```
