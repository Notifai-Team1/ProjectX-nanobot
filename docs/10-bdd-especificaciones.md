# Especificaciones BDD

Escenarios Behavior Driven Development para flujos principales del sistema.

---

## Feature 1 Procesamiento de mensaje normal

```gherkin
Feature: Procesamiento de mensaje de usuario
  Como usuario de un canal integrado
  Quiero enviar un mensaje normal
  Para recibir una respuesta del agente

  Scenario: Mensaje sin uso de herramientas
    Given existe una sesion para el chat
    And el provider responde sin tool calls
    When el usuario envia un mensaje de texto
    Then el agente guarda el turno en la sesion
    And publica un OutboundMessage con la respuesta final
```

## Feature 2 Tool calling iterativo

```gherkin
Feature: Ejecucion iterativa con herramientas
  Como agente runtime
  Quiero ejecutar tool calls del modelo
  Para completar tareas de varios pasos

  Scenario: El modelo solicita una tool y luego finaliza
    Given hay una tool registrada con schema valido
    And el provider devuelve tool calls en la primera iteracion
    And la tool responde exitosamente
    When el agente ejecuta el bucle iterativo
    Then el resultado de tool se agrega al historial como role tool
    And el agente realiza una nueva llamada al provider
    And finalmente retorna contenido final de assistant
```

## Feature 3 Validacion de parametros de tools

```gherkin
Feature: Validacion de parametros de herramientas
  Como sistema
  Quiero validar parametros antes de ejecutar tools
  Para reducir errores evitables

  Scenario: Parametros invalidos
    Given una tool registrada con JSON schema
    When el agente intenta ejecutar la tool con parametros invalidos
    Then ToolRegistry retorna un error de validacion en texto
    And incluye una sugerencia para reintento inteligente
```

## Feature 4 Comando stop

```gherkin
Feature: Cancelacion de tareas por sesion
  Como usuario
  Quiero detener tareas activas
  Para recuperar control del procesamiento

  Scenario: Existen tareas activas
    Given hay tareas activas asociadas a la sesion
    And hay subagentes activos para la misma sesion
    When el usuario envia el comando stop
    Then el sistema cancela tareas activas de la sesion
    And cancela subagentes asociados
    And publica un mensaje con el total de tareas detenidas
```

## Feature 5 Consolidacion incremental de memoria

```gherkin
Feature: Consolidacion de memoria
  Como runtime
  Quiero consolidar mensajes antiguos
  Para mantener contexto util y control de tamano de historial

  Scenario: Consolidacion incremental exitosa
    Given la sesion supera la ventana de memoria configurada
    And existe progreso desde last consolidated
    And el provider devuelve tool call save memory valida
    When se ejecuta consolidate en modo incremental
    Then se agrega history entry en HISTORY md
    And se actualiza MEMORY md cuando hay cambios
    And se actualiza last consolidated de la sesion
```

## Feature 6 Nueva conversacion

```gherkin
Feature: Reinicio controlado de contexto
  Como usuario
  Quiero iniciar una nueva conversacion
  Para separar contexto actual del siguiente tema

  Scenario: Comando new con consolidacion previa
    Given existe una sesion con mensajes acumulados
    When el usuario envia el comando new
    Then el sistema intenta consolidar memoria con archive all
    And si consolidacion es exitosa limpia la sesion
    And responde confirmando nueva conversacion
```
