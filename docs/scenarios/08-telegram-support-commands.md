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

# Step 4 — Router (Send vs Custom Reply)

Agents can either:


send


or type a custom reply.

Router splits the logic.

---

## Route 1 — SEND AI Suggested Reply

Condition:

```
lower(trim(1.message.text)) = "send"
```

Message sent to customer:


ai_suggested_reply


(from Airtable record)

---

## Route 2 — Custom Reply

Condition:

```
lower(trim(1.message.text)) != "send"
```

Message sent to customer:


1.message.text


(the agent's Telegram message)

---

# Step 5 — Channel Router

After determining the reply text, the scenario routes based on the message channel.

Currently implemented:

- Instagram

Future:

- WhatsApp

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

# Echo Handling (Important)

Instagram sends a webhook event for messages sent by the bot.

Example:

```
"is_echo": true
```

These must be ignored in Scenario 07 to prevent infinite loops.

Required filter:

```
message.is_echo != true
```

---

# Current Status

✔ Telegram replies captured  
✔ Airtable record lookup working  
✔ Instagram replies successfully sent  
✔ Echo messages filtered correctly  

---

# Known Improvements (TODO)

### 1. Prevent duplicate replies

Add Airtable field:


reply_sent


Filter before sending message:

```
reply_sent = false
```

Then update record:

```
reply_sent = true
```

---

### 2. Log agent reply

Save support agent message in Airtable:


agent_reply


This improves AI training datasets.

---

### 3. Add WhatsApp support

Future router branch:

```
channel = whatsapp
```

Send via WhatsApp Cloud API.

---

# Example Workflow

Customer DM:


Здравствуйте, сколько занимает доставка?


Telegram notification:

```
📩 New customer message

👤 Здравствуйте, сколько занимает доставка?

🤖 AI Suggested reply:
Доставка обычно занимает 2–3 часа.

🆔 rectCfWw73GesVNYP
```

Agent actions:

Option 1:

```
send
```

→ sends AI reply

Option 2:

```
Доставка обычно занимает около 3 часов 😊
```

→ sends custom reply

---

# Dependencies

Scenario 07  
`07-ai-support-reply-draft`

Handles:

- message classification
- AI suggested replies
- Telegram notification

---

# Notes

Telegram messages must be replies to the bot notification.

Direct messages in the Telegram group will be ignored.
