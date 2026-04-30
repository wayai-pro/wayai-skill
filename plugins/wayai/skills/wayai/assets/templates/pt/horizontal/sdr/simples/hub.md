---
name: "{NOME_EMPRESA} - SDR"
description: "Qualificação de leads inbound"
ai_mode: pilot+copilot
hub_type: chat
connections:
  - connector_name: OpenAI
    connector_id: "0cd6a292-895b-4667-b89e-dd298628c272"
    connector_type: Agent
  - connector_name: WhatsApp
    connector_id: "5fb214cb-aaa8-4b3d-8c65-c9370b3e7c85"
    connector_type: Channel
  - connector_name: Google Calendar
    connector_id: "189c2e74-2275-43b6-8dac-0fb3b782e9de"
    connector_type: Tool - Native
agents:
  - agent_name: SDR
    agent_role: pilot
    model: gpt-5.4-mini
    instructions_file: sdr-instructions.md
    tools:
      native:
        - connector_name: Google Calendar
          connector_id: "189c2e74-2275-43b6-8dac-0fb3b782e9de"
          tools:
            - tool_name: Create Event
              tool_native_id: "2482de79-2f7d-444f-a6a1-e943faf59ec6"
            - tool_name: List Events
              tool_native_id: "37f60e18-eb76-4efa-968d-1f961bd8325d"
            - tool_name: Check Availability
              tool_native_id: "a5e8c649-0f7d-4b3e-b9ac-96efb8e4c93b"
---

# SDR - Simples

Template para SDR (Sales Development Representative) focado em qualificação de leads inbound.

## Casos de Uso

- Qualificar leads vindos do site/anúncios
- Responder dúvidas sobre o produto
- Agendar reuniões com o time comercial
- Nutrir leads que não estão prontos

## Checklist de Customização

- [ ] Substituir `{NOME_EMPRESA}` pelo nome da empresa
- [ ] Substituir `{NOME_PRODUTO}` pelo nome do produto
- [ ] Definir critérios BANT específicos
- [ ] Adicionar informações do produto
- [ ] Configurar respostas para objeções comuns
- [ ] Configurar integração com CRM (opcional)
- [ ] Conectar Google Calendar
