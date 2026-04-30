---
name: "{ACADEMY_NAME}"
description: "Visit scheduling for a swimming academy"
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
  - connector_name: Wayai Conversation
    connector_id: "b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4"
    connector_type: Tool - Native
agents:
  - agent_name: Agent
    agent_role: pilot
    model: gpt-5.4-mini
    instructions_file: agent-instructions.md
    tools:
      native:
        - connector_name: Google Calendar
          connector_id: "189c2e74-2275-43b6-8dac-0fb3b782e9de"
          tools:
            - tool_name: Check Availability
              tool_native_id: "a5e8c649-0f7d-4b3e-b9ac-96efb8e4c93b"
            - tool_name: Create Event
              tool_native_id: "2482de79-2f7d-444f-a6a1-e943faf59ec6"
            - tool_name: Update Event
              tool_native_id: "24f82d08-ee88-439e-851f-a33f48c8471e"
            - tool_name: Delete Event
              tool_native_id: "763413b8-4464-44d2-989e-682d4c2e8385"
        - connector_name: Wayai Conversation
          connector_id: "b17d9f3a-4e1b-46c9-b648-a2f0c3611aa4"
          tools:
            - tool_name: Transfer to Team
              tool_native_id: "1fcac563-34d5-4546-80cd-9ac9c3f19ef7"
---

# Swimming - Customer Service

Template for swimming academies on WhatsApp, focused on **booking visits** to convert prospects into students.

## Use Cases

- **Book visits to tour the academy** (primary objective)
- Reschedule or cancel visits
- Share class types and schedules
- Answer questions about plans and pricing
- Walk through the enrollment process
- Cover pool rules and what to bring

## Customization Checklist

- [ ] Replace `{ACADEMY_NAME}` with the academy's name
- [ ] Add classes offered (kids' swim, adult, water aerobics, etc.)
- [ ] Define class schedules
- [ ] Configure plans and pricing (monthly, quarterly, annual)
- [ ] List operating hours
- [ ] Add pool rules
- [ ] Define enrollment requirements
- [ ] Configure Google Calendar for visit scheduling
- [ ] Configure team for handoff
