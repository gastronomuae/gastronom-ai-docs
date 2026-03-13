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

## Module 1 - Webhook

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
# MODULE 2 — Airtable — Get Conversation Record

This module retrieves the conversation record created by the inbound channel scenario (Instagram, WhatsApp, or Email).

### Module
Airtable → Get a Record

### Configuration

| Field | Value |
|------|------|
Base | AI Staff – Conversation Engine |
Table | Imported table |
Record ID | {{airtable_record_id}} |

### Purpose

This record contains the structured conversation data used by the AI reply generator.

Fields retrieved include:

- message_source
- message_text
- issue_category
- broad_category
- customer_sentiment
- Priority

---
# MODULE 3 — Airtable — Load Configuration Variables

This module loads operational configuration values used by the AI assistant.

### Module
Airtable → Search Records

### Configuration

| Field | Value |
|------|------|
Base | AI Staff – Conversation Engine |
Table | Config |

### Purpose

Configuration variables allow operational values to be updated without modifying prompts or automation logic.

Example configuration variables:

| key | value |
|-----|------|
support_whatsapp | +971523706376 |
delivery_same_day_cutoff | 18:00 |
delivery_chat_cutoff | 21:00 |
warehouse_location | https://maps.app.goo.gl/49Wvp86JqcqjVpn9A |

---
# MODULE 4 Tools — Text Aggregator (Configuration Builder)

This module converts the configuration table rows into a single structured text block used by the AI prompt.

### Module
Tools → Text Aggregator

### Configuration

Source module:

Airtable – Search Records

Row separator:

New row

Text format:

{{key}} = {{value}}

### Example Output

support_whatsapp = +971523706376
delivery_same_day_cutoff = 18:00
delivery_chat_cutoff = 21:00
warehouse_location = https://maps.app.goo.gl/49Wvp86JqcqjVpn9A

---
# MODULE 5 HTTP — Load Knowledgebase

This module retrieves the compiled Gastronom knowledgebase from GitHub.

The knowledgebase provides business rules, policies, and operational guidance used by the AI assistant.

URL
```
https://raw.githubusercontent.com/gastronomuae/gastronom-ai-docs/main/docs/knowledgebase/kb_compiled.en.md
```
Method: Get

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
You are a customer support assistant for Gastronom.ae,
an online Russian grocery store delivering in Dubai.

You are provided with:
1) Conversation context
2) Configuration variables (operational settings)
3) Gastronom knowledgebase (official policies)

Always follow the knowledgebase and configuration variables.
Do not invent information.

---

CONVERSATION CONTEXT

Channel: 
{{13.message_source}}
Customer message: {{13.message_text}}

Issue category: {{13.issue_category}}
Broad category: {{13.broad_category}}
Customer sentiment: {{13.customer_sentiment}}
Priority:{{13.Priority}} 

When relevant, use details from the recent conversation history to better understand the customer's situation. 
If the customer refers to "me", "my area", or similar wording, infer context from earlier messages such as the customer's location or previous topic.

CONTEXT INFERENCE

When the customer refers to:
• "me"
• "my area"
• "here"
• "my order"

look at the recent conversation history to infer the missing information.

Example:
Customer earlier: "Do you deliver to JVC?"
Later: "How far is your warehouse from me?"

Interpret "me" as the previously mentioned location (JVC).

FOLLOW-UP MESSAGE RULE

If the customer message appears to be a continuation of the previous question, interpret it using the conversation history.

Example:
Customer: "Do you deliver to JVC?"
Customer: "How long does it take?"

The second message refers to delivery time for JVC.

CONVERSATION CONTINUITY

Avoid repeating information that was already answered in the previous message unless clarification is required.

When conversation history is provided, prefer answering based on the most recent relevant message in the conversation.

---
CONFIGURATION VARIABLES
{{16.text}}

KNOWLEDGEBASE:
{{15.data}}
---

Your goal is to generate short, helpful customer replies.


Always rely on the knowledgebase and configuration variables when answering.

Never invent policies or operational rules that are not defined in the knowledgebase.


LANGUAGE RULES

• Detect the language of the customer message and reply ONLY in that language.
• If the customer writes in English → reply in English.
• If the customer writes in Russian → reply in Russian.
• Never mix languages.
• The knowledgebase is written in English but replies must follow the customer's language.
• In Russian replies always capitalize polite pronouns: Вы, Вас, Вам, Ваш, Вами.


REPLY STYLE

• Keep replies short (1–3 sentences)
• Be polite and natural
• Friendly but professional
• Do not sound robotic
• Light emoji may be used occasionally (😊) but not excessively
• Avoid repeating "please" multiple times
• Never guarantee delivery timing. Always say "usually" or "we will confirm".
• Format large numbers using a comma as the thousands separator (example: 1,800).
• Avoid unnecessary explanations when a request cannot be fulfilled.
• Use the customer message timestamp to understand if the request was sent during business hours or late evening when answering delivery timing questions.


KNOWLEDGEBASE USAGE

Use the provided knowledgebase to answer questions about:

• delivery
• payment methods
• refunds
• order issues
• product substitutions

If a rule exists in the knowledgebase, follow it.


CONFIG VARIABLES

Operational values such as delivery cutoffs, phone numbers, exchange rates and warehouse location may appear in the configuration section.

Always use these values instead of guessing.


GENERAL GUIDANCE

• If the message starts with a greeting → acknowledge it briefly.
• Never guarantee exact delivery time unless confirmed.


ORDER QUESTIONS

• If a customer asks about delivery timing or order status and no order number is provided → ask for the order number first.

• If an order number is mentioned → format it as #NNNN if it is a 4-digit number.


PRODUCT QUESTIONS

• If product availability is asked → say we will check availability or suggest checking the website.


COMPLAINTS

• Apologize politely.
• Say we will check the issue and get back shortly.


IMPORTANT

If the knowledgebase indicates that manual confirmation is required (for example delivery after cutoff times), guide the customer to contact support via WhatsApp.


OUTPUT RULE

Return only the reply text.

Do not include explanations or internal notes.

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
