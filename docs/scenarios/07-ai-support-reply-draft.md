## Purpose

Generate AI-suggested replies for customer support messages coming from Instagram and WhatsApp after classification and storage in Airtable.

The scenario receives structured message data via webhook, generates a suggested reply using OpenAI, stores the suggestion in Airtable, and posts the message plus recommendation to the Telegram support group for human approval.

---

# Architecture Overview

```
Instagram / WhatsApp
↓
Channel Ingestion + Classification Scenarios
↓
Airtable Conversation Record Created
↓
HTTP Request to Scenario 07
↓
07-ai-support-reply-draft
↓
Load Current Conversation Record (Airtable Get Record)
↓
Load Recent Conversation Context (Airtable Search)
↓
Build Conversation Context Text (Aggregator)
↓
Receive Known Order Number from Webhook Payload / Airtable Record
↓
Shopify Search Orders (optional, only when order number exists)
↓
Load Configuration Variables
↓
Load Knowledgebase
↓
AI Reply Generated
↓
Parse AI JSON Response
↓
Airtable Record Updated with AI Suggested Reply
↓
Router
├── Customer Support Telegram Notification
└── Logistics Telegram Notification
```

---

# Trigger

## Module 1 - Webhook

Module: **Custom Webhook**

Expected payload structure:

