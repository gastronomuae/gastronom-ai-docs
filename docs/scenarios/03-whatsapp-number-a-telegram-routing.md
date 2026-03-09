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
{{4.entry[].changes[].value.messages[].text.body}} = EXISTS
```


This prevents triggering on:

- delivery receipts
- message status updates
- read receipts
- webhook status callbacks
- system / non-text events

This prevents creation of empty Airtable rows and eliminates duplicate executions triggered by WhatsApp status webhooks.

---

# Module 3 — Router

Purpose:

Split the workflow into **two parallel processing paths**:

Branch 1 → Send message to Telegram support group 

Filter Conditions: Message only filter

```
{{4.entry[].changes[].field}} = MESSAGES
```
AND

```
{{4.entry[].changes[].value.messages}} = EXISTS
```
Branch 2 → Log structured conversation record in Airtable

This allows both **real-time operator visibility** and **AI-ready message storage**.

---

# Module 4 — Telegram Bot: Send Text (BRANCH 1)

Purpose:

Forward the incoming customer message to the **internal Telegram support group**.

Telegram group chat ID: -5133624518

Message format sent to Telegram:

```
📩 New WhatsApp Message

Customer:
+971525970815

Message:
Здравствуйте, у вас есть Боржоми?
```

This enables operators to see incoming messages instantly.

This includes:
- Sender phone (wa_id)
- Message text content

If later you support media:
- Add media URL
- Add message type

Example output:

```json
[
{
"message_id": 48,
"from": {
"id": 8135508887,
"is_bot": true,
"first_name": "gastronom.support",
"username": "gastronomae_bot"
},
"chat": {
"id": -5133624518,
"title": "Gastronom cusomer support",
"type": "group",
"all_members_are_administrators": true,
"accepted_gift_types": {
"unlimited_gifts": false,
"limited_gifts": false,
"unique_gifts": false,
"premium_subscription": false,
"gifts_from_channels": false
}
},
"date": 1771682555,
"text": "📩 New WhatsApp Message\n\n👤 WA A: 971561345294\n\n💬 Thanks. When expect delivery?"
}
]
```

---

# Module 5 — Webhook Response (BRANCH 1)

Purpose:

Return **HTTP 200 OK** to Meta so the webhook event is acknowledged.

This prevents retries from the WhatsApp Cloud API.

Verification Logic

```
{{if(4.`hub.verify_token` = "gastronom2026"; 4.`hub.challenge`; """ """)}}
```

Response body example:

```
EVENT_RECEIVED
```
Purpose:
- Handle Meta webhook verification challenge
- Return challenge string when verify token matches
- Required during initial webhook setup

What This Scenario Currently Does

✅ Receives inbound WA messages 
✅ Filters only message events 
✅ Forwards to Telegram group 
✅ Responds 200 OK to Meta 
✅ Handles webhook verification


What It Does NOT Do (Yet)
- No AI classification
- No order lookup
- No auto-replies
- No retry handling

It is a pure forwarder.

Important Technical Note

You are using:

Custom Webhook module NOT WhatsApp “Watch messages” module

This means:
- You have full payload control
- You must maintain webhook verification logic
- You control filtering manually

Which is actually good for long-term AI expansion.


---

# Module 6 — Airtable: Create Record (Inbound Log) (BRANCH 2)

Base: AI Staff – Conversation Engine Table: conversation_log

Purpose: 
Create structured inbound message records for analytics, AI training, and conversation lifecycle tracking.

Data Handling Principles
- Only messages containing text.body are logged
- Status events (value.statuses[]) are ignored
- Each inbound message creates exactly one Airtable row
- conversation_hash = normalized wa_number
- conversation_started_at = inbound message timestamp (UTC)
  
Example record:

```json
[ { "conversation_id": "wamid.HBgMOTcxNTYxMzQ1Mjk0FQIAEhgUM0E1RjUxNUE0RjZCMDkyRkVDRkEA",
  "wa_number": "971561345294",
  "message_id_external": "wamid.HBgMOTcxNTYxMzQ1Mjk0FQIAEhgUM0E1RjUxNUE0RjZCMDkyRkVDRkEA",
  "message_direction": "inbound",
  "message_source": "whatsapp_A",
  "message_text": "Can you deliver after 5pm?",
  "timestamp_utc": "2026-03-05T07:39:29.000Z",
  "resolution_status": "unresolved",
  "conversation_status": "open",
  "issue_category": "order_status",
  "confi dence_score": 1,
  "channel": "whatsapp_A",
  "conversation_started_at": "2026-03-05T07:39:29.000Z",
  "conversation_hash": "971561345294",
  "Priority": "high",
  "broad_category": "support",
  "label_source": "system_default",
  "id": "recvxte3kd6bsCGZr",
  "createdTime": "2026-03-05T11:39:32.000Z" }
]
```
