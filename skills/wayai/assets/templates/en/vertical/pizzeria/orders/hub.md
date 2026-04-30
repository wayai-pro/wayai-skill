---
name: "{COMPANY_NAME}"
description: "Order intake for a pizzeria"
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
  - agent_name: Server
    agent_role: pilot
    model: gpt-5.4-mini
    instructions_file: server-instructions.md
---

# Pizzeria - Orders

Template for taking pizzeria orders over WhatsApp.

## Use Cases

- Take pizza orders
- Answer menu questions
- Share delivery times
- Modify orders before they enter the kitchen

## Customization Checklist

- [ ] Replace `{COMPANY_NAME}` with the pizzeria's name
- [ ] Add the full menu with prices
- [ ] Define delivery area and fees
- [ ] Configure accepted payment methods
- [ ] Set the average delivery time
- [ ] Add active promotions