```json
{
        "airtable_record_id": "recSmH6QUKG2RC8Nq",
        "channel": "whatsapp_A",
        "data": {
            "image_type": null,
            "image_issue": null,
            "order_id": "#4646",
            "message_text": "hello, where is my order?",
            "order_number": null,
            "product_name": null,
            "image_summary": null
        },
        "broad_category": "support",
        "issue_category": "order_status",
        "sentiment": "unknown",
        "priority": "normal",
        "message_source": "whatsapp_A",
        "message_direction": "inbound",
        "timestamp_utc": "2026-03-21T08:36:31.000Z",
        "customer_phone": "971561345294",
        "conversation_hash": "971561345294"
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
# MODULE 17 — Airtable — Search Conversation History

This module retrieves recent messages from the same conversation in order to provide context for the AI reply generator.

### Module
Airtable → Search Records

### Configuration

| Field | Value |
|------|------|
Base | AI Staff – Conversation Engine |
Table | conversation_log |
Formula | {wa_number} = "{{1.conversation_hash}}" |
Sort | timestamp_utc Ascending |
Limit | 4 |

### Purpose

Customer messages often appear as short follow-ups that depend on previous conversation context.

Examples include:

- order numbers
- short confirmations
- delivery location references
- follow-up questions

Retrieving recent conversation history allows the AI model to correctly interpret these messages.

Example conversation history:
Customer: Do you deliver to JVC?
Store: Yes, delivery is available.
Customer: How long does it take?


Without context the last message could be ambiguous.  
With context the AI understands the question refers to delivery timing for JVC.


---
# MODULE 18 — Tools — Conversation History Aggregator

This module converts the retrieved conversation history into a single text block that can be passed to the AI prompt.

### Module
Tools → Text Aggregator

### Configuration

Source module:

Airtable – Search Records (17)

Row separator: New row

Example formatting:
```
{{if(17.message_direction = "inbound"; "Customer"; "Store")}}: {{17.message_text}}
```

Trying new formaula with enriched order id:
```
{{if(17.message_direction = inbound; "Customer"; "Store")}}: {{17.message_text}}{{if(length(trim(17.order_id)) > 0; " [Order #" + replace(17.order_id; "#"; emptystring) + "]"; emptystring)}}
```

Example output:

Customer: Do you deliver to JVC?
Store: Yes, delivery is available.
Customer: How long does it take?

New example output expected:
```
Customer: Когда доставка? [Order #4669]
Store: Сейчас уточним [Order #4669]
```

### Purpose

The aggregated text is passed to the OpenAI prompt so the AI assistant can unders


---
# MODULE 15 HTTP — Load Knowledgebase

This module retrieves the compiled Gastronom knowledgebase from GitHub.

The knowledgebase provides business rules, policies, and operational guidance used by the AI assistant.

URL
```
https://raw.githubusercontent.com/gastronomuae/gastronom-ai-docs/main/docs/knowledgebase/kb_compiled.en.md
```
Method: Get

---

---
# Shopify Search Orders — Order Context Enrichment

## Purpose

Scenario 07 uses Shopify Search Orders to enrich the AI prompt with basic Shopify order context.

This replaces the previous Delivery Orders / dispatcher-table lookup.

The goal is to help the AI understand whether the order exists and what its basic Shopify status is before drafting a support reply.

---

## When This Module Runs

The Shopify Search Orders module is used only when Scenario 07 receives an order number.

Order number source priority:

1. `data.order_number` from the webhook payload sent by the ingestion scenario
2. `order_id` from the Airtable conversation record
3. Text parser fallback, only if no structured order number exists

If no order number is available, Scenario 07 continues without Shopify order context.

---

## Module

**App:** Shopify  
**Module:** Search Orders  
**Query mode:** Advanced / Manual

### Query

If `data.order_number` is stored without `#`:

```text
name:#{{1.data.order_number}}

---

# Step 2 — OpenAI Reply Generation module 8

Module:

```
OpenAI → Generate Completion
```

Model: gpt-4o-mini

---

You are a customer support assistant for Gastronom.ae,
an online Russian grocery store delivering in Dubai.

You are provided with:

1) Conversation context
2) Dispatcher order context (delivery operations table)
3) Gastronom knowledgebase (official policies)

Always follow the data priority rules defined below and use the provided information sources.
Do not invent information or operational details.

---

CONVERSATION CONTEXT

RECENT CONVERSATION HISTORY
{{18.text}}

Message Source: {{ifempty(1.channel; "")}}
Customer message: {{ifempty(1.data.message_text; "")}}
Issue category: {{ifempty(1.issue_category; "")}}
Broad category: {{ifempty(1.broad_category; "")}}
Customer sentiment: {{ifempty(1.sentiment; "")}}
Priority: {{ifempty(1.priority; "")}}
Customer message date and time: {{ifempty(1.timestamp_utc; "")}}

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

ORDER CONTEXT (Shopify Order)

The following fields come from Shopify.
This information confirms the order exists and shows payment / fulfillment status, but it does not provide courier ETA or real delivery route status.

Known order number from webhook: {{ifempty(1.data.order_number; )}}

Shopify order number: {{ifempty(52.name; )}}
Shopify financial status: {{ifempty(52.displayFinancialStatus; )}}
Shopify fulfillment status: {{ifempty(52.displayFulfillmentStatus; )}}
Shopify confirmed: {{ifempty(52.confirmed; )}}
Shopify order created at: {{ifempty(52.createdAt; )}}
Shopify order updated at: {{ifempty(52.updatedAt; )}}
Shopify total: {{ifempty(52.totalPriceSet.amount; )}} {{ifempty(52.totalPriceSet.currencyCode; )}}

If Shopify order number is empty but known order number from webhook exists, treat the webhook order number as the likely related order, but do not invent Shopify status.

Important:
Use Shopify fulfillment status to understand the basic order stage, but remember that Shopify fulfillment status is not the same as live courier / driver ETA.

If financial status is VOIDED, REFUNDED, or the order appears cancelled, do not say it will be delivered. Say we will check the order internally.

If fulfillment status is UNFULFILLED:
- Treat the order as received but not yet marked as fulfilled in Shopify.
- Do not say the order is already with the courier or out for delivery.
- If the customer asks when it will arrive, say we can see the order and will check delivery timing with the team.

If fulfillment status is FULFILLED:
- Treat the order as completed/fulfilled in Shopify.
- If the customer asks whether it was delivered, say it appears fulfilled in the system.
- If the customer says they have not received it, apologize and say we will check with the team immediately.

If fulfillment status is PARTIALLY_FULFILLED:
- Say part of the order appears fulfilled and we will check the remaining items/status with the team.

Never give an exact delivery time unless it is explicitly provided in the prompt.

---
DATA PRIORITY RULES

Always prioritize information sources in the following order:

1) DELIVERY ORDER TABLE (Dispatcher Airtable)
This is the operational source of truth managed by the dispatcher.

If operational_status is NOT "new_order", treat this information as authoritative and high priority.

If operational_status = "new_order", treat the order as only received but not yet processed. 
In this case do not assume delivery timing or status unless confirmed.

2) RECENT CONVERSATION HISTORY
Use conversation history to understand context, follow-up questions, or references like "my order".

3) KNOWLEDGEBASE
Use the knowledgebase for official policies, delivery rules, refunds and store procedures.

Never contradict the dispatcher order table if operational_status was updated.


---

KNOWLEDGEBASE:
{{15.data}}
---

Your goal is to generate short, helpful customer replies.


Always follow the data priority rules when answering.

Never invent policies or operational rules that are not defined in the knowledgebase.


LANGUAGE RULES

• Determine the reply language primarily from the customer's latest message.
• If the customer writes in English → reply in English.
• If the customer writes in Russian → reply in Russian.
• Never mix languages.
• The knowledgebase is written in English but replies must follow the customer's language.
• In Russian replies always capitalize polite pronouns: Вы, Вас, Вам, Ваш, Вами.


Language determination logic:

• If the latest customer message is clearly written in English → reply only in English.
• If the latest customer message is clearly written in Russian → reply only in Russian.

• Use recent conversation history only when the latest message is ambiguous (for example: "ok", "thanks", order numbers, or short confirmations).

• Never choose Russian only because the store is Russian or the knowledgebase contains Russian-related rules.

• Never mix languages in a single reply.

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

• If an order number is mentioned → format it as #NNNN if it is a 4-digit number.

• If order context is provided in the prompt, use the dispatcher order table information to answer.

• If operational_status = new_order → acknowledge the order but do not assume delivery timing.

• If operational_status is updated (for example out_for_delivery, dispatched, delayed) → use that information when replying.

• If operational_status = new_order → set needs_dispatch_check = true.


PRODUCT QUESTIONS

• If product availability is asked → say we will check availability or suggest checking the website.

ORDER NUMBER CONTEXT RULES

For order-status, delivery timing, courier arrival, missing item, address change, or delivery problem questions, determine the order number in this priority:

1. ORDER CONTEXT from the dispatcher table.
2. RECENT CONVERSATION HISTORY, especially automated acknowledgement messages like:
   “Automated order acknowledgement sent for Order #NNNN”.
3. Explicit order number in the latest customer message, unless it is clearly an address-related number.

If ORDER CONTEXT is empty but RECENT CONVERSATION HISTORY contains a recent automated acknowledgement for Order #NNNN:
- treat #NNNN as the likely related order
- do not ask the customer for the order number again
- do not invent delivery timing or courier status
- reply that we will check the order status and get back shortly
- return "order_number": "NNNN"
- set "needs_dispatch_check": true

Do not treat address-related numbers as order numbers. Numbers following these words are address details, not order IDs:
apartment, apt, flat, unit, villa, room, building, floor, office,
апартамент, квартира, кв, дом, вилла, офис, этаж.

Only ask the customer for the order number if no likely order number exists in ORDER CONTEXT, RECENT CONVERSATION HISTORY, or the latest message.

COMPLAINTS

• Apologize politely.
• Say we will check the issue and get back shortly.


IMPORTANT

If the knowledgebase indicates that manual confirmation is required (for example delivery after cutoff times), guide the customer to contact support via WhatsApp.


OUTPUT RULE

Return ONLY valid JSON using this format:

{
  "reply": "message to send to the customer",
  "order_number": "order number digits only (remove # or other symbols), otherwise null",
  "customer_phone": "phone if mentioned, otherwise null",
"phone_detected": true or false,
  "customer_email": "email if mentioned, otherwise null",
  "needs_dispatch_check": true or false
}

Rules:
- reply must contain the full message to send to the customer.
- If the message or recent conversation history contains a likely order number like #1234 or an explicit order reference, extract it into order_number. Do not extract address-related numbers such as apartment, flat, villa, room, building, floor, or office numbers.
- If no order number exists return null.
- needs_dispatch_check = true if the message relates to order status, delivery timing, courier arrival, missing items, or delivery problems.
- If the customer asks about order status, delivery timing, courier arrival, tracking, missing items, or delivery problems, NEVER guess the order status.
- If the real order status is not provided in the prompt context, reply that you will check the order and get back shortly.

Do not include explanations or internal notes.
Return ONLY the JSON object. No text before or after it.

```

---

## User Input

```
Customer message:
{{ifempty(1.data.message_text; )}}
```

## Conversation Context Usage

The AI assistant receives recent conversation history before generating a reply.

This allows the assistant to correctly interpret short follow-up messages.

Examples:

| Message | Without Context | With Context |
|------|------|------|
4177 | unclear | order_status |
Yes | unclear | delivery_area |
How long? | ambiguous | delivery_time |

The assistant should always use the most recent relevant message in the conversation when interpreting the customer's request.

---

# Step 3 — Json Parse 20 

JSON String: {{8.choices[].message.content}}

---

# Step 4 — Airtable Update 9 

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

# Step 5 — Router 25

A -> Normal support flow (to send a notification to support group chat in telegram)
B -> if order number available, search for order details in shopify and then send an escalation message

---

# Step 5.A — Telegram Notification 12 (Gastronom • Logistics Group)

Chat ID: -5133624518

Example message format:

```
📩 New customer message

📢 {{9.channel}}
📲 {{9.wa_number}}
🛒 {{9.order_id}}
💬 {{9.message_text}}

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

# Step 5.B — Shopify Search Orders 29

## Filter - Dispatcher escalation

```
{{20.needs_dispatch_check}} = true
AND
{{20.order_number}} Exists
```

# Step 6 — Send Telegram Message 30 (Gastronom • Logistics Group)

Chat ID: -5133624518

Example message format:

```
📦 ORDER CHECK REQUIRED

🧾 {{29.name}}
👤 {{29.billingAddress.firstName}} {{29.billingAddress.lastName}}
📲 {{9.wa_number}}
@ {{9.conversation_hash}}
📢 {{9.channel}}
📍 {{29.billingAddress.address1}}
💰 {{29.totalPriceSet.amount}}

Customer message:
{{9.message_text}}

@e_dnrvn please confirm status
🆔 {{9.id}}
🤖 AI Suggested reply:
{{9.ai_suggested_reply}}

```

Purpose:

Escalate to dispatcher and tag him fro action

---

# Scenario Status

Current implementation:

```
✓ Webhook trigger
✓ Support filter
✓ Conversation history retrieval
✓ Context aggregation
✓ OpenAI response generation
✓ Airtable record update
✓ Telegram notification
```

Next step:

```
08-telegram-support-commands
```

This scenario will allow operators to send replies using Telegram commands.
