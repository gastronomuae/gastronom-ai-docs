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
Customer message (Instagram DM)
↓  
Scenario 07 processes message  
↓  
Telegram notification sent to support group  
↓  
Agent replies in Telegram  
↓  
Scenario 08 captures the reply  
↓  
Customer receives response in Instagram DM
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

After determining the reply text, the scenario routes the response based on the customer communication channel.

Router condition:

# Step 5 — Channel Router

After determining the reply text, the scenario routes the response based on the customer communication channel.

Router condition:


---
# Instagram Channel

Customer replies are delivered via the Instagram Graph API.

Module:

HTTP → Make a Request

Endpoint: 

POST
```
 https://graph.facebook.com/v25.0/me/messages
```

Request Body:

```json
{
  "recipient": {
    "id": "{{conversation_id}}"
  },
  "message": {
    "text": "{{reply_text}}"
  }
}

Field mapping:

- conversation_id	-> Airtable record
- reply_text	-> AI suggestion OR Telegram reply

The Instagram API returns:

recipient_id

message_id

The message_id is saved in Airtable as: message_id_external


---

# Updated GitHub Section — WhatsApp Placeholder


# WhatsApp Channel (Placeholder)

Future support for WhatsApp Cloud API will be implemented using the same reply router.

Router condition: channel = whatsapp


Planned module:

HTTP → Make a Request

Endpoint:

POST
```
https://graph.facebook.com/v19.0/{{phone_number_id}}/messages
```

Example payload:

```json
{
  "messaging_product": "whatsapp",
  "to": "{{wa_number}}",
  "type": "text",
  "text": {
    "body": "{{reply_text}}"
  }
}

Dataset logging will remain identical to Instagram:

- message_direction -> outbound
- message_source	-> whatsapp
- ai_reply_used	-> true / false
- label_source ->	ai_prediction / human_labeled


---

# Updated GitHub Section — Dataset Logging

Add this new section (your doc currently lacks it).

```markdown
# Dataset Logging

Every outbound reply is recorded in Airtable to build the AI training dataset.

Fields written:

| Field | Description |
|------|-------------|
conversation_id | Instagram or WhatsApp user ID |
message_direction | inbound / outbound |
message_text | actual message sent |
channel | instagram / whatsapp |
ai_reply_used | true if AI suggestion used |
label_source | ai_prediction / human_labeled |
confidence_score | AI classification confidence |
priority | message priority |
broad_category | support / order / etc |
timestamp_utc | message timestamp |
conversation_hash | unique conversation key |

This structure allows the system to continuously improve the AI support model.

---

# Instagram Reply

**Module**
HTTP → Make a Request

Endpoint:

```
POST https://graph.facebook.com/v25.0/me/messages
```

Body:

```json
{
"recipient": {
"id": "{{conversation_id}}"
},
"message": {
"text": "{{reply_text}}"
}
}
```

Where:

| Field | Source |
|------|------|
conversation_id | Airtable |
reply_text | Telegram or AI |

---


