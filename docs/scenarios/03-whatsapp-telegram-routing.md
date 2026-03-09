---
scenario_id: 03
scenario_name: whatsapp_to_telegram_routing
system: make.com
trigger: webhook (WhatsApp Cloud API)
owner: Vahagn Ohanjanyan
status: active
last_updated: 2026-03-10
---

# Scenario 3 — WhatsApp Number A → Telegram Routing

## Purpose

This scenario routes incoming customer messages from **WhatsApp Number A (automation number)** into an internal **Telegram support group**, allowing human operators to reply directly through Telegram.

It acts as the bridge between:

Customer → WhatsApp → Make Automation → Telegram → Staff → WhatsApp reply

This enables a lightweight support workflow without building a full CRM interface.

---

# Automation Overview

Workflow:

Customer WhatsApp Message
↓
WhatsApp Cloud API Webhook
↓
Make Scenario Trigger
↓
Parse Incoming Message
↓
Normalize Phone Number
↓
Log Message to Google Sheets
↓
Send Message to Telegram Group
↓
Staff Reply via Telegram Command
↓
Reply Sent Back via WhatsApp API

---

# Module 1 — Webhook: Receive WhatsApp Message

Purpose:

Receive incoming customer messages from **WhatsApp Business Cloud API**.

Webhook receives payload containing:

- sender phone number
- message text
- WhatsApp message ID
- timestamp

Example payload:

```json
{
  "messages": [
    {
      "from": "971525970815",
      "id": "wamid.HBgMOTcxNTI1OTcwODE1FQIAEhgg...",
      "timestamp": "1709834812",
      "text": {
        "body": "Здравствуйте, у вас есть Боржоми?"
      },
      "type": "text"
    }
  ]
}
```

Key extracted fields:

- customer_phone
- message_text
- message_id

---

# Module 2 — Tools: Set Multiple Variables (Phone Normalization)

Purpose:

Normalize the customer phone number for consistent processing and logging.

Characters removed:

- "+"
- spaces
- "-"
- "("
- ")"

Normalization rules:

- If number starts with **5** → prefix **971**
- If number starts with **0** → remove **0** and prefix **971**
- Otherwise → keep as is

Example normalized value:

971525970815

---

# Module 3 — Google Sheets: Add Row (Message Log)

Sheet:

`whatsapp_incoming_messages`

Purpose:

Log all incoming WhatsApp messages for traceability and support history.

Fields stored:

- timestamp
- customer_phone
- message_text
- message_id
- message_status
- assigned_operator (optional)

Example row created:

```json
{
  "timestamp": "2026-03-10 14:12",
  "customer_phone": "971525970815",
  "message_text": "Здравствуйте, у вас есть Боржоми?",
  "message_status": "new"
}
```

---

# Module 4 — Telegram: Send Message to Support Group

Purpose:

Forward the customer message into the **Telegram support group**.

Telegram message format:

📩 New WhatsApp Message

Customer: +971525970815

Message:
Здравствуйте, у вас есть Боржоми?

Reply using:
/reply 971525970815 your_message

This allows support staff to reply directly from Telegram.

---

# Module 5 — Telegram Bot Command Listener

Purpose:

Listen for staff replies using the command format.

Example command:

/reply 971525970815 Да, Боржоми есть. Сколько бутылок нужно?

Parsed values:

- phone number
- reply message

Extracted variables:

- reply_phone
- reply_message

---

# Module 6 — WhatsApp Business Cloud API: Send Reply

Purpose:

Send the staff response back to the customer via WhatsApp.

Example API request:

```json
{
  "messaging_product": "whatsapp",
  "to": "971525970815",
  "type": "text",
  "text": {
    "body": "Да, Боржоми есть. Сколько бутылок нужно?"
  }
}
```

Example API response:

```json
{
  "messages": [
    {
      "id": "wamid.HBgMOTcxNTI1OTcwODE1FQIAEhgg..."
    }
  ]
}
```

---

# Control Logic

Key rules:

- All incoming messages are logged before routing.
- Telegram operators reply using `/reply` command.
- Replies are sent through the **same WhatsApp automation number (Number A)**.
- Customer replies continue the conversation thread.

---

# System Components Used

| Component | Purpose |
|----------|--------|
| WhatsApp Cloud API | Receive and send messages |
| Make.com | Automation workflow |
| Google Sheets | Message logging |
| Telegram Bot | Staff interface |
| Telegram Group | Internal support communication |

---

# Outcome

This scenario enables **human-assisted WhatsApp support without building a custom interface**, using Telegram as the operator console.

Benefits:

- extremely fast to implement
- no CRM required
- all messages logged
- scalable with AI classification later

---

# Future Improvements

Possible enhancements:

- AI intent detection for messages
- automatic FAQ responses
- priority routing (orders / complaints / delivery issues)
- automatic customer profile creation
- integration with Airtable CRM
