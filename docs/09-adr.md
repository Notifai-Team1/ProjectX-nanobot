# Architecture Decision Records ADR

Este documento resume decisiones arquitectónicas clave del proyecto en formato ADR compacto.

---

## ADR 001 Mensajeria desacoplada con colas async

- **Estado:** Aceptado
- **Contexto:** Se requiere desacoplar canales externos del core de agente para tolerar variabilidad de latencias y procesamiento.
- **Decisión:** Usar `MessageBus` con colas asíncronas `inbound` y `outbound`.
- **Consecuencias:**
  - Positivo: aislamiento entre I O de canales y lógica de agente.
  - Positivo: posibilidad de escalar consumidores y productores de forma independiente.
  - Negativo: mayor necesidad de observabilidad de cola y control de backlog.

## ADR 002 Sesiones en JSONL append only

- **Estado:** Aceptado
- **Contexto:** Se necesita persistencia simple, legible y robusta para historial conversacional.
- **Decisión:** Persistir sesiones como JSONL con primera línea de metadata más líneas de mensajes.
- **Consecuencias:**
  - Positivo: inspección manual sencilla y recuperación tolerante.
  - Positivo: modelo natural para escritura append only.
  - Negativo: `save` reescribe archivo completo y crece costo O n con sesiones largas.

## ADR 003 Memoria en dos capas MEMORY y HISTORY

- **Estado:** Aceptado
- **Contexto:** Mantener contexto útil sin inflar indefinidamente el prompt ni perder trazabilidad histórica.
- **Decisión:** Separar memoria viva en `MEMORY.md` y trazabilidad en `HISTORY.md`.
- **Consecuencias:**
  - Positivo: prompt compacto con hechos relevantes.
  - Positivo: auditoría cronológica mantenida por `HISTORY.md`.
  - Negativo: mayor complejidad operativa en consolidación y consistencia.

## ADR 004 Consolidacion delegada al LLM mediante tool call estructurada

- **Estado:** Aceptado
- **Contexto:** Se requiere resumir y actualizar memoria semántica sin codificar reglas rígidas por dominio.
- **Decisión:** Forzar contrato `save_memory` con `history_entry` y `memory_update` en una tool call.
- **Consecuencias:**
  - Positivo: flexibilidad semántica y adaptación por contexto.
  - Positivo: interfaz estable para proveedores.
  - Negativo: dependencia de calidad del modelo y manejo de fallos de tool call.

## ADR 005 Registro unificado de herramientas

- **Estado:** Aceptado
- **Contexto:** El agente necesita exponer capacidades heterogéneas al modelo de forma uniforme.
- **Decisión:** Centralizar tools en `ToolRegistry` con validación por JSON Schema y ejecución robusta.
- **Consecuencias:**
  - Positivo: integración consistente y extensibilidad modular.
  - Positivo: errores convertidos a texto para auto corrección del agente.
  - Negativo: riesgo de errores silenciosos si no hay telemetría suficiente.

## ADR 006 Bucle iterativo LLM con tool calling

- **Estado:** Aceptado
- **Contexto:** Resolver tareas multi paso que requieren observación y acciones externas.
- **Decisión:** Ejecutar iteraciones `provider chat` hasta respuesta final o límite de iteraciones.
- **Consecuencias:**
  - Positivo: permite razonamiento herramienta observación ajuste.
  - Positivo: habilita tareas complejas en tiempo de ejecución.
  - Negativo: costo variable de tokens y latencia acumulada.

## ADR 007 Control explicito de cancelacion por sesion

- **Estado:** Aceptado
- **Contexto:** Se necesita capacidad de detener procesamiento en curso por usuario.
- **Decisión:** Soportar comando `stop` que cancela tasks activas y subagentes por `session_key`.
- **Consecuencias:**
  - Positivo: control operativo y mejor experiencia de usuario.
  - Positivo: evita trabajo inútil y costos adicionales.
  - Negativo: requiere cuidado para cleanup y consistencia tras cancelación.
