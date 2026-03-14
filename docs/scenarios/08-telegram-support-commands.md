# Scenario 08 — Telegram Support Commands

## Purpose

This scenario allows support agents to reply to customer Instagram messages directly from a Telegram group.

It receives Telegram replies to support notifications and sends the response back to the customer via Instagram DM.

Agents can either:
- send the AI suggested reply (`send`)
- send a custom reply by typing a message.

---

# High Level Flow
```
Telegram Bot (Watch Updates)
↓
Tools (Set variable → record_id)
↓
Airtable (Get record)
↓
Router A — command router
     ├ send
     └ reply
↓
Router B — channel router
     ├ instagram
     └ whatsapp (placeholder)
↓
HTTP → send message
↓
Airtable → log outbound message

```
---

# Trigger

**Module**
Telegram Bot → Watch Updates

This module listens to all messages posted in the support Telegram group.

---

# Step 1 — Filter Only Replies

Only process messages that are replies to bot notifications.

Condition:
```
     {{1.message.reply_to_message.message_id}} exists
AND
     {{1.message.reply_to_message.from.username}} exists
```
This prevents the scenario from triggering on unrelated messages in the Telegram group.

---

# Step 2 Set Variable

Purpose: fetch original customer message sent to telegram group, where telegram bot listening. here we have customer support group and gastronom logistic groups. both has airtyable record Id as a case ID, starting with 🆔, in a new line
The original Telegram notification contains the Airtable record ID:

Example:

```
📩 New customer message

📢 instagram
👤 Customer message text

🤖 AI Suggested reply:
🆔 rectCfWw73GesVNYP

```

record_id
```
{{trim(get(split(last(split(1.message.reply_to_message.text; "🆔 ")); newline); 1))}}
```

## Order Existing Support Route

### Filter

{{1.message.reply_to_message.text}} != 🆔

### - Retrieve Conversation Data

**Module** Airtable → Get Record 

record_id

This retrieves:

- conversation_id
- channel
- ai_suggested_reply
- issue_category
- other metadata

---

# Step 4 — Reply Command Router

Support agents control the response behavior using Telegram commands.

Two reply paths exist.

---

## Route 1 — Send AI Suggested Reply

Condition:

```
lower(trim(1.message.text)) = "send"
```

Behavior:

The AI generated reply from the Airtable record is sent to the customer without modification.

Message source: ai_suggested_reply


Dataset flags written to Airtable:

| Field | Value |
|------|------|
message_direction | outbound |
ai_reply_used | true |
label_source | ai_prediction |
escalation_flag | false |

---

## Route 2 — Custom Reply

Condition:

```
lower(trim(1.message.text)) != "send"
```


Behavior:

The Telegram message written by the support agent is sent directly to the customer.

Message source: 1.message.text


Dataset flags written to Airtable:

| Field | Value |
|------|------|
message_direction | outbound |
ai_reply_used | false |
label_source | human_labeled |
escalation_flag | false |


---

# Step 5 — Channel Router

# Instagram Channel

Customer replies are delivered via the Instagram Graph API.

Two reply paths exist depending on the command used in Telegram.

| Command | Behaviour |
|--------|-----------|
send | Send AI suggested reply |
custom reply | Send agent-written message |

Both paths send a message using the same Instagram endpoint.

---

## Instagram API Endpoint

HTTP → Make a Request: POST

```
https://graph.facebook.com/v25.0/me/messages
```
---

# Send AI Suggested Reply

Used when the support agent types: send

The AI-generated response stored in Airtable is sent without modification.

### Request Body

```json
{
  "recipient": {
    "id": "{{8.conversation_id}}"
  },
  "message": {
    "text": "{{8.ai_suggested_reply}}"
  }
}
```

Field mapping:

- conversation_id	-> Airtable record
- ai_suggested_reply -> AI response generated in Scenario 07

Instagram returns:

- recipient_id
- message_id

The message_id is saved in Airtable as: message_id_external

---
# Reply With Custom Message

Used when the support agent writes a reply directly in Telegram.

Request Body
```json
{
  "recipient": {
    "id": "{{8.conversation_id}}"
  },
  "message": {
    "text": "{{1.message.text}}"
  }
}
```

Field mapping:

- conversation_id	-> Airtable record
- ai_suggested_reply -> Telegram message written by agent

The message_id is saved in Airtable as: message_id_external


---

# Airtable Logging

After sending the message to Instagram, the scenario logs the outbound message in Airtable.

Two logging variations exist depending on whether the AI reply was used or a custom reply was written.

---

# Airtable Record — Instagram - AI Reply (send)

Used when the AI suggested reply was sent.

