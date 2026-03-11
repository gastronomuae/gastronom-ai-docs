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

message.reply_to_message exists


This prevents the scenario from triggering on unrelated messages in the Telegram group.

---

# Step 2 — Extract Airtable Record ID

The original Telegram notification contains the Airtable record ID:

Example:

```
📩 New customer message

📢 instagram
👤 Customer message text

🤖 AI Suggested reply:
🆔 rectCfWw73GesVNYP

```

We extract this ID from the original message.

**Module**
Tools → Set Variable

Variable name:


record_id


Value:

```
trim(get(split(1.message.reply_to_message.text; "🆔 "); 2))
```

Result:

```
rectCfWw73GesVNYP
```

---

# Step 3 — Retrieve Conversation Data

**Module**
Airtable → Get Record

Record ID:


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

## Route 2 — Custom Reply (/reply)

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
ai_reply_used | true |
escalation_flag | false |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | instagram |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `8.conversation_hash` |
label_source | ai_prediction |

---

# Airtable Record — Instagram - Custom Reply (/reply)

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
ai_reply_used | false |
escalation_flag | true |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | instagram |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `8.conversation_hash` |
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
ai_reply_used | true |
escalation_flag | false |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | whatsapp_A |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `8.conversation_hash` |
label_source | ai_prediction |

---

# Airtable Record — WhatsApp - Custom Reply (/reply)

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
ai_reply_used | false |
escalation_flag | true |
customer_sentiment | `8.customer_sentiment` |
priority | `8.Priority` |
confidence_score | `8.confidence_score` |
channel | whatsapp_A |
conversation_started_at | `8.conversation_started_at` |
conversation_hash | `8.conversation_hash` |
label_source | human_labeled |
