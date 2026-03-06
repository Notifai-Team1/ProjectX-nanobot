# Documentación de API Swagger OpenAPI

## Objetivo

Definir un contrato HTTP formal para integración externa con `nanobot`, alineado al comportamiento del runtime actual (bus de mensajes, sesiones, memoria y tools).

---

## Archivo fuente OpenAPI

- Especificación principal: [`openapi.yaml`](./openapi.yaml)
- Versión de OpenAPI: `3.0.3`
- URL de servidor ejemplo: `http://localhost:8080`

---

## Cobertura funcional

La especificación cubre cinco grupos principales:

1. **Health**
   - `GET /health`
2. **Mensajes**
   - `POST /v1/messages/inbound`
   - `GET /v1/messages/outbound/next`
3. **Sesiones**
   - `GET /v1/sessions`
   - `GET /v1/sessions/{sessionKey}`
   - `DELETE /v1/sessions/{sessionKey}`
   - `GET /v1/sessions/{sessionKey}/history`
4. **Memoria**
   - `GET /v1/memory`
   - `PUT /v1/memory`
   - `POST /v1/memory/history`
   - `POST /v1/memory/consolidate`
5. **Tools**
   - `GET /v1/tools`
   - `POST /v1/tools/{toolName}/execute`

---

## Modelos principales

- `InboundMessage`
- `OutboundMessage`
- `Session`
- `SessionMessage`
- `SessionSummary`
- `ErrorResponse`

Estos modelos están alineados con las estructuras del runtime documentadas previamente.

---

## Uso con Swagger UI

Opciones rápidas:

1. Abrir [`openapi.yaml`](./openapi.yaml) en cualquier visor OpenAPI compatible.
2. Integrar con Swagger UI o Redoc en una app de documentación interna.
3. Generar SDKs cliente con herramientas de OpenAPI Generator.

---

## Notas de alcance

- El repositorio actual implementa el runtime principal y canales, no un servidor HTTP nativo completo.
- Esta especificación define una **API de fachada recomendada** para exponer capacidades de `nanobot` vía HTTP sin ambigüedad de contrato.
- El diseño evita acoplar clientes externos a detalles internos de colas y archivos JSONL.
