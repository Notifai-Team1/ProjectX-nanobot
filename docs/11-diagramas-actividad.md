# Diagramas de Actividad UML

Diagramas de actividad para procesos principales del runtime.

---

## 1) Actividad Procesar mensaje en AgentLoop

```mermaid
flowchart TD
    A[Inicio] --> B[Consumir InboundMessage]
    B --> C{Comando stop}
    C -- si --> D[Cancelar tareas y subagentes]
    D --> E[Publicar confirmacion]
    E --> Z[Fin]
    C -- no --> F[Cargar o crear sesion]
    F --> G{Comando new o help}
    G -- si --> H[Ejecutar control de comando]
    H --> I[Publicar respuesta de control]
    I --> Z
    G -- no --> J[Construir contexto y mensajes]
    J --> K[Ejecutar bucle LLM tools]
    K --> L[Guardar turno y sesion]
    L --> M{Message tool ya respondio}
    M -- si --> Z
    M -- no --> N[Publicar respuesta final]
    N --> Z
```

## 2) Actividad Ejecutar tool call en ToolRegistry

```mermaid
flowchart TD
    A[Inicio] --> B[Recibir nombre y parametros]
    B --> C{Tool existe}
    C -- no --> D[Retornar error tool not found]
    D --> Z[Fin]
    C -- si --> E[Validar parametros]
    E --> F{Validacion correcta}
    F -- no --> G[Retornar error validacion]
    G --> Z
    F -- si --> H[Ejecutar tool]
    H --> I{Resultado error textual}
    I -- si --> J[Adjuntar hint de reintento]
    J --> K[Retornar resultado]
    I -- no --> K
    K --> Z
```

## 3) Actividad Consolidar memoria

```mermaid
flowchart TD
    A[Inicio] --> B[Calcular rango de mensajes a consolidar]
    B --> C{Hay mensajes elegibles}
    C -- no --> D[Retornar false]
    D --> Z[Fin]
    C -- si --> E[Construir prompt de consolidacion]
    E --> F[Llamar provider con save memory]
    F --> G{Tool call valida}
    G -- no --> H[Retornar false]
    H --> Z
    G -- si --> I[Extraer history entry y memory update]
    I --> J[Append HISTORY]
    I --> K[Actualizar MEMORY si cambia]
    J --> L[Actualizar last consolidated]
    K --> L
    L --> M[Retornar true]
    M --> Z
```

## 4) Actividad Persistir sesion JSONL

```mermaid
flowchart TD
    A[Inicio] --> B[Resolver path de sesion]
    B --> C[Construir linea metadata]
    C --> D[Escribir metadata en archivo]
    D --> E[Iterar mensajes]
    E --> F[Escribir mensaje JSON por linea]
    F --> G{Quedan mensajes}
    G -- si --> F
    G -- no --> H[Cerrar archivo]
    H --> Z[Fin]
```
