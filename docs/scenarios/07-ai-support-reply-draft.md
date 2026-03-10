# Scenario: 07-ai-support-reply-draft

## Purpose

Generate AI-suggested replies for customer support messages coming from Instagram and WhatsApp after classification and storage in Airtable.

The scenario receives structured message data via webhook, generates a suggested reply using OpenAI, stores the suggestion in Airtable, and posts the message plus recommendation to the Telegram support group for human approval.

---

# Architecture Overview

```
Instagram / WhatsApp
        ↓
Classification Scenarios
        ↓
Airtable Record Created
        ↓
HTTP Request
        ↓
07-ai-support-reply-draft
        ↓
AI Reply Generated
        ↓
Airtable Updated
        ↓
Telegram Notification
```

---

# Trigger

## Webhook

Module: **Custom Webhook**

Expected payload structure:

```json
{
  "airtable_record_id": "recXXXX",
  "channel": "instagram",
  "message_text": "Customer message text",
  "broad_category": "support",
  "issue_category": "product_question",
  "priority": "normal",
  "message_direction": "inbound"
}
```

---

# Step 1 — Filter

Filter name:

```
support_only
```

Condition:

```
broad_category = support
```

Purpose:

Ensure AI replies are generated **only for support messages**.

Other categories are ignored:

- `supplier_prospect`
- `b2b_sales`
- `marketing`
- `other`

---

# Step 2 — OpenAI Reply Generation

Module:

```
OpenAI → Generate Completion
```

Model:

```
gpt-4o-mini
```

---

## System Prompt

```
You are a support assistant for Gastronom.ae,
an online Russian grocery store delivering in Dubai.

Write a short and friendly reply to the customer.

Rules:

• Detect the language of the customer message and reply ONLY in that language.
• If the customer writes in English → reply in English.
• If the customer writes in Russian → reply in Russian.
• Never switch languages.
• Keep replies short (1–3 sentences)
• Be polite and natural
• Do not invent information


General guidance:
• If the message starts with a greeting → acknowledge it briefly.  Light friendly emoji (😊) may be used occasionally but not excessively
• Avoid repeating "please" multiple times in one sentence
• Delivery usually happens the same day if the order is placed earlier in the day, but it depends on workload and routing.
• If delivery timing is asked → say we will check the order and confirm shortly.
• Never guarantee exact delivery time unless confirmed.

Order questions:

• If a customer asks about delivery timing or order status and no order number is provided → ask for the order number first.

Product questions:

• If product availability is asked → say we will check availability or suggest checking the website.
• If an order number is mentioned → format it as #NNNN if it is a 4-digit number

Delivery questions:
• If customer asks about delivery area → confirm that we deliver across Dubai.
• Currently delivery is available within Dubai only.
• If a customer asks about delivery to another emirate (for example Abu Dhabi, Sharjah, etc.) → politely explain that delivery is currently limited to Dubai.
• Offer to assist if they will be in Dubai or plan to order within Dubai.


Complaints:

• Apologize politely
• Say we will check the issue and get back shortly.

Tone:

• Friendly
• Helpful
• Short
• Natural (not robotic)

Return only the reply text.

```

---

## User Input

```
Customer message:
{{message_text}}
```

---

# Step 3 — Airtable Update

Module:

```
Airtable → Update Record
```

Record ID:

```
{{airtable_record_id}}
```

Field updated:

```
ai_suggested_reply
```

Value:

```
OpenAI result
```

---

# Step 4 — Telegram Notification

Module:

```
Telegram Bot → Send Message
```

Example message format:

```
📩 New customer message

📢 {{9.channel}}
👤 {{9.message_text}}
🤖 AI Suggested reply:
{{9.ai_suggested_reply}}
🆔 {{9.id}}

Commands:
send
/reply your text
```

Purpose:

Allow human operators to **review AI suggestions before sending replies to customers**.

---

# Human-in-the-Loop Workflow

Operator reviews Telegram message.

Possible actions:

```
send
```

→ Sends AI suggestion to the customer.

```
/reply custom text
```

→ Sends edited reply instead.

---

# Airtable Fields Used

| Field | Purpose |
|------|------|
| airtable_record_id | Link to message record |
| message_text | Customer message |
| channel | Source platform |
| broad_category | Message classification |
| issue_category | Support subtype |
| priority | Urgency |
| ai_suggested_reply | AI generated response |
| ai_reply_used | Tracks whether AI suggestion was used |

---

# Scenario Status

Current implementation:

```
✓ Webhook trigger
✓ Support filter
✓ OpenAI response generation
✓ Airtable record update
✓ Telegram notification
```

Next step:

```
08-telegram-support-commands
```

This scenario will allow operators to send replies using Telegram commands.
