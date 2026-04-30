---
name: "{NOME_CLINICA}"
description: "Agendamento de consultas odontológicas"
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
  - agent_name: Recepcionista
    agent_role: pilot
    model: gpt-5.4-mini
    instructions_file: recepcionista-instructions.md
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

# Odonto - Agendamento

Template para agendamento de consultas em clínicas odontológicas via WhatsApp.

## Casos de Uso

- Agendar consultas e avaliações
- Remarcar ou cancelar consultas
- Tirar dúvidas sobre procedimentos
- Informar sobre valores e formas de pagamento
- Enviar lembretes de consulta

## Checklist de Customização

- [ ] Substituir `{NOME_CLINICA}` pelo nome da clínica
- [ ] Adicionar lista de procedimentos e valores
- [ ] Configurar horários de funcionamento
- [ ] Definir convênios aceitos
- [ ] Configurar agenda no Google Calendar
- [ ] Adicionar instruções de pré-consulta
