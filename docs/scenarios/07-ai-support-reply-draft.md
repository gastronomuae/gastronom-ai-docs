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
Load Conversation Context (Airtable Search)
↓
Build Context Text (Aggregator)
↓
Load Configuration Variables
↓
Load Knowledgebase
↓
AI Reply Generated
        ↓
Airtable Updated
     ├ Telegram Notification
     └ Shopify Search Orders -> Telegram Notification
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
Formula | `{conversation_id} = "{{conversation_id}}"` |
Sort | timestamp_utc descending |
Limit | recent messages |

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

Airtable – Search Records

Row separator:

New row

Example formatting:

{{if(17.message_direction = "inbound"; "Customer"; "Store")}}: {{17.message_text}}


Example output:


Customer: Do you deliver to JVC?
Store: Yes, delivery is available.
Customer: How long does it take?


### Purpose

The aggregated text is passed to the OpenAI prompt so the AI assistant can unders

---
# MODULE 18 — Text Parser - match Pattern - extract order number

Pattern: (?:#|order\s*)?([0-9]{4,6})
Global Match = Yes
Case Sensitive = No
Multline = No
Singleline = No
Continue the execution of the route even if the module finds no matches = Yes
Text = {{18.text}}

---
# MODULE 14 — Airtable — Load Configuration Variables

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
# MODULE 16 Tools — Text Aggregator (Configuration Builder)

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
# MODULE 15 HTTP — Load Knowledgebase

This module retrieves the compiled Gastronom knowledgebase from GitHub.

The knowledgebase provides business rules, policies, and operational guidance used by the AI assistant.

URL
```
https://raw.githubusercontent.com/gastronomuae/gastronom-ai-docs/main/docs/knowledgebase/kb_compiled.en.md
```
Method: Get

---
# MODULE 36 Set Variable — order_number

Purpose:
Extract a 4-digit order number from the customer message so we can decide whether to search Airtable by order number or by phone.

Variable name: order_number

Formula
```
{{first(map(split(18.text; ); "[0-9]{4}"))}}
```

Explanation: 18.text is the recent conversation history block which contains the latest customer message.
The formula searches for a 4-digit sequence, which matches Gastronom order numbers (e.g. 4201).

Example: Customer message - Where is my order 4201?
Result - order_number = 4201

Example without number: Customer message - When will you deliver?

Result - order_number = Customer: When will you deliver?

This is why the router filter must also check for numeric length.

---
# MODULE 33 Flow control → Router (IF / ELSE)

Two branches:
- Branch 1 → order number detected
- Branch 2 → fallback search by phone

## Branch 1  - order number detected

### Router Filter — Order Number Branch

Filter condition: 
```
{{36.order_number}} exists
AND
length({{36.order_number}}) = 4
AND
{{36.order_number}} matches pattern ^\d{4,6}$
```

This ensures that only 4-digit numbers trigger the order search.
If true → search Airtable using order number.

### Airtable 34 → Search records — Order Number Branch

Table : Delivery Orders

Formula 
```
{order_number} = "{{36.order_number}}"
```
Max records = 1

This returns the dispatcher record for the order.

Example output
```
order_number: 4201
customer_phone: 971561345294
operational_status: new_order
customer_status: order_received
order_total_original: 203
delivery_address: Sulafa Tower 902
```

---
## Branch 2 - ELSE Branch — Search Order by Phone

### Airtable 35 → Search records — Phone Number Branch

Table : Delivery Orders

Formula 
```
{customer_phone} = "{{1.customer_phone}}"
```
Max records = 1

This allows the AI to still find the order when the customer asks something like:
```
"When will you send my order?"
```
without mentioning the order number.


# MODULE 46 Flow Control → Merge

Purpose: Both branches must produce a single unified object called:

Output name: order_record

```
If 1st condition true 
        Select: 35 Airtable Search Records [bundle]
Else
        Select: 34 Airtable Search Records [bundle]
```

##Fields Available After Merge

These values are then used in the AI prompt:
```
{{46.order_record.order_number}}
{{46.order_record.customer_phone}}
{{46.order_record.order_total_original}}
{{46.order_record.operational_status}}
{{46.order_record.customer_status}}
{{46.order_record.last_update}}
```
These fields populate the ORDER CONTEXT (Dispatcher Order Table) section of the prompt.

---
# Step 2 — OpenAI Reply Generation module 8

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
2) Dispatcher order context (delivery operations table)
3) Configuration variables (operational settings)
4) Gastronom knowledgebase (official policies)

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

ORDER CONTEXT (Dispatcher Order Table)

The following fields come from the delivery operations table managed by the dispatcher.
This information reflects the real operational status of the order.

Order number: {{46.order_record.order_number}}
Customer phone: {{46.order_record.customer_phone}}
Order total: {{46.order_record.order_total_original}}

Operational status: {{46.order_record.operational_status}}
Customer status: {{46.order_record.customer_status}}
Last update: {{46.order_record.last_update}}

If the Order number field in ORDER CONTEXT is empty or missing, assume that no matching order was found in the dispatcher table.

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

3) CONFIGURATION VARIABLES
Use operational settings such as delivery cutoff times, phone numbers and working hours.

4) KNOWLEDGEBASE
Use the knowledgebase for official policies, delivery rules, refunds and store procedures.

Never contradict the dispatcher order table if operational_status was updated.


---
CONFIGURATION VARIABLES
{{16.text}}

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

• If a customer asks about delivery timing or order status and no order number is provided → ask for the order number first.

• If an order number is mentioned → format it as #NNNN if it is a 4-digit number.

• If order context is provided in the prompt, use the dispatcher order table information to answer.

• If operational_status = new_order → acknowledge the order but do not assume delivery timing.

• If operational_status is updated (for example out_for_delivery, dispatched, delayed) → use that information when replying.

• If operational_status = new_order → set needs_dispatch_check = true.



PRODUCT QUESTIONS

• If product availability is asked → say we will check availability or suggest checking the website.


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
- If the message contains an order number like #1234 or 1234, extract it into order_number.
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

🧾 #4196
👤 Kristina Chernova
📲 
@ unclevanya.uae
📢 instagram
📍 Sama Tower
💰 226.5

Customer message:
Здравствуйте.
Мой номер заказа #4196

@dispatcher please confirm status
🆔 rec0W4mpLQEJsFsWV

🤖 AI Suggested reply:
{{9.ai_suggested_reply}}


Commands:
send
/reply your text
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
