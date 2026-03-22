[01] Shopify Order Created
    ↓
Send WhatsApp Template (acknowledgement)

-------------------------------------

[Customer sends message]
    ↓

[03] WhatsApp Inbound Processing
    → Webhook receive
    → Airtable (message_buffer CREATE)
    → Sleep (30s buffer window)
    → Airtable (buffer SEARCH)
    → Aggregate messages (merge)
    → (optional) Image analysis (OpenAI)
    → Airtable (conversation history SEARCH)
    → OpenAI (classification)
    → Airtable (support_messages CREATE)
    → Trigger Scenario 07
    → Airtable (buffer DELETE)

-------------------------------------

[07] AI Reply Draft Generator
    → Airtable (GET record)
    → OpenAI (generate reply)
    → Airtable (UPDATE ai_suggested_reply)
    → Telegram SEND (support group)

-------------------------------------

[HUMAN CONTROL LAYER — TELEGRAM]

[00] Telegram Router
    ↓
    ├── Scenario 09 (Dispatcher commands)
    └── Scenario 08 (Support replies)

-------------------------------------

[08] Support Agent Action (MANDATORY)

Agent sees:
→ Customer message
→ AI suggested reply
→ Airtable record ID

Agent decides:
    ├── "send" → use AI reply
    └── custom reply → manual text

-------------------------------------

[08] Execute Response
    → Airtable (GET record)
    → WhatsApp SEND message
    → Airtable (log outbound)
    → conversation_status = waiting_customer

-------------------------------------

[Optional parallel flow]

[09] Dispatcher (Telegram logistics group)
    → Command (e.g. 4198-out)
    → Airtable (UPDATE order status)
    → Telegram confirmation

-------------------------------------

[Customer replies again]
    ↓
    LOOP back to Scenario 03

---
    🔑 KEY CHARACTERISTICS (your current system)
✅ Human approval ALWAYS required
nothing goes to customer automatically
Telegram = control layer
✅ Airtable = single source of truth
messages
order status (via dispatcher)
AI replies
✅ AI is advisory only
classify
suggest reply
no autonomous action

⚠️ CRITICAL CONTROL POINTS

Mark these mentally:

Scenario 03 → MUST produce 1 message only
Telegram message → MUST map to correct Airtable record
Dispatcher → MUST keep status updated
Scenario 08 → MUST send exactly once
