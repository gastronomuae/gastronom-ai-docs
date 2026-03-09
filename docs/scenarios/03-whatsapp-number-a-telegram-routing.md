
---
scenario_id: 03
scenario_name: whatsapp_number_a_telegram_routing
system: make.com
trigger: whatsapp cloud api webhook
status: active
---

# Scenario 3 — WhatsApp Number A → Telegram Routing

## Purpose

Forward inbound WhatsApp messages received on **WhatsApp Number A (Cloud API)** into the internal **Telegram support group** while also logging the message into Airtable for structured conversation tracking.

This scenario acts as the **entry point for customer support conversations** and prepares the data for future AI processing.

---

# Architecture Overview

Webhook (Meta)

↓

Filter (Only inbound text messages)

↓

Router

Branch 1 → Telegram Bot (Send Text)

Branch 2 → Airtable (Create Record – Inbound Log)

↓

Webhook Response (200 OK)

---

# Module 1 — Webhooks: Custom Webhook

Purpose:

Receive inbound WhatsApp events from the **Meta WhatsApp Cloud API**.

Event Source:

Meta webhook subscription (`messages` field)

Data received:

- entry[]
- changes[]
- value.messages[]
- from
- text.body
- timestamp

Example output:

```json
[
  {
    "object": "whatsapp_business_account",
    "entry": [
      {
        "id": "1384757296311650",
        "changes": [
          {
            "value": {
              "messaging_product": "whatsapp",
              "metadata": {
                "display_phone_number": "15551531305",
                "phone_number_id": "1016151078247095"
              },
              "contacts": [
                {
                  "profile": {
                    "name": "vohanjanyan"
                  },
                  "wa_id": "971525970815"
                }
              ],
              "messages": [
                {
                  "from": "971525970815",
                  "timestamp": "1709834812",
                  "text": {
                    "body": "Здравствуйте, у вас есть Боржоми?"
                  },
                  "type": "text"
                }
              ]
            }
          }
        ]
      }
    ]
  }
]
```

---

# Module 2 — Filter: Inbound Text Messages Only

Purpose:

Ensure only **customer text messages** trigger the automation.

Filter Conditions:

```
value.messages[0].type = text
```

Optional additional protection:

```
value.messages exists
```

This prevents triggering on:

- delivery receipts
- message status updates
- template acknowledgements
- system events

---

# Module 3 — Router

Purpose:

Split the workflow into **two parallel processing paths**:

Branch 1 → Send message to Telegram support group  
Branch 2 → Log structured conversation record in Airtable

This allows both **real-time operator visibility** and **AI-ready message storage**.

---

# Module 4 — Telegram Bot: Send Text

Purpose:

Forward the incoming customer message to the **internal Telegram support group**.

Message format sent to Telegram:

```
📩 New WhatsApp Message

Customer:
+971525970815

Message:
Здравствуйте, у вас есть Боржоми?
```

Optional enhanced format:

```
📩 WhatsApp

Customer: +971525970815
Name: vohanjanyan

Message:
Здравствуйте, у вас есть Боржоми?
```

This enables operators to see incoming messages instantly.

---

# Module 5 — Airtable: Create Record (Inbound Log)

Table:

`whatsapp_messages`

Purpose:

Store inbound messages in a structured format for:

- AI training
- conversation history
- analytics
- support tracking

Fields stored:

| Field | Value |
|-----|-----|
timestamp | message timestamp |
phone | customer phone |
name | WhatsApp profile name |
message_text | message body |
channel | whatsapp |
direction | inbound |
status | new |

Example record:

```json
{
  "phone": "971525970815",
  "name": "vohanjanyan",
  "message_text": "Здравствуйте, у вас есть Боржоми?",
  "channel": "whatsapp",
  "direction": "inbound",
  "status": "new"
}
```

---

# Module 6 — Webhook Response

Purpose:

Return **HTTP 200 OK** to Meta so the webhook event is acknowledged.

This prevents retries from the WhatsApp Cloud API.

Response body example:

```
EVENT_RECEIVED
```

---

# Control Logic

Key rules:

• Only inbound text messages are processed  
• Messages are forwarded to Telegram instantly  
• All messages are stored in Airtable  
• No reply is sent automatically in this scenario

---

# System Components Used

| Component | Purpose |
|------|------|
WhatsApp Cloud API | Receive messages |
Make.com | Automation workflow |
Telegram Bot | Internal support notifications |
Airtable | Conversation logging |

---

# Outcome

This scenario creates the **core messaging bridge** between:

Customer WhatsApp  
→ Make automation  
→ Telegram support team  
→ AI-ready conversation database

It forms the **foundation for the future AI customer support system**.