| Field | Value |
|------|------|
conversation_id | `8.conversation_id` |
message_id_external | `21.data.message_id` |
message_direction | outbound |
message_source | instagram dm |
message_text | `8.ai_suggested_reply` |
broad_category | `8.broad_category` |
timestamp_utc | `formatDate(now; YYYY-MM-DD HH:mm:ss)` |
agent_name | `1.Message: From: User Name` |
conversation_status | waiting_customer |
ai_reply_used | true |
escalation_flag | false |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | instagram |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `1.message.from.username` |
label_source | ai_prediction |

---

# Airtable Record — Instagram - Custom Reply

Used when the agent writes a custom reply in Telegram.

| Field | Value |
|------|------|
conversation_id | `8.conversation_id` |
message_id_external | `12.data.message_id` |
message_direction | outbound |
message_source | instagram dm |
message_text | `1.Message: Text` |
broad_category | `8.broad_category` |
timestamp_utc | `formatDate(now; YYYY-MM-DD HH:mm:ss)` |
agent_name | `1.Message: From: User Name` |
conversation_status | waiting_customer |
ai_reply_used | false |
escalation_flag | true |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | instagram |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `1.message.from.username` |
label_source | human_labeled |

---

# WhatsApp Channel

Scenario 08 sends messages to customers via the WhatsApp Business Cloud API.

Two routes are supported:

- Send —> sends the AI-generated suggested reply
- Reply —> sends a manual operator message

Both routes use the same HTTP module configuration with different message text sources.

Route 1 — Send (AI Suggested Reply)

This route sends the AI-generated support reply stored in Airtable.

Used when the operator presses Send in Telegram.

HTTP — Send WhatsApp Message

Module

HTTP → Make a request
URL
```
https://graph.facebook.com/v19.0/{{WHATSAPP_PHONE_NUMBER_ID}}/messages
```
Method: POST
Headers
```
Authorization: Bearer {{WHATSAPP_ACCESS_TOKEN}}
Content-Type: application/json
```
Body type: Raw → JSON

JSON Body
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "{{8.customer_phone}}",
  "type": "text",
  "text": {
    "preview_url": false,
    "body": "{{8.ai_suggested_reply}}"
  }
}
```
Airtable Update (Send)

After sending the message, update the Airtable record.

Route 2 — Reply (Manual Operator Message)

This route sends a manual reply written by the operator in Telegram.

Used when the operator responds directly to the Telegram message.

HTTP — Send WhatsApp Message

Module

HTTP → Make a request
URL
```
https://graph.facebook.com/v19.0/{{WHATSAPP_PHONE_NUMBER_ID}}/messages
```
Method: POST

Headers
```
Authorization: Bearer {{WHATSAPP_ACCESS_TOKEN}}
Content-Type: application/json
```
Body type: Raw → JSON
JSON Body
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "{{8.customer_phone}}",
  "type": "text",
  "text": {
    "preview_url": false,
    "body": "{{1.message.text}}"
  }
}
```
---

# Airtable Logging

After sending the message to WhatsApp, the scenario logs the outbound message in Airtable.

Two logging variations exist depending on whether the AI reply was used or a custom reply was written.

---

# Airtable Record — WhatsApp - AI Reply (send)

Used when the AI suggested reply was sent.

| Field | Value |
|------|------|
conversation_id | `8.conversation_id` |
message_id_external | `21.data.message_id` |
message_direction | outbound |
message_source | whatsapp |
message_text | `8.ai_suggested_reply` |
broad_category | `8.broad_category` |
timestamp_utc | `formatDate(now; YYYY-MM-DD HH:mm:ss)` |
agent_name | `1.Message: From: User Name` |
conversation_status | waiting_customer |
ai_reply_used | true |
escalation_flag | false |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | whatsapp_A |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `1.message.from.username` |
label_source | ai_prediction |

---

# Airtable Record — WhatsApp - Custom Reply

Used when the agent writes a custom reply in Telegram.

| Field | Value |
|------|------|
conversation_id | `8.conversation_id` |
message_id_external | `12.data.message_id` |
message_direction | outbound |
message_source | whatsapp |
message_text | `1.Message: Text` |
broad_category | `8.broad_category` |
timestamp_utc | `formatDate(now; YYYY-MM-DD HH:mm:ss)` |
agent_name | `1.Message: From: User Name` |
conversation_status | waiting_customer |
ai_reply_used | false |
escalation_flag | true |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | whatsapp_A |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `1.message.from.username` |
label_source | human_labeled |

## Mapping `conversation_hash` in Scenario 8 (Agent Replies)

