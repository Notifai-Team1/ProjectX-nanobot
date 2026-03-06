# Diagrama de clases y relaciones (Mermaid)

## Diagrama principal de clases

```mermaid
classDiagram
    class MessageBus {
      +Queue inbound
      +Queue outbound
      +publish_inbound(msg)
      +consume_inbound()
      +publish_outbound(msg)
      +consume_outbound()
    }

    class InboundMessage {
      +channel: str
      +sender_id: str
      +chat_id: str
      +content: str
      +media: list
      +metadata: dict
      +session_key: str
    }

    class OutboundMessage {
      +channel: str
      +chat_id: str
      +content: str
      +reply_to: str
      +media: list
      +metadata: dict
    }

    class AgentLoop {
      +run()
      +process_direct()
      -_process_message()
      -_run_agent_loop()
      -_register_default_tools()
      -_save_turn()
    }

    class ContextBuilder {
      +build_system_prompt()
      +build_messages()
      +add_assistant_message()
      +add_tool_result()
    }

    class SessionManager {
      +get_or_create(key)
      +save(session)
      +list_sessions()
      +invalidate(key)
    }

    class Session {
      +key
      +messages
      +last_consolidated
      +get_history(max_messages)
      +clear()
    }

    class MemoryStore {
      +read_long_term()
      +write_long_term(content)
      +append_history(entry)
      +consolidate(session, provider, model)
    }

    class ToolRegistry {
      +register(tool)
      +get_definitions()
      +execute(name, params)
      +tool_names
    }

    class Tool {
      <<abstract>>
      +name
      +description
      +parameters
      +execute(**kwargs)*
      +validate_params(params)
      +to_schema()
    }

    class BaseChannel {
      <<abstract>>
      +start()*
      +stop()*
      +send(msg)*
      +is_allowed(sender_id)
      +_handle_message(...)
    }

    class ChannelManager {
      +start_all()
      +stop_all()
      +get_status()
      -_dispatch_outbound()
      -_init_channels()
    }

    class LLMProvider {
      <<abstract>>
      +chat(messages, tools, model)*
      +get_default_model()*
    }

    class LiteLLMProvider {
      +chat(messages, tools, model)
      -_resolve_model()
      -_parse_response()
    }

    class SubagentManager {
      +spawn(task,...)
      +cancel(task_id)
      +cancel_by_session(session_key)
      -_run_subagent(...)
    }

    class HeartbeatService {
      +start()
      +stop()
      +trigger_now()
      -_tick()
      -_decide(content)
    }

    MessageBus --> InboundMessage
    MessageBus --> OutboundMessage
    AgentLoop --> MessageBus
    AgentLoop --> ContextBuilder
    AgentLoop --> SessionManager
    AgentLoop --> ToolRegistry
    AgentLoop --> LLMProvider
    AgentLoop --> SubagentManager
    AgentLoop --> MemoryStore

    SessionManager --> Session
    ContextBuilder --> MemoryStore

    ToolRegistry o--> Tool
    ChannelManager o--> BaseChannel
    LiteLLMProvider --|> LLMProvider
```

## Notas de diseño OO

- **`AgentLoop`** actúa como *application service* principal.
- **`Tool` y `BaseChannel`** definen contratos de extensión (polimorfismo).
- **`LLMProvider`** permite intercambiar backend sin tocar orquestación.
- **`ToolRegistry`** elimina dependencias rígidas y facilita composición de capacidades.

