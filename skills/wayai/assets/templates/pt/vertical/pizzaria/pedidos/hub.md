---
name: "{NOME_EMPRESA}"
description: "Atendimento de pedidos para pizzaria"
ai_mode: pilot+copilot
hub_type: chat
connections:
  - connector_name: OpenAI
    connector_id: "0cd6a292-895b-4667-b89e-dd298628c272"
    connector_type: Agent
  - connector_name: WhatsApp
    connector_id: "5fb214cb-aaa8-4b3d-8c65-c9370b3e7c85"
    connector_type: Channel
agents:
  - agent_name: Atendente
    agent_role: pilot
    model: gpt-5.4-mini
    instructions_file: atendente-instructions.md
---

# Pizzaria - Pedidos

Template para atendimento de pedidos de pizzaria via WhatsApp.

## Casos de Uso

- Receber pedidos de pizza
- Tirar dúvidas sobre cardápio
- Informar tempo de entrega
- Modificar pedidos antes do preparo

## Checklist de Customização

- [ ] Substituir `{NOME_EMPRESA}` pelo nome da pizzaria
- [ ] Adicionar cardápio completo com preços
- [ ] Definir área de entrega e taxas
- [ ] Configurar formas de pagamento aceitas
- [ ] Definir tempo médio de entrega
- [ ] Adicionar promoções ativas