In **Scenario 8**, a new Airtable row is created when a human agent sends a reply via Telegram.  
Unlike inbound messages (which originate from the customer channel such as WhatsApp or Instagram), outbound messages originate from an **agent using Telegram**.

For this reason, the `conversation_hash` field must represent the **sender of the message**, not the conversation thread.

The conversation thread itself is already preserved through the field:

`conversation_id`

This value is inherited from the original inbound message using **Airtable Module 8 (Get Record)**.

---

### Field Responsibilities

| Field | Purpose | Source |
|------|------|------|
| `conversation_id` | Conversation thread identifier | Airtable 8 |
| `conversation_hash` | Sender handle (customer or agent) | Telegram / inbound source |
| `message_direction` | Indicates inbound vs outbound | Scenario logic |

---

### Why `conversation_hash` comes from Telegram

Inbound messages store the **customer handle** in `conversation_hash`.

Examples:
unclevanya.uae
+9715XXXXXXX
instagram_user123

When an agent replies via Telegram, the sender becomes the **Telegram agent**, so we map:

conversation_hash → 1.Message.From.UserName

Example: vohanjanyan

---

### Resulting Conversation Structure

| conversation_id | conversation_hash | message_direction | message_text |
|---|---|---|---|
| 7251418034970035 | unclevanya.uae | inbound | Can I get my order now? |
| 7251418034970035 | vohanjanyan | outbound | Yes, it will arrive shortly |
| 7251418034970035 | unclevanya.uae | inbound | How long exactly? |

---
---
## Order Dispatcher Route

A new router path was introduced to allow human operators in the Telegram support group to reply directly to customer messages from Instagram or WhatsApp.

This route detects when an operator replies to a Telegram message generated by the system and routes the response to the correct messaging channel.

### Filter: Order Dispatcher

Condition:
{{1.message.reply_to_message.text}} contains 🆔


This indicates that the operator is replying to a message generated by the AI system that contains the Airtable record identifier.

If the condition is true, the scenario continues to retrieve the conversation record from Airtable and dispatch the reply to the correct channel.

---

### Workflow
```
Telegram operator reply
↓
Router → Order Dispatcher
↓
Extract Airtable record ID
↓
Airtable → Get Record
↓
Channel Router
↓
Send reply to Instagram / WhatsApp
↓
Log outbound message to Airtable
```

---

# Step 3 — Retrieve Conversation Data

**Module**
Airtable → Get Record

Searching via Record ID:

This retrieves:

- conversation_id
- channel
- ai_suggested_reply
- issue_category
- other metadata

---

## Channel Router

After retrieving the Airtable record, the scenario determines which messaging platform should receive the reply.

Routing is based on the Airtable field:

channel

Possible values currently supported:

- `whatsapp`
- `instagram`

### Router Structure
```
Router
├── WhatsApp Reply
└── Instagram Reply
```

Each route sends the message through the corresponding API module.

These modules reuse the same configuration as the existing outbound messaging modules already documented earlier in this scenario.

---

### WhatsApp Reply Route

Modules reused from the existing WhatsApp outbound message implementation:

1. HTTP → Send WhatsApp message
2. Airtable → Create outbound message record

Mappings and formulas are identical to those already documented in the WhatsApp messaging section and are therefore not repeated here.

### Instagram Reply Route

Modules reused from the existing Instagram messaging implementation:

1. HTTP → Send Instagram message via Meta Graph API
2. Airtable → Create outbound message record

Mappings and formulas remain identical to the original Instagram outbound configuration.

## Conversation Status Update

When a human operator replies through the Telegram dispatcher, the conversation status is updated to: waiting_customer

This reflects that the support team has responded and the system is now waiting for the customer's next message.

### Status Flow

Customer message received
→ status = open

Operator reply
→ status = waiting_customer

Customer replies again
→ status = open


This temporary model is used while the dataset is still being trained and before automated conversation resolution logic is introduced.

## Telegram Support Group

A dedicated Telegram group is used as the dispatcher interface for human operators.

Operators can reply directly to AI-generated messages in the group, and the system will automatically forward those replies to the customer on the correct platform.

### Telegram Chat ID

-570449120

This ID is used in the Telegram bot modules for both message delivery and reply detection.

Only replies to AI-generated messages containing the Airtable record identifier (`🆔`) will trigger the dispatcher route.

### Important

Dispatcher replies only trigger when the operator replies to a Telegram message that contains the Airtable record identifier:

---
### Summary

- `conversation_id` groups all messages belonging to the same conversation.
- `conversation_hash` identifies the **sender of each message**.
- For agent replies, `conversation_hash` must be mapped to:


1.Message.From.UserName


This ensures that outbound messages are correctly attributed to the responding agent while m
